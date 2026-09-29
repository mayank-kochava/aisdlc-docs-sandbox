---
id: plan-b6
title: "B6 — CSM Action Endpoints"
---

## Goal

Add `POST /{id}/approve` and `POST /{id}/tier-override` to `OnboardingSessionsController`.
Both are CSM-only: the B5 gate (`IsAsAdmin`) inside each service method returns
`DataUpdateResult.AccessDenied` → 403 via `HandleDataUpdateResult`/`ForbidWithReason`.
Every action appends a `ChangeLogEntry` server-side with `UserEmail` from the authenticated
session — never from the request body.

**Tier-override asymmetry** (product-spec-v2.md §4.3, Gary PR comment B10):

- AIM Pro → AIM X: always permitted (downgrade).
- AIM X → AIM Pro: only permitted when `tier.recommended == "aim_pro"` (the algorithm already
  reached Pro; the CSM is unlocking an override rather than upgrading unilaterally).

Any call that violates this constraint returns 422.

---

## Files

**Create**

- `Kochava.Aim.Portal.Tests/OnboardingSessionsServiceCsmTests.cs` — service-level tests
- `Kochava.Aim.Portal.Tests/OnboardingSessionsControllerCsmTests.cs` — controller pass-through tests

**Modify**

- `Kochava.Aim.Portal.Models/ActionResults/DataUpdateErrorType.cs` — add `UnprocessableEntity`
- `Kochava.Aim.Portal.Api/Areas/App/Controllers/AppControllerBase.cs` — map new error type
- `Kochava.Aim.Portal.Models/DTOs/TierOverrideDto.cs` — create request body DTO
- `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs` — add two methods
- `Kochava.Aim.Portal.Services/OnboardingSessionService.cs` — implement
- `Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs` — add actions

---

## Dependencies

