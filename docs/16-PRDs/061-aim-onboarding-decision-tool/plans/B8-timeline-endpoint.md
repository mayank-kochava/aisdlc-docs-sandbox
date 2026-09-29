---
id: plan-b8
title: "B8 — Timeline Endpoint + Cascade"
---

## Goal

Add `PUT /{id}/timeline` to `OnboardingSessionsController`. The endpoint accepts a single milestone edit (status, delayed-to-date, or notes), persists the change to the milestone array, recomputes all subsequent milestones' implied est-dates from the cumulative delay delta, and appends a `ChangeLogEntry` server-side (author = `CurrentUser.UserId`, which carries the session email from the `x-user-id` header).

Editable by **client and CSM** — no CSM gate on this endpoint (design-consolidated §7, §8a).

---

## Files

**Create**

- `Kochava.Aim.Portal.Models/DTOs/TimelineEditDto.cs`
- `Kochava.Aim.Portal.Services/TimelineCascadeHelper.cs`
- `Kochava.Aim.Portal.Tests/TimelineCascadeHelperTests.cs`
- `Kochava.Aim.Portal.Tests/OnboardingSessionTimelineServiceTests.cs`

**Modify**

- `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs`
- `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs`
- `Kochava.Aim.Portal.Services/OnboardingSessionService.cs`
- `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs`
- `Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs`

---

## Dependencies

B4 — `OnboardingSessionsController`, `OnboardingSessionService`, `OnboardingSessionDataLayer`, and `IOnboardingSessionDataLayer` all exist.

---

## Tasks

### Task 1 — Write failing cascade unit tests

**1.1** Create `Kochava.Aim.Portal.Tests/TimelineCascadeHelperTests.cs`:

