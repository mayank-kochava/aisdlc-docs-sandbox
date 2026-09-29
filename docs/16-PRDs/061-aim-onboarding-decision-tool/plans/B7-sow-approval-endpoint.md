---
id: plan-b7
title: "B7 — SoW Approval Endpoint (Terminal)"
---

## B7 — SoW Approval Endpoint (Terminal) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `PUT /{id}/sow-approval` to `OnboardingSessionsController` — the client submits `ApproverName` + `ApproverJobTitle` + client-provided `ApprovedAt` timestamp, the server sets those fields, sets `Status = "complete"`, anchors the timeline (`Timeline.AnchorDate = ApprovedAt`), and appends a `changeLog` entry stamped with the server's UTC time and the authenticated user ID; calling this endpoint on an already-approved session returns 409.

**Architecture:** Thin controller action delegates to `OnboardingSessionService.ApproveSowAsync`, which calls a new data-layer method. The data layer uses a MongoDB conditional update (`ApprovedAt == null` in the filter) to ensure atomicity — the operation is self-serialising without a separate lock document. A `DataUpdateResult` with error type `NameConflict` (re-used as the existing "conflict/already-done" signal) carries the 409 case back up; the controller maps it inline since `HandleDataUpdateResult` does not map `NameConflict → 409`.

**Tech Stack:** C# .NET 7, MongoDB.Driver, NUnit 3, Moq, FluentAssertions.

---

## Files

**Create**

- `Kochava.Aim.Portal.Models/DTOs/SowApprovalDto.cs` — request body shape
- `Kochava.Aim.Portal.Tests/OnboardingSessionSowApprovalTests.cs` — all NUnit tests for this plan

**Modify**

- `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs` — add `ApproveSowAsync`
- `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs` — implement `ApproveSowAsync`
- `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs` — add `ApproveSowAsync`
- `Kochava.Aim.Portal.Services/OnboardingSessionService.cs` — implement `ApproveSowAsync`
- `Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs` — add `PUT /{id}/sow-approval` action

---

## Dependencies

**B4** must be complete before this plan. B4 creates:

- `OnboardingSessionsController.cs`
- `IOnboardingSessionService.cs` / `OnboardingSessionService.cs`
- `IOnboardingSessionDataLayer.cs` / `OnboardingSessionDataLayer.cs`
- `IOnboardingSessionDataLayer` already has advertiser-scoped filter helpers you will call.

B1 creates `OnboardingSession.cs` (model). B2 creates the data-layer base with `GetAdvertiserScopedFilter()`.

---

## Design notes for the implementer

### No email on `ICurrentUserContext`

`ICurrentUserContext` exposes only `UserId` and `UserDisplayName` (set from the `x-user-id` / `x-user-name` headers in `AdvertiserContextActionFilter`). There is **no** `.Email` property. Use `user.UserId` as `changeLog[].UserEmail` — the design doc says "session email" but the identity contract here is an opaque user ID. Do **not** invent a header read inside the service.

### 409 mechanism

`HandleDataUpdateResult` in `AppControllerBase` has no `NameConflict → 409` branch (it falls through to 500). Map the conflict inline in the action method using `Conflict(new ProblemDetails {...})`, mirroring `AppsController.Post`. Use `DataUpdateErrorType.NameConflict` as the sentinel — do not add a new enum value.

### Approval is irreversible

There is no unlock endpoint (design decision 2026-06-11). Once approved, `PUT /{id}/sow-approval` returns 409 on every subsequent call. Do not add any unlocking logic here or elsewhere in this plan.

### `ApprovedAt` source

The DTO carries the client-provided timestamp (`ApprovedAt`). Store it as-is on `ScopeOfWork.ApprovedAt` and copy it to `Timeline.AnchorDate`. The `changeLog` entry's `At` field is **server-stamped** (`DateTimeOffset.UtcNow` inside the service, not from the DTO).

### `Timeline.AnchorDate` only

Est. start/end dates are computed-never-stored (design §8a). B7 only writes `Timeline.AnchorDate = ApprovedAt`. Do not recompute or seed milestone dates.

---

## Tasks

### Task 1 — Create `SowApprovalDto.cs`

**Files:**

- Create: `Kochava.Aim.Portal.Models/DTOs/SowApprovalDto.cs`

- [ ] **Step 1.1: Write the DTO**

```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace Kochava.Aim.Portal.Models.DTOs;

public class SowApprovalDto {

    [Required]
    public string ApproverName { get; set; }

    [Required]
    public string ApproverJobTitle { get; set; }

    [Required]
    public DateTimeOffset ApprovedAt { get; set; }

}
```

- [ ] **Step 1.2: Verify it compiles**

```bash
cd /path/to/mmm-portal-api
dotnet build Kochava.Aim.Portal.Models/Kochava.Aim.Portal.Models.csproj 2>&1 | tail -10
```