- **B4** — `OnboardingSessionsController` and `IOnboardingSessionService` exist; GET/POST/PUT wired.
- **B5** — `ICurrentUserContext` is extended with:
    - `bool IsAsAdmin` — set from `is_as_admin` mos-iam resolution
    - `string UserEmail` — set from session identity (the caller's email, server-resolved)

    These symbols are referenced verbatim below. If B5 uses different names, update here before
    implementing.

---

## Tasks

### Task 1 — Write the failing service tests

Service tests use a real `OnboardingSessionService` with mocked data layer and user context.
This is where the three required properties (403 enforcement, audit append, asymmetric rule) are
actually exercised.

**1.1** Create `Kochava.Aim.Portal.Tests/OnboardingSessionsServiceCsmTests.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using FluentAssertions;
using Kochava.Aim.Portal.Common.Identity.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.DataLayers.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using Kochava.Aim.Portal.Models.ActionResults;
using Kochava.Aim.Portal.Services;
using Moq;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class OnboardingSessionsServiceCsmTests {

    private const string SessionId   = "6650000000000000000aaaaa";
    private const string CsmEmail    = "csm@kochava.com";
    private const string ClientEmail = "client@example.com";

    private Mock<IOnboardingSessionDataLayer> _dataLayerMock;
    private Mock<ICurrentUserContext>         _csmCtx;
    private Mock<ICurrentUserContext>         _clientCtx;

    [SetUp]
    public void Setup() {
        _dataLayerMock = new Mock<IOnboardingSessionDataLayer>();

        _csmCtx = new Mock<ICurrentUserContext>();
        _csmCtx.Setup(x => x.IsAsAdmin).Returns(true);
        _csmCtx.Setup(x => x.UserEmail).Returns(CsmEmail);
        _csmCtx.Setup(x => x.UserId).Returns("csm-id");

        _clientCtx = new Mock<ICurrentUserContext>();
        _clientCtx.Setup(x => x.IsAsAdmin).Returns(false);
        _clientCtx.Setup(x => x.UserEmail).Returns(ClientEmail);
        _clientCtx.Setup(x => x.UserId).Returns("client-id");
    }

    private OnboardingSessionService BuildService(ICurrentUserContext ctx) =>
        new(ctx, _dataLayerMock.Object);

    private OnboardingSession SessionWith(string recommended, string currentOverride = null) =>
        new() {
            Id           = SessionId,
            AdvertiserId = "adv-42",
            Tier         = new Tier {
                Recommended = recommended,
                Override    = currentOverride,
                Effective   = currentOverride ?? recommended
            },
            ScopeOfWork = new ScopeOfWork { CsmApproved = false },
            ChangeLog   = new List<ChangeLogEntry>()
        };

    // ── approve: access control ──────────────────────────────────────────────

    [Test]
    public async Task Approve_NonAdmin_ReturnsAccessDenied_DataLayerNotCalled() {
        var svc    = BuildService(_clientCtx.Object);
        var result = await svc.ApproveAsync(SessionId, ClientEmail);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.AccessDenied);

        _dataLayerMock.Verify(
            x => x.GetByIdAsync(It.IsAny<string>()),
            Times.Never
        );
    }

    // ── approve: happy path ──────────────────────────────────────────────────

    [Test]
    public async Task Approve_Admin_SetsCsmApproved_And_WritesChangeLogEntry() {
        var session = SessionWith("aim_x");
        _dataLayerMock.Setup(x => x.GetByIdAsync(SessionId)).ReturnsAsync(session);
        _dataLayerMock.Setup(x => x.ReplaceAsync(session)).Returns(Task.CompletedTask);

        var svc    = BuildService(_csmCtx.Object);
        var result = await svc.ApproveAsync(SessionId, CsmEmail);

        result.Successful.Should().BeTrue();
        session.ScopeOfWork.CsmApproved.Should().BeTrue();

        session.ChangeLog.Should().HaveCount(1);
        var entry = session.ChangeLog[0];
        entry.UserEmail.Should().Be(CsmEmail);
        entry.Type.Should().Be("csm_approved");
        entry.Field.Should().Be("scopeOfWork.csmApproved");
        entry.OldValue.Should().Be("false");
        entry.NewValue.Should().Be("true");

        _dataLayerMock.Verify(x => x.ReplaceAsync(session), Times.Once);
    }

    // ── approve: session not found ────────────────────────────────────────────

    [Test]
    public async Task Approve_SessionNotFound_ReturnsNotFound() {
        _dataLayerMock.Setup(x => x.GetByIdAsync(SessionId))
                      .ReturnsAsync((OnboardingSession)null);

        var svc    = BuildService(_csmCtx.Object);
        var result = await svc.ApproveAsync(SessionId, CsmEmail);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.NotFound);
        _dataLayerMock.Verify(x => x.ReplaceAsync(It.IsAny<OnboardingSession>()), Times.Never);
    }

    // ── tier-override: access control ────────────────────────────────────────

    [Test]
    public async Task TierOverride_NonAdmin_ReturnsAccessDenied_DataLayerNotCalled() {
        var svc    = BuildService(_clientCtx.Object);
        var result = await svc.TierOverrideAsync(SessionId, "aim_pro", ClientEmail);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.AccessDenied);

        _dataLayerMock.Verify(
            x => x.GetByIdAsync(It.IsAny<string>()),
            Times.Never
        );
    }

    // ── tier-override: asymmetry (blocked direction) ──────────────────────────

    [Test]
    public async Task TierOverride_XToProWhenAutoIsX_ReturnsUnprocessableEntity() {
        // aim_x auto-rec: upgrading to aim_pro is blocked
        var session = SessionWith(recommended: "aim_x");
        _dataLayerMock.Setup(x => x.GetByIdAsync(SessionId)).ReturnsAsync(session);

        var svc    = BuildService(_csmCtx.Object);
        var result = await svc.TierOverrideAsync(SessionId, "aim_pro", CsmEmail);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.UnprocessableEntity);
        _dataLayerMock.Verify(x => x.ReplaceAsync(It.IsAny<OnboardingSession>()), Times.Never);
        session.ChangeLog.Should().BeEmpty();
    }

    // ── tier-override: allowed paths ──────────────────────────────────────────

    [Test]
    public async Task TierOverride_XToProWhenAutoIsPro_Succeeds_And_WritesAuditEntry() {
        // aim_pro auto-rec + current effective=aim_x → allowed upgrade
        var session = SessionWith(recommended: "aim_pro");
        _dataLayerMock.Setup(x => x.GetByIdAsync(SessionId)).ReturnsAsync(session);
        _dataLayerMock.Setup(x => x.ReplaceAsync(session)).Returns(Task.CompletedTask);

        var svc    = BuildService(_csmCtx.Object);
        var result = await svc.TierOverrideAsync(SessionId, "aim_pro", CsmEmail);

        result.Successful.Should().BeTrue();
        session.Tier.Override.Should().Be("aim_pro");
        session.Tier.Effective.Should().Be("aim_pro");

        session.ChangeLog.Should().HaveCount(1);
        var entry = session.ChangeLog[0];
        entry.UserEmail.Should().Be(CsmEmail);
        entry.Type.Should().Be("tier_override");
        entry.Field.Should().Be("tier.override");
        entry.NewValue.Should().Be("aim_pro");

        _dataLayerMock.Verify(x => x.ReplaceAsync(session), Times.Once);
    }

    [Test]
    public async Task TierOverride_ProToX_Succeeds_Always() {
        // Downgrade aim_pro → aim_x is always permitted
        var session = SessionWith(recommended: "aim_pro", currentOverride: "aim_pro");
        _dataLayerMock.Setup(x => x.GetByIdAsync(SessionId)).ReturnsAsync(session);
        _dataLayerMock.Setup(x => x.ReplaceAsync(session)).Returns(Task.CompletedTask);

        var svc    = BuildService(_csmCtx.Object);
        var result = await svc.TierOverrideAsync(SessionId, "aim_x", CsmEmail);

        result.Successful.Should().BeTrue();
        session.Tier.Override.Should().Be("aim_x");
        session.Tier.Effective.Should().Be("aim_x");

        session.ChangeLog.Should().HaveCount(1);
        var entry = session.ChangeLog[0];
        entry.UserEmail.Should().Be(CsmEmail);
        entry.Type.Should().Be("tier_override");
        entry.NewValue.Should().Be("aim_x");

        _dataLayerMock.Verify(x => x.ReplaceAsync(session), Times.Once);
    }

    // ── tier-override: audit author is session-only ──────────────────────────

    [Test]
    public async Task TierOverride_AuditEntry_UserEmail_IsServerStamped_FromSessionContext() {
        // Confirm audit UserEmail == session email passed by controller, not any other value
        var session = SessionWith(recommended: "aim_pro");
        _dataLayerMock.Setup(x => x.GetByIdAsync(SessionId)).ReturnsAsync(session);
        _dataLayerMock.Setup(x => x.ReplaceAsync(session)).Returns(Task.CompletedTask);

        var svc = BuildService(_csmCtx.Object);
        await svc.TierOverrideAsync(SessionId, "aim_pro", CsmEmail);

        session.ChangeLog[0].UserEmail.Should().Be(CsmEmail);
        session.ChangeLog[0].UserEmail.Should().NotBe(ClientEmail);
    }

    // ── tier-override: session not found ─────────────────────────────────────

    [Test]
    public async Task TierOverride_SessionNotFound_ReturnsNotFound() {
        _dataLayerMock.Setup(x => x.GetByIdAsync(SessionId))
                      .ReturnsAsync((OnboardingSession)null);

        var svc    = BuildService(_csmCtx.Object);
        var result = await svc.TierOverrideAsync(SessionId, "aim_pro", CsmEmail);

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.NotFound);
        _dataLayerMock.Verify(x => x.ReplaceAsync(It.IsAny<OnboardingSession>()), Times.Never);
    }
}
```

**1.2** Create `Kochava.Aim.Portal.Tests/OnboardingSessionsControllerCsmTests.cs` — verifies
controller maps `DataUpdateResult` cases to correct HTTP status codes:

```csharp
using System.Threading.Tasks;
using FluentAssertions;
using Kochava.Aim.Portal.Api.Areas.App.Controllers;
using Kochava.Aim.Portal.Common.Identity.Interfaces;
using Kochava.Aim.Portal.Models.ActionResults;
using Kochava.Aim.Portal.Models.DTOs;
using Kochava.Aim.Portal.Services.Interfaces;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Routing;
using Moq;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class OnboardingSessionsControllerCsmTests {

    private const string AdvertiserId = "adv-42";
    private const string SessionId    = "6650000000000000000aaaaa";
    private const string CsmEmail     = "csm@kochava.com";

    private Mock<IOnboardingSessionService> _serviceMock;
    private Mock<ICurrentUserContext>       _userMock;
    private OnboardingSessionsController   _controller;

    [SetUp]
    public void Setup() {
        _serviceMock = new Mock<IOnboardingSessionService>();

        _userMock = new Mock<ICurrentUserContext>();
        _userMock.Setup(x => x.UserEmail).Returns(CsmEmail);
        _userMock.Setup(x => x.IsAsAdmin).Returns(true);

        _controller = new OnboardingSessionsController(
            () => _userMock.Object,
            _serviceMock.Object
        );
        _controller.ControllerContext = new ControllerContext {
            HttpContext = new DefaultHttpContext(),
            RouteData   = new RouteData()
        };
    }

    [Test]
    public async Task Approve_ServiceReturnsSuccess_Returns204() {
        _serviceMock.Setup(x => x.ApproveAsync(SessionId, CsmEmail))
                    .ReturnsAsync(DataUpdateResult.Success);

        var result = await _controller.Approve(AdvertiserId, SessionId);

        ((NoContentResult)result).StatusCode.Should().Be(StatusCodes.Status204NoContent);
        _serviceMock.Verify(x => x.ApproveAsync(SessionId, CsmEmail), Times.Once);
    }

    [Test]
    public async Task Approve_ServiceReturnsAccessDenied_Returns403() {
        _serviceMock.Setup(x => x.ApproveAsync(SessionId, CsmEmail))
                    .ReturnsAsync(DataUpdateResult.AccessDenied);

        var result = await _controller.Approve(AdvertiserId, SessionId);

        ((ObjectResult)result).StatusCode.Should().Be(StatusCodes.Status403Forbidden);
    }

    [Test]
    public async Task Approve_ServiceReturnsNotFound_Returns404() {
        _serviceMock.Setup(x => x.ApproveAsync(SessionId, CsmEmail))
                    .ReturnsAsync(DataUpdateResult.NotFound);

        var result = await _controller.Approve(AdvertiserId, SessionId);

        ((ObjectResult)result).StatusCode.Should().Be(StatusCodes.Status404NotFound);
    }

    [Test]
    public async Task TierOverride_ServiceReturnsSuccess_Returns204() {
        var dto = new TierOverrideDto { Override = "aim_pro" };
        _serviceMock.Setup(x => x.TierOverrideAsync(SessionId, "aim_pro", CsmEmail))
                    .ReturnsAsync(DataUpdateResult.Success);

        var result = await _controller.TierOverride(AdvertiserId, SessionId, dto);

        ((NoContentResult)result).StatusCode.Should().Be(StatusCodes.Status204NoContent);
        _serviceMock.Verify(x => x.TierOverrideAsync(SessionId, "aim_pro", CsmEmail), Times.Once);
    }

    [Test]
    public async Task TierOverride_ServiceReturnsAccessDenied_Returns403() {
        var dto = new TierOverrideDto { Override = "aim_pro" };
        _serviceMock.Setup(x => x.TierOverrideAsync(SessionId, "aim_pro", CsmEmail))
                    .ReturnsAsync(DataUpdateResult.AccessDenied);

        var result = await _controller.TierOverride(AdvertiserId, SessionId, dto);

        ((ObjectResult)result).StatusCode.Should().Be(StatusCodes.Status403Forbidden);
    }

    [Test]
    public async Task TierOverride_ServiceReturnsUnprocessableEntity_Returns422() {
        var dto = new TierOverrideDto { Override = "aim_pro" };
        _serviceMock.Setup(x => x.TierOverrideAsync(SessionId, "aim_pro", CsmEmail))
                    .ReturnsAsync(new DataUpdateResult(
                        "Asymmetric constraint violated.",
                        DataUpdateErrorType.UnprocessableEntity
                    ));

        var result = await _controller.TierOverride(AdvertiserId, SessionId, dto);

        ((ObjectResult)result).StatusCode.Should().Be(StatusCodes.Status422UnprocessableEntity);
    }

    [Test]
    public async Task TierOverride_PassesSessionEmail_NotAnyClientValue() {
        // Controller derives author from CurrentUser.UserEmail — never request body
        var dto = new TierOverrideDto { Override = "aim_x" };
        _serviceMock.Setup(x => x.TierOverrideAsync(SessionId, "aim_x", CsmEmail))
                    .ReturnsAsync(DataUpdateResult.Success);

        await _controller.TierOverride(AdvertiserId, SessionId, dto);

        _serviceMock.Verify(
            x => x.TierOverrideAsync(SessionId, "aim_x", CsmEmail),
            Times.Once
        );
        _serviceMock.Verify(
            x => x.TierOverrideAsync(
                SessionId, "aim_x",
                It.Is<string>(e => e != CsmEmail)),
            Times.Never
        );
    }
}
```

**1.3** Run (expect compile failure — new types and methods don't exist yet):

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionsServiceCsm|OnboardingSessionsControllerCsm" \
  --no-build 2>&1 | tail -20
```

---

### Task 2 — Add `UnprocessableEntity` to `DataUpdateErrorType`

Read `Kochava.Aim.Portal.Models/Exceptions/DataUpdateErrorType.cs` (or whichever file defines
the enum), then append `UnprocessableEntity` if not already present:

```csharp
UnprocessableEntity,
```

Then read `AppControllerBase.cs` and extend `HandleDataUpdateResult(DataUpdateResult result)` to
map the new case before the default `_` arm:

```csharp
DataUpdateErrorType.UnprocessableEntity => UnprocessableEntityWithReason(result.ErrorMessage),
```

Add the helper in `AppControllerBase`:

```csharp
protected ObjectResult UnprocessableEntityWithReason(string reason) =>
    StatusCode(StatusCodes.Status422UnprocessableEntity, new ProblemDetails {
        Title  = "Unprocessable Entity",
        Detail = reason,
        Status = StatusCodes.Status422UnprocessableEntity
    });
```

Apply the same mapping in the generic `HandleDataUpdateResult<T>` overload.

---

### Task 3 — Add `TierOverrideDto`

Find the DTOs folder (mirror `SavedViewCreateDto.cs` for location and namespace), then add:

```csharp
namespace Kochava.Aim.Portal.Models.DTOs;

/// <summary>Request body for POST /{id}/tier-override.</summary>
public class TierOverrideDto {

    /// <summary>Requested override value: "aim_pro" or "aim_x".</summary>
    public string Override { get; set; }

}
```

---

### Task 4 — Extend `IOnboardingSessionService`

Add after the existing PUT method signature:

```csharp
/// <summary>
/// CSM-only. Sets ScopeOfWork.CsmApproved = true and appends a changeLog entry.
/// Returns AccessDenied if not CSM, NotFound if session missing.
/// </summary>
Task<DataUpdateResult> ApproveAsync(string id, string authorEmail);

/// <summary>
/// CSM-only asymmetric override.
/// Allowed: aim_pro → aim_x (always); aim_x → aim_pro only when Tier.Recommended == "aim_pro".
/// Appends a mandatory changeLog entry. Returns AccessDenied, NotFound, or UnprocessableEntity.
/// </summary>
Task<DataUpdateResult> TierOverrideAsync(string id, string overrideValue, string authorEmail);
```

---

### Task 5 — Implement `OnboardingSessionService`

**5.1** Read `OnboardingSessionService.cs`. Add `ApproveAsync`:

```csharp
public async Task<DataUpdateResult> ApproveAsync(string id, string authorEmail) {
    if (!_currentUser.IsAsAdmin) {
        return DataUpdateResult.AccessDenied;
    }

    var session = await _dataLayer.GetByIdAsync(id);
    if (session is null) {
        return DataUpdateResult.NotFound;
    }

    session.ScopeOfWork.CsmApproved = true;
    session.ChangeLog.Add(new ChangeLogEntry {
        At        = DateTimeOffset.UtcNow,
        UserEmail = authorEmail,      // server-stamped from session; never client-typed
        Type      = "csm_approved",
        Field     = "scopeOfWork.csmApproved",
        OldValue  = "false",
        NewValue  = "true"
    });
    session.UpdatedAt       = DateTimeOffset.UtcNow;
    session.UpdatedByUserId = _currentUser.UserId;

    await _dataLayer.ReplaceAsync(session);
    return DataUpdateResult.Success;
}
```

**5.2** Add `TierOverrideAsync`. Asymmetric rule from product-spec-v2.md §4.3 (Gary PR B10):
AIM Pro → AIM X is always allowed. AIM X → AIM Pro is only allowed when
`session.Tier.Recommended == "aim_pro"` — the algorithm already identified the client as Pro;
the CSM is activating that recommendation, not overriding it unilaterally.

```csharp
public async Task<DataUpdateResult> TierOverrideAsync(
    string id,
    string overrideValue,
    string authorEmail
) {
    if (!_currentUser.IsAsAdmin) {
        return DataUpdateResult.AccessDenied;
    }

    var session = await _dataLayer.GetByIdAsync(id);
    if (session is null) {
        return DataUpdateResult.NotFound;
    }

    // Enforce asymmetric override policy (spec §4.3, PR comment B10):
    //   aim_pro → aim_x  : always permitted (downgrade)
    //   aim_x   → aim_pro: only when auto-recommendation is already aim_pro
    var currentEffective = session.Tier.Override ?? session.Tier.Recommended;
    var isUpgrade        = currentEffective == "aim_x" && overrideValue == "aim_pro";
    if (isUpgrade && session.Tier.Recommended != "aim_pro") {
        return new DataUpdateResult(
            $"Tier upgrade to aim_pro is only permitted when the auto-recommendation " +
            $"is aim_pro; current auto-recommendation is {session.Tier.Recommended}.",
            DataUpdateErrorType.UnprocessableEntity
        );
    }

    var previousOverride = session.Tier.Override;
    session.Tier.Override   = overrideValue;
    session.Tier.Effective  = overrideValue;

    // Mandatory audit entry — every override writes to changeLog (spec §4.3, CG-004).
    session.ChangeLog.Add(new ChangeLogEntry {
        At        = DateTimeOffset.UtcNow,
        UserEmail = authorEmail,      // server-stamped from session; never client-typed
        Type      = "tier_override",
        Field     = "tier.override",
        OldValue  = previousOverride,
        NewValue  = overrideValue
    });
    session.UpdatedAt       = DateTimeOffset.UtcNow;
    session.UpdatedByUserId = _currentUser.UserId;

    await _dataLayer.ReplaceAsync(session);
    return DataUpdateResult.Success;
}
```

---

### Task 6 — Add controller actions to `OnboardingSessionsController`

Read `OnboardingSessionsController.cs`, then add:

```csharp
[HttpPost("{id}/approve")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status403Forbidden)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<ActionResult> Approve(string advertiserId, string id) =>
    HandleDataUpdateResult(
        await _service.ApproveAsync(id, CurrentUser.UserEmail)
    );

[HttpPost("{id}/tier-override")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status403Forbidden)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
[ProducesResponseType(StatusCodes.Status422UnprocessableEntity)]
public async Task<ActionResult> TierOverride(
    string advertiserId,
    string id,
    [FromBody] TierOverrideDto dto
) =>
    HandleDataUpdateResult(
        await _service.TierOverrideAsync(id, dto.Override, CurrentUser.UserEmail)
    );
```

The `advertiserId` parameter is bound by the `[Route]` template and enforced by
`AdvertiserContextActionFilter`; these actions do not need to read it directly.

---

### Task 7 — Run tests (expect pass)

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionsServiceCsm|OnboardingSessionsControllerCsm" \
  2>&1 | tail -30
```

All eleven service tests must pass:

- `Approve_NonAdmin_ReturnsAccessDenied_DataLayerNotCalled`
- `Approve_Admin_SetsCsmApproved_And_WritesChangeLogEntry`
- `Approve_SessionNotFound_ReturnsNotFound`
- `TierOverride_NonAdmin_ReturnsAccessDenied_DataLayerNotCalled`
- `TierOverride_XToProWhenAutoIsX_ReturnsUnprocessableEntity`
- `TierOverride_XToProWhenAutoIsPro_Succeeds_And_WritesAuditEntry`
- `TierOverride_ProToX_Succeeds_Always`
- `TierOverride_AuditEntry_UserEmail_IsServerStamped_FromSessionContext`
- `TierOverride_SessionNotFound_ReturnsNotFound`

All seven controller tests must pass:

- `Approve_ServiceReturnsSuccess_Returns204`
- `Approve_ServiceReturnsAccessDenied_Returns403`
- `Approve_ServiceReturnsNotFound_Returns404`
- `TierOverride_ServiceReturnsSuccess_Returns204`
- `TierOverride_ServiceReturnsAccessDenied_Returns403`
- `TierOverride_ServiceReturnsUnprocessableEntity_Returns422`
- `TierOverride_PassesSessionEmail_NotAnyClientValue`

Then confirm no regressions:

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

---

### Task 8 — Commit

```bash
git add \
  Kochava.Aim.Portal.Tests/OnboardingSessionsServiceCsmTests.cs \
  Kochava.Aim.Portal.Tests/OnboardingSessionsControllerCsmTests.cs \
  Kochava.Aim.Portal.Models/ActionResults/DataUpdateErrorType.cs \
  Kochava.Aim.Portal.Api/Areas/App/Controllers/AppControllerBase.cs \
  Kochava.Aim.Portal.Models/DTOs/TierOverrideDto.cs \
  Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs \
  Kochava.Aim.Portal.Services/OnboardingSessionService.cs \
  Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs

git commit -m "feat(onboarding): add CSM approve + tier-override endpoints with audit log (B6)"
```