```csharp
using System;
using System.Collections.Generic;
using FluentAssertions;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using Kochava.Aim.Portal.Services;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class TimelineCascadeHelperTests {

    // anchorDate: Monday 2026-06-01
    private static readonly DateTimeOffset Anchor = new DateTimeOffset(2026, 6, 1, 0, 0, 0, TimeSpan.Zero);

    private static List<Milestone> BuildMilestones() => new() {
        new Milestone { Key = "m1", WeekOffset = 0,  DelayedToDate = null },
        new Milestone { Key = "m2", WeekOffset = 2,  DelayedToDate = null },
        new Milestone { Key = "m3", WeekOffset = 4,  DelayedToDate = null },
        new Milestone { Key = "m4", WeekOffset = 8,  DelayedToDate = null },
    };

    // ── est-date formula ──────────────────────────────────────────────────────

    [Test]
    public void EstDate_NoDelay_ReturnsAnchorPlusWeekOffsetRoundedToMonday() {
        // m2: anchorDate + 2 weeks = 2026-06-15 (already Monday)
        var milestones = BuildMilestones();
        var result = TimelineCascadeHelper.ComputeEstDates(milestones, Anchor);

        result["m1"].Should().Be(new DateTimeOffset(2026, 6, 1,  0, 0, 0, TimeSpan.Zero));
        result["m2"].Should().Be(new DateTimeOffset(2026, 6, 15, 0, 0, 0, TimeSpan.Zero));
        result["m3"].Should().Be(new DateTimeOffset(2026, 6, 29, 0, 0, 0, TimeSpan.Zero));
        result["m4"].Should().Be(new DateTimeOffset(2026, 7, 27, 0, 0, 0, TimeSpan.Zero));
    }

    [Test]
    public void EstDate_AnchorOnNonMonday_RoundsUpToNextMonday() {
        // anchor = Wednesday 2026-06-03; weekOffset=0 → next Monday = 2026-06-08
        var anchor = new DateTimeOffset(2026, 6, 3, 0, 0, 0, TimeSpan.Zero);
        var milestones = new List<Milestone> {
            new Milestone { Key = "m1", WeekOffset = 0, DelayedToDate = null }
        };

        var result = TimelineCascadeHelper.ComputeEstDates(milestones, anchor);

        result["m1"].DayOfWeek.Should().Be(DayOfWeek.Monday);
        result["m1"].Should().Be(new DateTimeOffset(2026, 6, 8, 0, 0, 0, TimeSpan.Zero));
    }

    // ── cascade from delayedToDate ────────────────────────────────────────────

    [Test]
    public void Cascade_DelayOnM2_ShiftsM3AndM4ByDelta() {
        // m2 natural est = 2026-06-15; delayed to 2026-07-06 (+21 days = +3 weeks)
        var milestones = BuildMilestones();
        milestones[1].DelayedToDate = new DateTimeOffset(2026, 7, 6, 0, 0, 0, TimeSpan.Zero);

        var result = TimelineCascadeHelper.ComputeEstDates(milestones, Anchor);

        // m1 unchanged
        result["m1"].Should().Be(new DateTimeOffset(2026, 6, 1, 0, 0, 0, TimeSpan.Zero));
        // m2 = delayed date
        result["m2"].Should().Be(new DateTimeOffset(2026, 7, 6, 0, 0, 0, TimeSpan.Zero));
        // m3: natural = 2026-06-29; shifted by +21 days = 2026-07-20 (Monday check)
        result["m3"].Should().Be(new DateTimeOffset(2026, 7, 20, 0, 0, 0, TimeSpan.Zero));
        // m4: natural = 2026-07-27; shifted by +21 days = 2026-08-17
        result["m4"].Should().Be(new DateTimeOffset(2026, 8, 17, 0, 0, 0, TimeSpan.Zero));
    }

    [Test]
    public void Cascade_TwoDelays_CumulativeDeltaApplied() {
        // m1 delayed by +7 days (2026-06-08), m2 delayed by additional +7 days
        // m2 natural+m1delta = 2026-06-22; delayed to 2026-06-29 (+7 more)
        // total delta hitting m3+ = 21 days
        var milestones = BuildMilestones();
        milestones[0].DelayedToDate = new DateTimeOffset(2026, 6, 8, 0, 0, 0, TimeSpan.Zero); // +7 days on m1
        milestones[1].DelayedToDate = new DateTimeOffset(2026, 6, 29, 0, 0, 0, TimeSpan.Zero); // +7 more on m2

        var result = TimelineCascadeHelper.ComputeEstDates(milestones, Anchor);

        result["m1"].Should().Be(new DateTimeOffset(2026, 6, 8, 0, 0, 0, TimeSpan.Zero));
        result["m2"].Should().Be(new DateTimeOffset(2026, 6, 29, 0, 0, 0, TimeSpan.Zero));
        // m3 natural = 2026-06-29; cumulative delta = 21 days → 2026-07-20
        result["m3"].Should().Be(new DateTimeOffset(2026, 7, 20, 0, 0, 0, TimeSpan.Zero));
        result["m4"].Should().Be(new DateTimeOffset(2026, 8, 17, 0, 0, 0, TimeSpan.Zero));
    }

    [Test]
    public void Cascade_NoDelays_ReturnsNaturalDates() {
        var milestones = BuildMilestones();
        var result = TimelineCascadeHelper.ComputeEstDates(milestones, Anchor);

        foreach (var key in result.Keys) {
            result[key].DayOfWeek.Should().Be(DayOfWeek.Monday, because: $"{key} must land on Monday");
        }
    }

    [Test]
    public void Cascade_EmptyMilestones_ReturnsEmptyDictionary() {
        var result = TimelineCascadeHelper.ComputeEstDates(new List<Milestone>(), Anchor);
        result.Should().BeEmpty();
    }

    [Test]
    public void Cascade_DelayedDateAlreadyMonday_NoFurtherRounding() {
        var milestones = new List<Milestone> {
            new Milestone { Key = "m1", WeekOffset = 0,
                DelayedToDate = new DateTimeOffset(2026, 6, 8, 0, 0, 0, TimeSpan.Zero) } // Monday
        };
        var result = TimelineCascadeHelper.ComputeEstDates(milestones, Anchor);
        result["m1"].DayOfWeek.Should().Be(DayOfWeek.Monday);
        result["m1"].Should().Be(new DateTimeOffset(2026, 6, 8, 0, 0, 0, TimeSpan.Zero));
    }

    // ── NextMonday helper ─────────────────────────────────────────────────────

    [TestCase(DayOfWeek.Monday,    0)]  // already Monday — stays
    [TestCase(DayOfWeek.Tuesday,   6)]  // +6 to next Monday
    [TestCase(DayOfWeek.Wednesday, 5)]
    [TestCase(DayOfWeek.Thursday,  4)]
    [TestCase(DayOfWeek.Friday,    3)]
    [TestCase(DayOfWeek.Saturday,  2)]
    [TestCase(DayOfWeek.Sunday,    1)]
    public void NextMonday_ReturnsCorrectDayOffset(DayOfWeek dow, int expectedDaysToAdd) {
        var baseDate = new DateTimeOffset(2026, 6, 1, 0, 0, 0, TimeSpan.Zero); // known Monday
        var input = baseDate.AddDays((int)dow - (int)DayOfWeek.Monday);
        var result = TimelineCascadeHelper.NextMonday(input);
        result.DayOfWeek.Should().Be(DayOfWeek.Monday);
        result.Should().Be(input.AddDays(expectedDaysToAdd));
    }
}
```