Expected output contains: `Build succeeded`

---

### Task 2 — Write failing service tests

**Files:**

- Create: `Kochava.Aim.Portal.Tests/OnboardingSessionSowApprovalTests.cs`

- [ ] **Step 2.1: Create the test file**

```csharp
using System;
using System.Threading.Tasks;
using FluentAssertions;
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
public class OnboardingSessionSowApprovalTests {

    private Mock<IOnboardingSessionDataLayer> _dataLayerMock;
    private Mock<ICurrentUserContext> _currentUserMock;
    private OnboardingSessionService _service;

    private const string AdvertiserId = "adv-456";
    private const string UserId = "user-xyz";
    private const string SessionId = "session-id-001";

    [SetUp]
    public void Setup() {
        _dataLayerMock = new Mock<IOnboardingSessionDataLayer>();
        _currentUserMock = new Mock<ICurrentUserContext>();

        _currentUserMock.Setup(x => x.UserId).Returns(UserId);
        _currentUserMock.Setup(x => x.ActiveAdvertiserIdAsString).Returns(AdvertiserId);

        _service = new OnboardingSessionService(
            _dataLayerMock.Object,
            () => _currentUserMock.Object
        );
    }

    // Happy path: approval sets all required fields and appends audit entry
    [Test]
    public async Task ApproveSowAsync_HappyPath_ReturnsSuccess_AndPassesCorrectModelToDataLayer() {
        var approvedAt = new DateTimeOffset(2026, 6, 15, 10, 0, 0, TimeSpan.Zero);
        var dto = new SowApprovalDto {
            ApproverName = "Jane Doe",
            ApproverJobTitle = "CMO",
            ApprovedAt = approvedAt
        };

        OnboardingSession capturedSession = null;
        _dataLayerMock
            .Setup(x => x.ApproveSowAsync(SessionId, It.IsAny<OnboardingSession>()))
            .Callback<string, OnboardingSession>((_, s) => capturedSession = s)
            .ReturnsAsync(DataUpdateResult.Success);

        var result = await _service.ApproveSowAsync(SessionId, dto);

        result.Successful.Should().BeTrue();
        capturedSession.Should().NotBeNull();
        capturedSession.ScopeOfWork.ApproverName.Should().Be("Jane Doe");
        capturedSession.ScopeOfWork.ApproverJobTitle.Should().Be("CMO");
        capturedSession.ScopeOfWork.ApprovedAt.Should().Be(approvedAt);
        capturedSession.Status.Should().Be("complete");
        capturedSession.Timeline.AnchorDate.Should().Be(approvedAt);
        capturedSession.ChangeLog.Should().HaveCount(1);
        capturedSession.ChangeLog[0].Type.Should().Be("sow_approved");
        capturedSession.ChangeLog[0].Field.Should().Be("status");
        capturedSession.ChangeLog[0].NewValue.Should().Be("complete");
        // UserEmail is stamped from the authenticated session user ID
        capturedSession.ChangeLog[0].UserEmail.Should().Be(UserId);
        // At is server-stamped — verify it's close to UtcNow (within 5 seconds)
        capturedSession.ChangeLog[0].At.Should().BeCloseTo(DateTimeOffset.UtcNow, TimeSpan.FromSeconds(5));
    }

    // Re-approval must be rejected — approval is terminal
    [Test]
    public async Task ApproveSowAsync_AlreadyApproved_ReturnsNameConflict() {
        var dto = new SowApprovalDto {
            ApproverName = "Jane Doe",
            ApproverJobTitle = "CMO",
            ApprovedAt = DateTimeOffset.UtcNow
        };

        _dataLayerMock
            .Setup(x => x.ApproveSowAsync(SessionId, It.IsAny<OnboardingSession>()))
            .ReturnsAsync(DataUpdateResult.NameConflict);

        var result = await _service.ApproveSowAsync(SessionId, dto);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.NameConflict);
    }

    // Session not found
    [Test]
    public async Task ApproveSowAsync_SessionNotFound_ReturnsNotFound() {
        var dto = new SowApprovalDto {
            ApproverName = "Jane Doe",
            ApproverJobTitle = "CMO",
            ApprovedAt = DateTimeOffset.UtcNow
        };

        _dataLayerMock
            .Setup(x => x.ApproveSowAsync(SessionId, It.IsAny<OnboardingSession>()))
            .ReturnsAsync(DataUpdateResult.NotFound);

        var result = await _service.ApproveSowAsync(SessionId, dto);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.NotFound);
    }

}
```

- [ ] **Step 2.2: Run the tests — expect compile failure**

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionSowApproval" \
  --no-build 2>&1 | tail -20
