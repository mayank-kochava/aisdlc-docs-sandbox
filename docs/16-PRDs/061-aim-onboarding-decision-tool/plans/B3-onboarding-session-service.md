---
id: plan-b3
title: "B3 — DTOs + OnboardingSessionService"
---

## B3 — DTOs + OnboardingSessionService

**Date:** 2026-06-11
**Status:** Ready
**Repo:** `mmm-portal-api` (C# .NET 7, NUnit + Moq)
**Design of record:** `design-consolidated.md` §5, §6, §8a

---

## Goal

Add `OnboardingSessionDto` / `OnboardingSessionUpsertDto` to `Kochava.Aim.Portal.Models`,
and implement `IOnboardingSessionService` / `OnboardingSessionService` in
`Kochava.Aim.Portal.Services`. The service exposes three operations — get-latest,
create, and full-doc replace — and enforces the terminal-approval guard:
once `ScopeOfWork.ApprovedAt` is set the session is locked; any subsequent
create or replace returns `DataUpdateResult.AlreadyApproved`
(`DataUpdateErrorType.NameConflict`) so B4's controller can dispatch it to HTTP 409.

Mirrors `SavedViewsService` / `ISavedViewsService`; tests mirror `AccountsServiceTests`.

---

## Files

| Action | Path |
|--------|------|
| New | `Kochava.Aim.Portal.Models/DTOs/OnboardingSessionDto.cs` |
| New | `Kochava.Aim.Portal.Models/DTOs/OnboardingSessionUpsertDto.cs` |
| New | `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs` |
| New | `Kochava.Aim.Portal.Services/OnboardingSessionService.cs` |
| New | `Kochava.Aim.Portal.Tests/OnboardingSessionServiceTests.cs` |

**Note — AutoMapper profile:** the `OnboardingSessionProfile` (mapping
`OnboardingSession ↔ DTOs`) is deferred to B2 alongside the data-layer; tests in
this plan mock `IMapper` so the profile is not required here.

**Note — 409 mapping:** B4's controller maps
`DataUpdateErrorType.NameConflict → Conflict(409)` in its own switch arm (precedent:
`AppsController`, `AccountsController`). No change to `AppControllerBase` or
`DataUpdateErrorType` is needed in this plan.

---

## Dependencies

**B1** must exist first:

- `OnboardingSession` POCO with at minimum:
  `Id`, `AdvertiserId`, `Status`, `CreatedAt`, `UpdatedAt`, `CreatedByUserId`,
  `UpdatedByUserId`, and the embedded `ScopeOfWork` type exposing `ApprovedAt`
  (nullable `DateTimeOffset?`). All PascalCase; collection name
  `aim_onboarding_sessions`.

**B2** must exist first:

- `IOnboardingSessionDataLayer` with at minimum:

  ```csharp
  Task<OnboardingSession?> GetLatestByAdvertiserAsync(string advertiserId);
  Task<OnboardingSession?> GetByIdAsync(string id);
  Task<OnboardingSession> CreateAsync(OnboardingSession session);
  Task<DataUpdateResult> ReplaceAsync(string id, OnboardingSession session);
  ```

These method signatures are mirrored verbatim in the service and mocked in the
test fixtures below — any rename in B2 must be propagated here.

---

## Tasks

### T1 — Failing test: `GetLatestAsync_ReturnsNull_WhenNoSession`

Write the test file first. The service is not yet implemented; `dotnet test` must fail.

```csharp
// Kochava.Aim.Portal.Tests/OnboardingSessionServiceTests.cs
using System;
using System.Threading.Tasks;
using AutoMapper;
using Kochava.Aim.Portal.Common.Identity.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.DataLayers.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using Kochava.Aim.Portal.Models.ActionResults;
using Kochava.Aim.Portal.Models.DTOs;
using Kochava.Aim.Portal.Services;
using Moq;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class OnboardingSessionServiceTests {

    private Mock<IOnboardingSessionDataLayer> _dataLayer;
    private Mock<IMapper> _mapper;
    private Mock<ICurrentUserContext> _userContext;
    private OnboardingSessionService _service;

    [SetUp]
    public void SetUp() {
        _dataLayer   = new Mock<IOnboardingSessionDataLayer>();
        _mapper      = new Mock<IMapper>();
        _userContext = new Mock<ICurrentUserContext>();

        _userContext.Setup(x => x.ActiveAdvertiserIdAsString).Returns("adv-1");
        _userContext.Setup(x => x.UserId).Returns("user-1");

        _service = new OnboardingSessionService(
            _dataLayer.Object,
            _mapper.Object,
            () => _userContext.Object
        );
    }

    // ── GetLatestAsync ────────────────────────────────────────────────────────

    [Test]
    public async Task GetLatestAsync_ReturnsNull_WhenNoSession() {
        _dataLayer.Setup(x => x.GetLatestByAdvertiserAsync("adv-1"))
            .ReturnsAsync((OnboardingSession?)null);

        var result = await _service.GetLatestAsync();

        Assert.That(result, Is.Null);
    }
}
```

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/ --filter "OnboardingSessionServiceTests"
# Expected: compile error — OnboardingSessionService does not exist
```

---

### T2 — Failing tests: create + replace happy paths + guard cases

Extend the test file with all remaining cases before writing any implementation.

```csharp
    // ── GetLatestAsync ────────────────────────────────────────────────────────

    [Test]
    public async Task GetLatestAsync_ReturnsMappedDto_WhenSessionExists() {
        var session = new OnboardingSession { AdvertiserId = "adv-1" };
        var dto     = new OnboardingSessionDto { AdvertiserId = "adv-1" };

        _dataLayer.Setup(x => x.GetLatestByAdvertiserAsync("adv-1"))
            .ReturnsAsync(session);
        _mapper.Setup(x => x.Map<OnboardingSessionDto>(session)).Returns(dto);

        var result = await _service.GetLatestAsync();

        Assert.That(result, Is.SameAs(dto));
    }

    // ── CreateAsync ───────────────────────────────────────────────────────────

    [Test]
    public async Task CreateAsync_ReturnsDto_WhenNoExistingSession() {
        var upsert  = new OnboardingSessionUpsertDto();
        var model   = new OnboardingSession();
        var created = new OnboardingSession { AdvertiserId = "adv-1" };
        var dto     = new OnboardingSessionDto { AdvertiserId = "adv-1" };

        _dataLayer.Setup(x => x.GetLatestByAdvertiserAsync("adv-1"))
            .ReturnsAsync((OnboardingSession?)null);
        _mapper.Setup(x => x.Map<OnboardingSession>(upsert)).Returns(model);
        _dataLayer.Setup(x => x.CreateAsync(model)).ReturnsAsync(created);
        _mapper.Setup(x => x.Map<OnboardingSessionDto>(created)).Returns(dto);

        var result = await _service.CreateAsync(upsert);

        Assert.That(result.Successful, Is.True);
        Assert.That(result.Value, Is.SameAs(dto));
        _dataLayer.Verify(x => x.CreateAsync(It.Is<OnboardingSession>(s =>
            s.AdvertiserId == "adv-1" &&
            s.CreatedByUserId == "user-1" &&
            s.UpdatedByUserId == "user-1" &&
            s.CreatedAt != default &&
            s.UpdatedAt != default
        )), Times.Once);
    }

    [Test]
    public async Task CreateAsync_ReturnsAlreadyApproved_WhenSessionIsApproved() {
        var approved = new OnboardingSession {
            ScopeOfWork = new ScopeOfWork { ApprovedAt = DateTimeOffset.UtcNow }
        };

        _dataLayer.Setup(x => x.GetLatestByAdvertiserAsync("adv-1"))
            .ReturnsAsync(approved);

        var result = await _service.CreateAsync(new OnboardingSessionUpsertDto());

        Assert.That(result.Successful, Is.False);
        Assert.That(result.ErrorType, Is.EqualTo(DataUpdateErrorType.NameConflict));
        _dataLayer.Verify(x => x.CreateAsync(It.IsAny<OnboardingSession>()), Times.Never);
    }

    // ── ReplaceAsync ──────────────────────────────────────────────────────────

    [Test]
    public async Task ReplaceAsync_StampsTimestampsAndDelegates_WhenNotApproved() {
        var upsert   = new OnboardingSessionUpsertDto();
        var existing = new OnboardingSession { ScopeOfWork = new ScopeOfWork { ApprovedAt = null } };
        var model    = new OnboardingSession();

        _dataLayer.Setup(x => x.GetByIdAsync("sess-1")).ReturnsAsync(existing);
        _mapper.Setup(x => x.Map<OnboardingSession>(upsert)).Returns(model);
        _dataLayer.Setup(x => x.ReplaceAsync("sess-1", model))
            .ReturnsAsync(DataUpdateResult.Success);

        var result = await _service.ReplaceAsync("sess-1", upsert);

        Assert.That(result.Successful, Is.True);
        _dataLayer.Verify(x => x.ReplaceAsync("sess-1", It.Is<OnboardingSession>(s =>
            s.UpdatedAt != default &&
            s.UpdatedByUserId == "user-1"
        )), Times.Once);
    }

    [Test]
    public async Task ReplaceAsync_ReturnsNotFound_WhenSessionMissing() {
        _dataLayer.Setup(x => x.GetByIdAsync("sess-x"))
            .ReturnsAsync((OnboardingSession?)null);

        var result = await _service.ReplaceAsync("sess-x", new OnboardingSessionUpsertDto());

        Assert.That(result.Successful, Is.False);
        Assert.That(result.ErrorType, Is.EqualTo(DataUpdateErrorType.NotFound));
        _dataLayer.Verify(x => x.ReplaceAsync(It.IsAny<string>(), It.IsAny<OnboardingSession>()),
            Times.Never);
    }

    [Test]
    public async Task ReplaceAsync_ReturnsAlreadyApproved_WhenSessionIsApproved() {
        var approved = new OnboardingSession {
            ScopeOfWork = new ScopeOfWork { ApprovedAt = DateTimeOffset.UtcNow }
        };

        _dataLayer.Setup(x => x.GetByIdAsync("sess-2")).ReturnsAsync(approved);

        var result = await _service.ReplaceAsync("sess-2", new OnboardingSessionUpsertDto());

        Assert.That(result.Successful, Is.False);
        Assert.That(result.ErrorType, Is.EqualTo(DataUpdateErrorType.NameConflict));
        _dataLayer.Verify(x => x.ReplaceAsync(It.IsAny<string>(), It.IsAny<OnboardingSession>()),
            Times.Never);
    }
```

```bash
dotnet test Kochava.Aim.Portal.Tests/ --filter "OnboardingSessionServiceTests"
# Expected: compile error — OnboardingSessionService does not exist
```

---

### T3 — Add static `DataUpdateResult.AlreadyApproved`

The terminal-approval guard returns `AlreadyApproved` (backed by `NameConflict` so
B4's switch naturally maps it to HTTP 409).

```csharp
// Kochava.Aim.Portal.Models/ActionResults/DataUpdateResult.cs
// Add inside the DataUpdateResult class, alongside the existing statics:

public static readonly DataUpdateResult AlreadyApproved = new(
    "This onboarding session has been approved and cannot be modified.",
    DataUpdateErrorType.NameConflict
);
```

```bash
dotnet build Kochava.Aim.Portal.Models/
# Expected: success
```

---

### T4 — Add `OnboardingSessionDto`

```csharp
// Kochava.Aim.Portal.Models/DTOs/OnboardingSessionDto.cs
using System;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;

namespace Kochava.Aim.Portal.Models.DTOs;

public class OnboardingSessionDto {

    public string Id { get; set; }

    public string AdvertiserId { get; set; }

    public string Status { get; set; }

    public DateTimeOffset CreatedAt { get; set; }

    public DateTimeOffset UpdatedAt { get; set; }

    public string CreatedByUserId { get; set; }

    public string UpdatedByUserId { get; set; }

    public WizardAnswers Wizard { get; set; } = new();

    public TierState Tier { get; set; } = new();

    public ScopeOfWork ScopeOfWork { get; set; } = new();

    public TimelineState Timeline { get; set; } = new();

    public IList<ChangeLogEntry> ChangeLog { get; set; } = new List<ChangeLogEntry>();

}
```

> The embedded types (`WizardAnswers`, `TierState`, `ScopeOfWork`, `TimelineState`,
> `ChangeLogEntry`) are the POCO types defined in B1. The DTO exposes them directly
> without re-wrapping — the same pattern `SavedViewDto` uses for `SavedViewFiltersDto`
> / `SavedViewStateDto`. If B1 introduces separate DTO-side equivalents, update the
> property types accordingly.

```bash
dotnet build Kochava.Aim.Portal.Models/
# Expected: success
```

---

### T5 — Add `OnboardingSessionUpsertDto`

```csharp
// Kochava.Aim.Portal.Models/DTOs/OnboardingSessionUpsertDto.cs
using Kochava.Aim.Portal.DataAccess.Mongo.Models;

namespace Kochava.Aim.Portal.Models.DTOs;

public class OnboardingSessionUpsertDto {

    public string Status { get; set; }

    public WizardAnswers Wizard { get; set; } = new();

    public TierState Tier { get; set; } = new();

    public ScopeOfWork ScopeOfWork { get; set; } = new();

    public TimelineState Timeline { get; set; } = new();

}
```

> `AdvertiserId` is intentionally absent — stamped server-side from
> `ICurrentUserContext.ActiveAdvertiserIdAsString`. `CreatedAt` / `UpdatedAt` /
> `*ByUserId` and `ChangeLog` are also server-stamped; clients never send them.

```bash
dotnet build Kochava.Aim.Portal.Models/
# Expected: success
```

---

### T6 — Add `IOnboardingSessionService`

```csharp
// Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs
using System.Threading.Tasks;
using Kochava.Aim.Portal.Models.ActionResults;
using Kochava.Aim.Portal.Models.DTOs;

namespace Kochava.Aim.Portal.Services.Interfaces;

public interface IOnboardingSessionService : ISingleInstanceService {

    /// <summary>Returns the latest session for the active advertiser, or null if none exists.</summary>
    Task<OnboardingSessionDto?> GetLatestAsync();

    /// <summary>
    /// Creates a new session for the active advertiser.
    /// Returns AlreadyApproved (NameConflict) if an approved session already exists.
    /// </summary>
    Task<DataUpdateResult<OnboardingSessionDto>> CreateAsync(OnboardingSessionUpsertDto dto);

    /// <summary>
    /// Full-document replace. Stamps UpdatedAt and UpdatedByUserId server-side.
    /// Returns NotFound if the session does not exist.
    /// Returns AlreadyApproved (NameConflict) if ApprovedAt is set — answers are terminal.
    /// </summary>
    Task<DataUpdateResult> ReplaceAsync(string id, OnboardingSessionUpsertDto dto);

}
```

```bash
dotnet build Kochava.Aim.Portal.Services/
# Expected: success (interface only)
```

---

### T7 — Implement `OnboardingSessionService`

```csharp
// Kochava.Aim.Portal.Services/OnboardingSessionService.cs
using System;
using System.Threading.Tasks;
using AutoMapper;
using Kochava.Aim.Portal.Common.Identity.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.DataLayers.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using Kochava.Aim.Portal.Models.ActionResults;
using Kochava.Aim.Portal.Models.DTOs;
using Kochava.Aim.Portal.Services.Interfaces;

namespace Kochava.Aim.Portal.Services;

public class OnboardingSessionService : IOnboardingSessionService {

    private readonly IOnboardingSessionDataLayer _dataLayer;
    private readonly IMapper _mapper;
    private readonly Func<ICurrentUserContext> _currentUser;

    public OnboardingSessionService(
        IOnboardingSessionDataLayer dataLayer,
        IMapper mapper,
        Func<ICurrentUserContext> currentUser
    ) {
        _dataLayer   = dataLayer;
        _mapper      = mapper;
        _currentUser = currentUser;
    }

    public async Task<OnboardingSessionDto?> GetLatestAsync() {
        var user    = _currentUser();
        var session = await _dataLayer.GetLatestByAdvertiserAsync(user.ActiveAdvertiserIdAsString);
        return session == null ? null : _mapper.Map<OnboardingSessionDto>(session);
    }

    public async Task<DataUpdateResult<OnboardingSessionDto>> CreateAsync(OnboardingSessionUpsertDto dto) {
        var user     = _currentUser();
        var existing = await _dataLayer.GetLatestByAdvertiserAsync(user.ActiveAdvertiserIdAsString);

        if (existing?.ScopeOfWork?.ApprovedAt != null) {
            return DataUpdateResult<OnboardingSessionDto>.FromResult(DataUpdateResult.AlreadyApproved);
        }

        var now   = DateTimeOffset.UtcNow;
        var model = _mapper.Map<OnboardingSession>(dto);
        model.AdvertiserId     = user.ActiveAdvertiserIdAsString;
        model.CreatedByUserId  = user.UserId;
        model.UpdatedByUserId  = user.UserId;
        model.CreatedAt        = now;
        model.UpdatedAt        = now;

        var created = await _dataLayer.CreateAsync(model);
        return DataUpdateResult<OnboardingSessionDto>.FromValue(_mapper.Map<OnboardingSessionDto>(created));
    }

    public async Task<DataUpdateResult> ReplaceAsync(string id, OnboardingSessionUpsertDto dto) {
        var existing = await _dataLayer.GetByIdAsync(id);
        if (existing == null) { return DataUpdateResult.NotFound; }
        if (existing.ScopeOfWork?.ApprovedAt != null) { return DataUpdateResult.AlreadyApproved; }

        var user  = _currentUser();
        var model = _mapper.Map<OnboardingSession>(dto);
        model.UpdatedAt        = DateTimeOffset.UtcNow;
        model.UpdatedByUserId  = user.UserId;

        return await _dataLayer.ReplaceAsync(id, model);
    }

}
```

```bash
dotnet build Kochava.Aim.Portal.Services/
# Expected: success
```

---

### T8 — Run tests: all pass

```bash
dotnet test Kochava.Aim.Portal.Tests/ --filter "OnboardingSessionServiceTests"
# Expected: 7 tests pass, 0 failures
```

---

### T9 — Lint-clean build

```bash
dotnet build
# Expected: zero warnings under TreatWarningsAsErrors
```

---

### T10 — Commit

```bash
git add \
  Kochava.Aim.Portal.Models/ActionResults/DataUpdateResult.cs \
  Kochava.Aim.Portal.Models/DTOs/OnboardingSessionDto.cs \
  Kochava.Aim.Portal.Models/DTOs/OnboardingSessionUpsertDto.cs \
  Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs \
  Kochava.Aim.Portal.Services/OnboardingSessionService.cs \
  Kochava.Aim.Portal.Tests/OnboardingSessionServiceTests.cs

git commit -m "feat(onboarding): add DTOs + OnboardingSessionService (B3)

Mirror SavedViewsService. Service enforces terminal-approval guard:
create and full-doc replace return AlreadyApproved (NameConflict →
HTTP 409 in B4) when ScopeOfWork.ApprovedAt is set.
Stamps UpdatedAt/UpdatedByUserId server-side on every write.
7 NUnit tests covering get-latest, create, replace, not-found,
and approval-terminal guard for both write paths."
```

---

## Summary

| # | Test | Guard path |
|---|------|------------|
| 1 | `GetLatestAsync_ReturnsNull_WhenNoSession` | — |
| 2 | `GetLatestAsync_ReturnsMappedDto_WhenSessionExists` | — |
| 3 | `CreateAsync_ReturnsDto_WhenNoExistingSession` | happy path |
| 4 | `CreateAsync_ReturnsAlreadyApproved_WhenSessionIsApproved` | terminal guard |
| 5 | `ReplaceAsync_StampsTimestampsAndDelegates_WhenNotApproved` | happy path |
| 6 | `ReplaceAsync_ReturnsNotFound_WhenSessionMissing` | not-found |
| 7 | `ReplaceAsync_ReturnsAlreadyApproved_WhenSessionIsApproved` | terminal guard |

**B4 note:** the controller dispatches `DataUpdateErrorType.NameConflict → Conflict(409)`
in its own switch arm, consistent with `AppsController` and `AccountsController`.
No changes to `AppControllerBase` or `DataUpdateErrorType` are required in this plan.

**AutoMapper profile** (`OnboardingSession ↔ DTOs`) is deferred to B2; all tests here
mock `IMapper` directly.