**1.2** Run (expect compile error — `TimelineCascadeHelper` not yet created):

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/mmm-portal-api
dotnet build Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

---

### Task 2 — Write failing service-layer tests

**2.1** Create `Kochava.Aim.Portal.Tests/OnboardingSessionTimelineServiceTests.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using FluentAssertions;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using Kochava.Aim.Portal.DataAccess.Mongo.DataLayers.Interfaces;
using Kochava.Aim.Portal.Common.Identity.Interfaces;
using Kochava.Aim.Portal.Models.ActionResults;
using Kochava.Aim.Portal.Models.DTOs;
using Kochava.Aim.Portal.Services;
using Moq;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class OnboardingSessionTimelineServiceTests {

    private Mock<IOnboardingSessionDataLayer> _dataLayerMock;
    private Mock<ICurrentUserContext> _userMock;
    private OnboardingSessionService _service;

    private const string SessionId   = "664000000000000000000001";
    private const string AdvertiserId = "adv-42";
    private const string AuthorEmail  = "alice@kochava.com";

    private static readonly DateTimeOffset Anchor =
        new DateTimeOffset(2026, 6, 1, 0, 0, 0, TimeSpan.Zero); // Monday

    [SetUp]
    public void Setup() {
        _dataLayerMock = new Mock<IOnboardingSessionDataLayer>();
        _userMock      = new Mock<ICurrentUserContext>();

        _userMock.Setup(u => u.ActiveAdvertiserIdAsString).Returns(AdvertiserId);
        _userMock.Setup(u => u.UserId).Returns(AuthorEmail);

        _service = new OnboardingSessionService(
            _dataLayerMock.Object,
            () => _userMock.Object
        );
    }

    // ── happy path: status edit ───────────────────────────────────────────────

    [Test]
    public async Task EditTimeline_StatusChange_PersistsAndAppendsChangeLog() {
        var session = BuildSession();
        _dataLayerMock
            .Setup(d => d.GetByIdAsync(SessionId, AdvertiserId))
            .ReturnsAsync(session);
        _dataLayerMock
            .Setup(d => d.UpdateTimelineAsync(SessionId, AdvertiserId,
                It.IsAny<List<Milestone>>(), It.IsAny<List<ChangeLogEntry>>()))
            .ReturnsAsync(DataUpdateResult.Success);

        var dto = new TimelineEditDto {
            MilestoneKey    = "kickoff",
            StatusUpdate    = "in_progress",
            DelayedToDate   = null,
            Notes           = null
        };

        var result = await _service.EditTimelineAsync(SessionId, dto);

        result.Should().Be(DataUpdateResult.Success);

        _dataLayerMock.Verify(d => d.UpdateTimelineAsync(
            SessionId, AdvertiserId,
            It.Is<List<Milestone>>(m => m[0].Status == "in_progress"),
            It.Is<List<ChangeLogEntry>>(log =>
                log.Count == 1 &&
                log[0].UserEmail == AuthorEmail &&
                log[0].Type      == "timeline_status" &&
                log[0].Field     == "kickoff.status" &&
                log[0].OldValue  == "not_started" &&
                log[0].NewValue  == "in_progress"
            )
        ), Times.Once);
    }

    // ── happy path: delay → cascade ──────────────────────────────────────────

    [Test]
    public async Task EditTimeline_Delay_CascadesSubsequentMilestones() {
        var session = BuildSession();
        // anchor = 2026-06-01 (Monday)
        // m1: weekOffset=0, natural=2026-06-01
        // m2: weekOffset=2, natural=2026-06-15
        // Delay m1 to 2026-06-08 (+7 days)
        // m2 natural shifted by +7 → 2026-06-22

        List<Milestone> capturedMilestones = null;
        _dataLayerMock
            .Setup(d => d.GetByIdAsync(SessionId, AdvertiserId))
            .ReturnsAsync(session);
        _dataLayerMock
            .Setup(d => d.UpdateTimelineAsync(SessionId, AdvertiserId,
                It.IsAny<List<Milestone>>(), It.IsAny<List<ChangeLogEntry>>()))
            .Callback<string, string, List<Milestone>, List<ChangeLogEntry>>(
                (_, _, m, _) => capturedMilestones = m)
            .ReturnsAsync(DataUpdateResult.Success);

        var newDate = new DateTimeOffset(2026, 6, 8, 0, 0, 0, TimeSpan.Zero);
        var dto = new TimelineEditDto {
            MilestoneKey  = "kickoff",
            StatusUpdate  = null,
            DelayedToDate = newDate,
            Notes         = null
        };

        await _service.EditTimelineAsync(SessionId, dto);

        capturedMilestones.Should().NotBeNull();
        capturedMilestones[0].DelayedToDate.Should().Be(newDate);
        // m2 est-date = 2026-06-22 — verified via cascade; no DelayedToDate set on m2
        capturedMilestones[1].DelayedToDate.Should().BeNull();
    }

    // ── happy path: notes edit ────────────────────────────────────────────────

    [Test]
    public async Task EditTimeline_NotesUpdate_PersistsNoteAndLogsChange() {
        var session = BuildSession();
        _dataLayerMock
            .Setup(d => d.GetByIdAsync(SessionId, AdvertiserId))
            .ReturnsAsync(session);
        _dataLayerMock
            .Setup(d => d.UpdateTimelineAsync(SessionId, AdvertiserId,
                It.IsAny<List<Milestone>>(), It.IsAny<List<ChangeLogEntry>>()))
            .ReturnsAsync(DataUpdateResult.Success);

        var dto = new TimelineEditDto {
            MilestoneKey  = "kickoff",
            StatusUpdate  = null,
            DelayedToDate = null,
            Notes         = "Delayed due to data access issue"
        };

        var result = await _service.EditTimelineAsync(SessionId, dto);

        result.Should().Be(DataUpdateResult.Success);
        _dataLayerMock.Verify(d => d.UpdateTimelineAsync(
            SessionId, AdvertiserId,
            It.Is<List<Milestone>>(m => m[0].Notes == "Delayed due to data access issue"),
            It.Is<List<ChangeLogEntry>>(log =>
                log.Count == 1 &&
                log[0].Type  == "timeline_notes" &&
                log[0].Field == "kickoff.notes"
            )
        ), Times.Once);
    }

    // ── author is always session email ────────────────────────────────────────

    [Test]
    public async Task EditTimeline_AuthorIsAlwaysSessionEmail_NotClientSupplied() {
        var session = BuildSession();
        _dataLayerMock
            .Setup(d => d.GetByIdAsync(SessionId, AdvertiserId))
            .ReturnsAsync(session);
        _dataLayerMock
            .Setup(d => d.UpdateTimelineAsync(SessionId, AdvertiserId,
                It.IsAny<List<Milestone>>(), It.IsAny<List<ChangeLogEntry>>()))
            .ReturnsAsync(DataUpdateResult.Success);

        var dto = new TimelineEditDto {
            MilestoneKey = "kickoff",
            StatusUpdate = "complete",
        };

        await _service.EditTimelineAsync(SessionId, dto);

        _dataLayerMock.Verify(d => d.UpdateTimelineAsync(
            SessionId, AdvertiserId,
            It.IsAny<List<Milestone>>(),
            It.Is<List<ChangeLogEntry>>(log => log[0].UserEmail == AuthorEmail)
        ), Times.Once);
    }

    // ── error path: session not found ────────────────────────────────────────

    [Test]
    public async Task EditTimeline_SessionNotFound_ReturnsNotFound() {
        _dataLayerMock
            .Setup(d => d.GetByIdAsync(SessionId, AdvertiserId))
            .ReturnsAsync((OnboardingSession)null);

        var dto = new TimelineEditDto { MilestoneKey = "kickoff", StatusUpdate = "done" };
        var result = await _service.EditTimelineAsync(SessionId, dto);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.NotFound);
    }

    // ── error path: unknown milestone key ────────────────────────────────────

    [Test]
    public async Task EditTimeline_UnknownMilestoneKey_ReturnsNotFound() {
        var session = BuildSession();
        _dataLayerMock
            .Setup(d => d.GetByIdAsync(SessionId, AdvertiserId))
            .ReturnsAsync(session);

        var dto = new TimelineEditDto { MilestoneKey = "does_not_exist", StatusUpdate = "done" };
        var result = await _service.EditTimelineAsync(SessionId, dto);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.NotFound);
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private static OnboardingSession BuildSession() => new() {
        Id           = SessionId,
        AdvertiserId = AdvertiserId,
        Status       = "complete",
        Timeline = new TimelineState {
            AnchorDate = Anchor,
            Milestones = new List<Milestone> {
                new Milestone {
                    Key        = "kickoff",
                    Name       = "Kickoff",
                    Responsible = "CSM",
                    WeekOffset = 0,
                    Duration   = "1w",
                    Status     = "not_started"
                },
                new Milestone {
                    Key        = "data_onboard",
                    Name       = "Data Onboarding",
                    Responsible = "Client",
                    WeekOffset = 2,
                    Duration   = "2w",
                    Status     = "not_started"
                }
            }
        },
        ChangeLog = new List<ChangeLogEntry>()
    };
}
```