```

Expected: build error — `ApproveSowAsync` does not exist yet.

---

### Task 3 — Extend data-layer interface

**Files:**

- Modify: `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs`

- [ ] **Step 3.1: Add `ApproveSowAsync` to the interface**

Open the interface file and add the following method signature. Place it after the existing `ReplaceAsync` / autosave method:

```csharp
/// <summary>
/// Conditionally sets SoW approval fields, Status=complete, and Timeline.AnchorDate.
/// Appends the changeLog entry from <paramref name="update"/>.
/// Returns NotFound if no session matches id+advertiserId.
/// Returns NameConflict if the session is already approved (ApprovedAt != null).
/// </summary>
Task<DataUpdateResult> ApproveSowAsync(string id, OnboardingSession update);
```

The interface should already import `Kochava.Aim.Portal.Models.ActionResults` and
`Kochava.Aim.Portal.DataAccess.Mongo.Models`. Confirm these `using` statements are present at the top; add them if missing.

- [ ] **Step 3.2: Confirm build**

```bash
cd /path/to/mmm-portal-api
dotnet build Kochava.Aim.Portal.DataAccess/Kochava.Aim.Portal.DataAccess.csproj 2>&1 | tail -10
```

Expected: compile error in `OnboardingSessionDataLayer.cs` — interface not implemented yet. That is correct.

---

### Task 4 — Implement `ApproveSowAsync` in the data layer

**Files:**

- Modify: `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs`

- [ ] **Step 4.1: Add the implementation**

Open `OnboardingSessionDataLayer.cs`. Add the following method. The file already has a `Collection()` helper and a `GetAdvertiserScopedFilter()` helper (both created in B2). Add `ApproveSowAsync` after the existing `ReplaceAsync` method:

```csharp
public async Task<DataUpdateResult> ApproveSowAsync(string id, OnboardingSession update) {
    // Filter: advertiser-scoped + matching id + not yet approved (atomic guard)
    var filter = GetAdvertiserScopedFilter()
        & Builders<OnboardingSession>.Filter.Eq(x => x.Id, id)
        & Builders<OnboardingSession>.Filter.Eq(x => x.ScopeOfWork.ApprovedAt, (DateTimeOffset?)null);

    var changeEntry = update.ChangeLog[^1]; // service appends exactly one entry before calling

    var mongoUpdate = Builders<OnboardingSession>.Update
        .Set(x => x.Status, "complete")
        .Set(x => x.ScopeOfWork.ApproverName, update.ScopeOfWork.ApproverName)
        .Set(x => x.ScopeOfWork.ApproverJobTitle, update.ScopeOfWork.ApproverJobTitle)
        .Set(x => x.ScopeOfWork.ApprovedAt, update.ScopeOfWork.ApprovedAt)
        .Set(x => x.Timeline.AnchorDate, update.ScopeOfWork.ApprovedAt)
        .Set(x => x.UpdatedAt, DateTimeOffset.UtcNow)
        .Push(x => x.ChangeLog, changeEntry);

    var result = await Collection().UpdateOneAsync(filter, mongoUpdate);

    if (result.MatchedCount == 0) {
        // Distinguish not-found from already-approved: check if the doc exists at all
        var existsFilter = GetAdvertiserScopedFilter()
            & Builders<OnboardingSession>.Filter.Eq(x => x.Id, id);
        var exists = await Collection().CountDocumentsAsync(existsFilter) > 0;
        return exists ? DataUpdateResult.NameConflict : DataUpdateResult.NotFound;
    }

    return DataUpdateResult.Success;
}
```

- [ ] **Step 4.2: Build the data-access project**

```bash
cd /path/to/mmm-portal-api
dotnet build Kochava.Aim.Portal.DataAccess/Kochava.Aim.Portal.DataAccess.csproj 2>&1 | tail -10
```

Expected: `Build succeeded`

---

### Task 5 — Extend service interface

**Files:**

- Modify: `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs`

- [ ] **Step 5.1: Add `ApproveSowAsync` to the interface**

Open `IOnboardingSessionService.cs` and add after the existing method signatures:

```csharp
/// <summary>
/// Validates and persists SoW approval. Terminal — sets Status=complete and locks wizard answers.
/// Returns NameConflict if already approved, NotFound if session doesn't exist.
/// </summary>
Task<DataUpdateResult> ApproveSowAsync(string id, SowApprovalDto dto);
```

Confirm the file imports `Kochava.Aim.Portal.Models.ActionResults` and `Kochava.Aim.Portal.Models.DTOs`; add them if missing.

- [ ] **Step 5.2: Build the services project**

```bash
cd /path/to/mmm-portal-api
dotnet build Kochava.Aim.Portal.Services/Kochava.Aim.Portal.Services.csproj 2>&1 | tail -10
```

Expected: compile error in `OnboardingSessionService.cs` — interface not yet implemented. Correct.

---

### Task 6 — Implement `ApproveSowAsync` in the service

**Files:**

- Modify: `Kochava.Aim.Portal.Services/OnboardingSessionService.cs`

- [ ] **Step 6.1: Add the implementation**

Open `OnboardingSessionService.cs`. Add the following method after the existing `ReplaceAsync` implementation:

```csharp
public async Task<DataUpdateResult> ApproveSowAsync(string id, SowApprovalDto dto) {
    var user = _currentUser();

    // Build the update model — service owns audit stamping, data layer owns persistence
    var update = new OnboardingSession {
        ScopeOfWork = new ScopeOfWork {
            ApproverName   = dto.ApproverName,
            ApproverJobTitle = dto.ApproverJobTitle,
            ApprovedAt     = dto.ApprovedAt   // client-provided timestamp stored as-is
        }
    };

    // Audit entry: At and UserEmail are server-stamped regardless of role
    update.ChangeLog.Add(new ChangeLogEntry {
        At        = DateTimeOffset.UtcNow,     // server clock — not from dto
        UserEmail = user.UserId,               // ICurrentUserContext has no .Email; UserId is the identity
        Type      = "sow_approved",
        Field     = "status",
        OldValue  = "in_progress",
        NewValue  = "complete"
    });

    return await _dataLayer.ApproveSowAsync(id, update);
}
```

`_dataLayer` is the `IOnboardingSessionDataLayer` field injected in B4. Confirm its field name matches what B4 established (typically `_dataLayer` or `_onboardingSessionDataLayer`); update the reference if different.

- [ ] **Step 6.2: Build the services project**

```bash
cd /path/to/mmm-portal-api
dotnet build Kochava.Aim.Portal.Services/Kochava.Aim.Portal.Services.csproj 2>&1 | tail -10
```

Expected: `Build succeeded`

---

### Task 7 — Run service tests — expect pass

- [ ] **Step 7.1: Run the failing tests from Task 2**

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionSowApproval" 2>&1 | tail -20
```