**2.2** Run (expect compile errors — DTO and service method not yet created):

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/mmm-portal-api
dotnet build Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

---

### Task 3 — Create `TimelineEditDto`

**3.1** Create `Kochava.Aim.Portal.Models/DTOs/TimelineEditDto.cs`:

```csharp
using System;

namespace Kochava.Aim.Portal.Models.DTOs;

/// <summary>
/// Payload for PUT /{id}/timeline.
/// Exactly one of StatusUpdate, DelayedToDate, or Notes must be non-null per call.
/// </summary>
public class TimelineEditDto {

    /// <summary>The milestone key to edit (required).</summary>
    public string MilestoneKey { get; set; }

    /// <summary>New status string, e.g. "in_progress" | "complete" | "delayed". Null = no change.</summary>
    public string StatusUpdate { get; set; }

    /// <summary>New date the milestone is rescheduled to. Triggers cascade recompute on subsequent milestones. Null = no change.</summary>
    public DateTimeOffset? DelayedToDate { get; set; }

    /// <summary>Free-text notes for the milestone. Null = no change.</summary>
    public string Notes { get; set; }

}
```

---

### Task 4 — Create `TimelineCascadeHelper`

**4.1** Create `Kochava.Aim.Portal.Services/TimelineCascadeHelper.cs`:

```csharp
using System;
using System.Collections.Generic;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;

namespace Kochava.Aim.Portal.Services;

/// <summary>
/// Computes est-dates for a milestone list given an anchor date and any DelayedToDate overrides.
///
/// Formula (design-consolidated §8a):
///   estDate = NextMonday(anchorDate + weekOffset × 7 days + cumulativeDeltaDays)
///
/// If a milestone has DelayedToDate set, the delta for that milestone (vs its natural date) accumulates
/// and carries forward to all subsequent milestones.
/// </summary>
public static class TimelineCascadeHelper {

    /// <summary>
    /// Returns a key→estDate dictionary for every milestone.
    /// Milestones are processed in list order; DelayedToDate overrides accumulate a delta that shifts
    /// all subsequent natural dates.
    /// </summary>
    public static Dictionary<string, DateTimeOffset> ComputeEstDates(
        List<Milestone> milestones,
        DateTimeOffset anchorDate
    ) {
        var result       = new Dictionary<string, DateTimeOffset>(milestones.Count);
        var cumulativeDeltaDays = 0.0;

        foreach (var m in milestones) {
            var naturalDate = NextMonday(anchorDate.AddDays(m.WeekOffset * 7.0 + cumulativeDeltaDays));

            if (m.DelayedToDate.HasValue) {
                var delayedMonday = NextMonday(m.DelayedToDate.Value);
                var delta         = (delayedMonday - naturalDate).TotalDays;
                cumulativeDeltaDays += delta;
                result[m.Key]        = delayedMonday;
            } else {
                result[m.Key] = naturalDate;
            }
        }

        return result;
    }

    /// <summary>
    /// Returns the date itself if it is already a Monday; otherwise advances to the next Monday.
    /// Time-of-day is zeroed; UTC offset is preserved.
    /// </summary>
    public static DateTimeOffset NextMonday(DateTimeOffset date) {
        var d = date.Date;
        int daysUntilMonday = ((int)DayOfWeek.Monday - (int)d.DayOfWeek + 7) % 7;
        return new DateTimeOffset(d.AddDays(daysUntilMonday), date.Offset);
    }
}
```