Expected: all three tests pass — `HappyPath_ReturnsSuccess`, `AlreadyApproved_ReturnsNameConflict`, `SessionNotFound_ReturnsNotFound`.

If a test fails, read the error before proceeding.

---

### Task 8 — Add controller action

**Files:**

- Modify: `Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs`

- [ ] **Step 8.1: Add the `SowApproval` action**

Open `OnboardingSessionsController.cs`. Add the following action after the existing `Put` (autosave) action. This endpoint is open to both advertiser and CSM — no `is_as_admin` check here:

```csharp
[HttpPut("{id}/sow-approval")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
[ProducesResponseType(StatusCodes.Status409Conflict)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<ActionResult> SowApproval(string id, [FromBody] SowApprovalDto dto) {
    var result = await _onboardingSessionService.ApproveSowAsync(id, dto);

    if (result.Successful) {
        return NoContent();
    }

    return result.ErrorType switch {
        DataUpdateErrorType.NotFound     => NotFoundWithReason(result.ErrorMessage),
        DataUpdateErrorType.NameConflict => Conflict(new ProblemDetails {
            Title  = "Conflict",
            Detail = "This session has already been approved. Approval is terminal and cannot be repeated.",
            Status = StatusCodes.Status409Conflict
        }),
        _ => StatusCode(StatusCodes.Status500InternalServerError, new ProblemDetails {
            Title  = "Internal Server Error",
            Detail = result.ErrorMessage,
            Status = StatusCodes.Status500InternalServerError
        })
    };
}
```

The file already imports `Kochava.Aim.Portal.Models.ActionResults`, `Kochava.Aim.Portal.Models.DTOs`, and `Microsoft.AspNetCore.Http`. Add any missing `using` statements if the build fails.

- [ ] **Step 8.2: Build the API project**

```bash
cd /path/to/mmm-portal-api
dotnet build Kochava.Aim.Portal.Api/Kochava.Aim.Portal.Api.csproj 2>&1 | tail -10
```

Expected: `Build succeeded`

---

### Task 9 — Full test suite

- [ ] **Step 9.1: Run the full solution test suite**

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -30
```

Expected: all tests pass, no regressions. If any test outside `OnboardingSessionSowApproval` fails, stop and diagnose before committing.

---

### Task 10 — Commit

- [ ] **Step 10.1: Stage and commit**

```bash
git add \
  Kochava.Aim.Portal.Models/DTOs/SowApprovalDto.cs \
  Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs \
  Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs \
  Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs \
  Kochava.Aim.Portal.Services/OnboardingSessionService.cs \
  Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs \
  Kochava.Aim.Portal.Tests/OnboardingSessionSowApprovalTests.cs

git commit -m "feat(onboarding): add PUT /sow-approval — terminal lock, Status=complete, anchor timeline (B7)"
```