---

### Task 5 — Run cascade tests (expect pass)

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "TimelineCascadeHelper" 2>&1 | tail -30
```

All 10 cascade tests must pass.

---

### Task 6 — Add `UpdateTimelineAsync` to the data layer

**6.1** Open `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs`. Add the method signature after the existing method list:

```csharp
Task<DataUpdateResult> UpdateTimelineAsync(
    string id,
    string advertiserId,
    List<Milestone> milestones,
    List<ChangeLogEntry> appendedEntries
);
```

**6.2** Open `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs`. Add the implementation:

```csharp
public async Task<DataUpdateResult> UpdateTimelineAsync(
    string id,
    string advertiserId,
    List<Milestone> milestones,
    List<ChangeLogEntry> appendedEntries
) {
    var filter = Builders<OnboardingSession>.Filter.Eq(x => x.Id, id)
               & Builders<OnboardingSession>.Filter.Eq(x => x.AdvertiserId, advertiserId);

    var update = Builders<OnboardingSession>.Update
        .Set(x => x.Timeline.Milestones, milestones)
        .Set(x => x.UpdatedAt, DateTimeOffset.UtcNow)
        .PushEach(x => x.ChangeLog, appendedEntries);

    var result = await Collection().UpdateOneAsync(filter, update);
    return result.MatchedCount >= 1 ? DataUpdateResult.Success : DataUpdateResult.NotFound;
}
```

---

### Task 7 — Add `EditTimelineAsync` to the service

**7.1** Open `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs`. Add:

```csharp
Task<DataUpdateResult> EditTimelineAsync(string id, TimelineEditDto dto);
```

**7.2** Open `Kochava.Aim.Portal.Services/OnboardingSessionService.cs`. Add the method:

```csharp
public async Task<DataUpdateResult> EditTimelineAsync(string id, TimelineEditDto dto) {
    var advertiserId = _currentUser().ActiveAdvertiserIdAsString;
    var authorEmail  = _currentUser().UserId;

    var session = await _dataLayer.GetByIdAsync(id, advertiserId);
    if (session == null) {
        return DataUpdateResult.NotFound;
    }

    var milestone = session.Timeline.Milestones.Find(m => m.Key == dto.MilestoneKey);
    if (milestone == null) {
        return DataUpdateResult.NotFound;
    }

    var entries = new List<ChangeLogEntry>();

    if (dto.StatusUpdate != null) {
        entries.Add(new ChangeLogEntry {
            At         = DateTimeOffset.UtcNow,
            UserEmail  = authorEmail,
            Type       = "timeline_status",
            Field      = $"{dto.MilestoneKey}.status",
            OldValue   = milestone.Status,
            NewValue   = dto.StatusUpdate
        });
        milestone.Status = dto.StatusUpdate;
    }

    if (dto.DelayedToDate.HasValue) {
        entries.Add(new ChangeLogEntry {
            At         = DateTimeOffset.UtcNow,
            UserEmail  = authorEmail,
            Type       = "timeline_delay",
            Field      = $"{dto.MilestoneKey}.delayedToDate",
            OldValue   = milestone.DelayedToDate?.ToString("O"),
            NewValue   = dto.DelayedToDate.Value.ToString("O")
        });
        milestone.DelayedToDate = dto.DelayedToDate;
    }

    if (dto.Notes != null) {
        entries.Add(new ChangeLogEntry {
            At         = DateTimeOffset.UtcNow,
            UserEmail  = authorEmail,
            Type       = "timeline_notes",
            Field      = $"{dto.MilestoneKey}.notes",
            OldValue   = milestone.Notes,
            NewValue   = dto.Notes
        });
        milestone.Notes = dto.Notes;
    }

    var anchorDate = session.Timeline.AnchorDate ?? DateTimeOffset.UtcNow;
    // Recompute cascade (results are for reference/response only;
    // DelayedToDate already stored on the milestone drives future est-date computations)
    _ = TimelineCascadeHelper.ComputeEstDates(session.Timeline.Milestones, anchorDate);

    return await _dataLayer.UpdateTimelineAsync(
        id, advertiserId, session.Timeline.Milestones, entries);
}
```

---

### Task 8 — Run service-layer tests (expect pass)

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionTimeline" 2>&1 | tail -30
```

All 6 service tests must pass.

---

### Task 9 — Add the controller endpoint

**9.1** Open `Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs`. Add the action after the existing `PUT /{id}` autosave action:

```csharp
/// <summary>
/// Edits a single milestone's status, delayed-to-date, or notes and persists
/// a cascade recompute + changelog entry.  Editable by client AND CSM — no CSM gate.
/// </summary>
[HttpPut("{id}/timeline")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<ActionResult> EditTimeline(string id, [FromBody] TimelineEditDto dto) {
    if (string.IsNullOrWhiteSpace(dto?.MilestoneKey)) {
        return BadRequest(new ProblemDetails {
            Title  = "Bad Request",
            Detail = "MilestoneKey is required.",
            Status = StatusCodes.Status400BadRequest
        });
    }

    return HandleDataUpdateResult(await _onboardingSessionService.EditTimelineAsync(id, dto));
}
```

Ensure `using Kochava.Aim.Portal.Models.DTOs;` is present in the using directives if not already.

---

### Task 10 — Full test run

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -30
```

All tests must pass with no regressions.

---

### Task 11 — Commit

```bash
cd /Users/mukey/Documents/kochava-projects/k4a/mmm-portal-api

git add \
  Kochava.Aim.Portal.Models/DTOs/TimelineEditDto.cs \
  Kochava.Aim.Portal.Services/TimelineCascadeHelper.cs \
  Kochava.Aim.Portal.Services/OnboardingSessionService.cs \
  Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs \
  Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs \
  Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs \
  Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs \
  Kochava.Aim.Portal.Tests/TimelineCascadeHelperTests.cs \
  Kochava.Aim.Portal.Tests/OnboardingSessionTimelineServiceTests.cs

git commit -m "feat(onboarding): add PUT /timeline endpoint with cascade recompute (B8)"
```
