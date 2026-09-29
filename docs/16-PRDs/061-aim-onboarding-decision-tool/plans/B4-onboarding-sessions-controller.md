---
id: plan-b4
title: "B4 — OnboardingSessionsController"
---

## Goal

Add `OnboardingSessionsController` under
`/mmm/advertisers/{advertiserId}/onboarding-sessions` with three endpoints:

- `GET /` — return the latest session for the advertiser (portal reads `status`
  from this response to decide wizard vs read-only record; `onboarding_status` is
  **derived, not stored**)
- `POST /` — create a new session
- `PUT /{id}` — autosave full-doc replace; returns **409** once `ApprovedAt` is set
  (approval is terminal — answers are immutable after that point)

Mirror `SavedViewsController` exactly: `[Area("mmm")]`,
`[Route("[area]/advertisers/{advertiserId}/[controller]")]`, `AppControllerBase`,
`HandleDataUpdateResult`, `ForbidWithReason`.

---

## Files

**Create**

- `Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs`
- `Kochava.Aim.Portal.Tests/OnboardingSessionsControllerTests.cs`

**Dependencies already in place (from B3)**

- `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs`
- `Kochava.Aim.Portal.Services/OnboardingSessionService.cs`
- `Kochava.Aim.Portal.Models/DTOs/OnboardingSessionDto.cs`
- `Kochava.Aim.Portal.Models/DTOs/OnboardingSessionCreateDto.cs`
- `Kochava.Aim.Portal.Models/DTOs/OnboardingSessionUpdateDto.cs`

---

## Dependencies

**B3** — DTOs + `OnboardingSessionService` (create / replace, terminal-approval
guard) must be complete and green before starting this plan.

---

## Tasks

### Task 1 — Write the failing controller tests

**1.1** Create
`Kochava.Aim.Portal.Tests/OnboardingSessionsControllerTests.cs`:

```csharp
using System;
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
public class OnboardingSessionsControllerTests {

    private const string AdvertiserId = "adv-001";
    private const string SessionId    = "session-abc";
    private const string UserId       = "user-xyz";

    private Mock<IOnboardingSessionService> _serviceMock;
    private Mock<ICurrentUserContext>       _currentUserMock;
    private OnboardingSessionsController   _controller;

    [SetUp]
    public void Setup() {
        _serviceMock     = new Mock<IOnboardingSessionService>();
        _currentUserMock = new Mock<ICurrentUserContext>();

        _currentUserMock.Setup(x => x.UserId).Returns(UserId);

        _controller = new OnboardingSessionsController(
            () => _currentUserMock.Object,
            _serviceMock.Object
        );

        _controller.ControllerContext = new ControllerContext {
            HttpContext = new DefaultHttpContext(),
            RouteData   = new RouteData()
        };
    }

    // -------------------------------------------------------------------------
    // GET /  (latest for advertiser)
    // -------------------------------------------------------------------------

    [Test]
    public async Task GetLatest_WhenSessionExists_Returns200WithDto() {
        // Arrange
        var dto = new OnboardingSessionDto { Id = SessionId, Status = "in_progress" };
        _serviceMock.Setup(x => x.GetLatestByAdvertiserAsync(AdvertiserId))
                    .ReturnsAsync(dto);

        // Act
        var result = await _controller.GetLatest(AdvertiserId);

        // Assert
        var ok = result.Result as OkObjectResult;
        ok.Should().NotBeNull();
        ok!.StatusCode.Should().Be(StatusCodes.Status200OK);
        ok.Value.Should().BeEquivalentTo(dto);
        _serviceMock.Verify(x => x.GetLatestByAdvertiserAsync(AdvertiserId), Times.Once);
    }

    [Test]
    public async Task GetLatest_WhenNoSession_Returns200WithNull() {
        // Arrange — no session yet; portal treats absence as not_started
        _serviceMock.Setup(x => x.GetLatestByAdvertiserAsync(AdvertiserId))
                    .ReturnsAsync((OnboardingSessionDto)null);

        // Act
        var result = await _controller.GetLatest(AdvertiserId);

        // Assert
        var ok = result.Result as OkObjectResult;
        ok.Should().NotBeNull();
        ok!.StatusCode.Should().Be(StatusCodes.Status200OK);
        ok.Value.Should().BeNull();
    }

    // -------------------------------------------------------------------------
    // POST /  (create)
    // -------------------------------------------------------------------------

    [Test]
    public async Task Create_WithValidPayload_Returns201WithDto() {
        // Arrange
        var createDto  = new OnboardingSessionCreateDto { AdvertiserId = AdvertiserId };
        var createdDto = new OnboardingSessionDto { Id = SessionId, Status = "not_started" };
        _serviceMock.Setup(x => x.CreateAsync(AdvertiserId, createDto))
                    .ReturnsAsync(DataUpdateResult<OnboardingSessionDto>.FromValue(createdDto));

        // Act
        var result = await _controller.Create(AdvertiserId, createDto);

        // Assert
        var created = result.Result as ObjectResult;
        created.Should().NotBeNull();
        created!.StatusCode.Should().Be(StatusCodes.Status201Created);
        created.Value.Should().BeEquivalentTo(createdDto);
        _serviceMock.Verify(x => x.CreateAsync(AdvertiserId, createDto), Times.Once);
    }

    [Test]
    public async Task Create_WhenAlreadyApproved_Returns409() {
        // Arrange — service rejects because an approved session exists
        _serviceMock.Setup(x => x.CreateAsync(AdvertiserId, It.IsAny<OnboardingSessionCreateDto>()))
                    .ReturnsAsync(new DataUpdateResult<OnboardingSessionDto>(
                        "An approved session already exists for this advertiser",
                        DataUpdateErrorType.Conflict));

        // Act
        var result = await _controller.Create(AdvertiserId, new OnboardingSessionCreateDto());

        // Assert
        var conflict = result.Result as ObjectResult;
        conflict.Should().NotBeNull();
        conflict!.StatusCode.Should().Be(StatusCodes.Status409Conflict);
    }

    // -------------------------------------------------------------------------
    // PUT /{id}  (autosave full-doc replace)
    // -------------------------------------------------------------------------

    [Test]
    public async Task Autosave_WithValidPayload_Returns204() {
        // Arrange
        var updateDto = new OnboardingSessionUpdateDto { Status = "in_progress" };
        _serviceMock.Setup(x => x.ReplaceAsync(AdvertiserId, SessionId, updateDto))
                    .ReturnsAsync(DataUpdateResult.Success);

        // Act
        var result = await _controller.Autosave(AdvertiserId, SessionId, updateDto);

        // Assert
        result.Should().BeOfType<NoContentResult>();
        _serviceMock.Verify(x => x.ReplaceAsync(AdvertiserId, SessionId, updateDto), Times.Once);
    }

    [Test]
    public async Task Autosave_WhenApproved_Returns409() {
        // Arrange — approval is terminal; answers are immutable
        _serviceMock.Setup(x => x.ReplaceAsync(AdvertiserId, SessionId, It.IsAny<OnboardingSessionUpdateDto>()))
                    .ReturnsAsync(new DataUpdateResult(
                        "Session is approved and cannot be modified",
                        DataUpdateErrorType.Conflict));

        // Act
        var result = await _controller.Autosave(AdvertiserId, SessionId, new OnboardingSessionUpdateDto());

        // Assert
        var conflict = result as ObjectResult;
        conflict.Should().NotBeNull();
        conflict!.StatusCode.Should().Be(StatusCodes.Status409Conflict);
    }

    [Test]
    public async Task Autosave_WhenSessionNotFound_Returns404() {
        // Arrange
        _serviceMock.Setup(x => x.ReplaceAsync(AdvertiserId, SessionId, It.IsAny<OnboardingSessionUpdateDto>()))
                    .ReturnsAsync(DataUpdateResult.NotFound);

        // Act
        var result = await _controller.Autosave(AdvertiserId, SessionId, new OnboardingSessionUpdateDto());

        // Assert
        var notFound = result as ObjectResult;
        notFound.Should().NotBeNull();
        notFound!.StatusCode.Should().Be(StatusCodes.Status404NotFound);
    }

    [Test]
    public async Task Autosave_WhenAccessDenied_Returns403() {
        // Arrange
        _serviceMock.Setup(x => x.ReplaceAsync(AdvertiserId, SessionId, It.IsAny<OnboardingSessionUpdateDto>()))
                    .ReturnsAsync(DataUpdateResult.AccessDenied);

        // Act
        var result = await _controller.Autosave(AdvertiserId, SessionId, new OnboardingSessionUpdateDto());

        // Assert
        var forbidden = result as ObjectResult;
        forbidden.Should().NotBeNull();
        forbidden!.StatusCode.Should().Be(StatusCodes.Status403Forbidden);
    }

}
```

**1.2** Run the targeted tests (expect compile failure — controller does not exist
yet):

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionsController" \
  --no-build 2>&1 | tail -20
```

---

### Task 2 — Verify `DataUpdateErrorType.Conflict` exists

**2.1** Open
`Kochava.Aim.Portal.Models/ActionResults/DataUpdateErrorType.cs` and confirm a
`Conflict` variant is present. If it is missing, add it:

```csharp
Conflict,
```

The enum must include at least:

```csharp
public enum DataUpdateErrorType {
    NotFound,
    AccessDenied,
    NameConflict,
    Conflict,
    InvalidModel,
}
```

**2.2** Add a `DataUpdateResult.Conflict` static singleton in
`DataUpdateResult.cs` alongside the existing statics (only if absent):

```csharp
public static readonly DataUpdateResult Conflict =
    new("The request conflicts with the current state of the resource",
        DataUpdateErrorType.Conflict);
```

And add the `Conflict` branch to `HandleDataUpdateResult` in
`AppControllerBase.cs` (only if absent):

```csharp
DataUpdateErrorType.Conflict => StatusCode(StatusCodes.Status409Conflict,
    new ProblemDetails {
        Title  = "Conflict",
        Detail = result.ErrorMessage,
        Status = StatusCodes.Status409Conflict
    }),
```

---

### Task 3 — Create `OnboardingSessionsController.cs`

**3.1** Create
`Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Kochava.Aim.Portal.Common.Identity.Interfaces;
using Kochava.Aim.Portal.Models.ActionResults;
using Kochava.Aim.Portal.Models.DTOs;
using Kochava.Aim.Portal.Models.Exceptions;
using Kochava.Aim.Portal.Models.ProblemDetails;
using Kochava.Aim.Portal.Services.Interfaces;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;

namespace Kochava.Aim.Portal.Api.Areas.App.Controllers;

[Area("mmm")]
[ApiController]
[Route("[area]/advertisers/{advertiserId}/[controller]")]
[Produces("application/json")]
[ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
[ProducesResponseType(typeof(ExceptionProblemDetails), StatusCodes.Status500InternalServerError)]
public class OnboardingSessionsController : AppControllerBase {

    private readonly IOnboardingSessionService _service;

    public OnboardingSessionsController(
        Func<ICurrentUserContext> currentUser,
        IOnboardingSessionService service
    ) : base(currentUser) {
        _service = service;
    }

    /// <summary>
    /// Returns the latest onboarding session for the advertiser, or null if none exists.
    /// The portal reads session.status to determine wizard vs read-only record:
    ///   null            → not_started (show wizard, "Start")
    ///   approvedAt null → in_progress (show wizard, "Continue")
    ///   approvedAt set  → complete    (show read-only record, "View your onboarding record")
    /// onboarding_status is derived from this response — it is NOT a stored field.
    /// </summary>
    [HttpGet]
    [ProducesResponseType(typeof(OnboardingSessionDto), StatusCodes.Status200OK)]
    public async Task<ActionResult<OnboardingSessionDto>> GetLatest(string advertiserId) {
        var session = await _service.GetLatestByAdvertiserAsync(advertiserId);
        return Ok(session);
    }

    /// <summary>
    /// Creates a new onboarding session for the advertiser.
    /// Returns 409 if an approved session already exists (approval is terminal).
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(OnboardingSessionDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<ActionResult<OnboardingSessionDto>> Create(
        string advertiserId,
        [FromBody] OnboardingSessionCreateDto dto
    ) {
        var result = await _service.CreateAsync(advertiserId, dto);
        if (!result.Successful) {
            return result.ErrorType == DataUpdateErrorType.Conflict
                ? StatusCode(StatusCodes.Status409Conflict, new ProblemDetails {
                    Title  = "Conflict",
                    Detail = result.ErrorMessage,
                    Status = StatusCodes.Status409Conflict
                })
                : BadRequest(new ProblemDetails {
                    Title  = "Bad Request",
                    Detail = result.ErrorMessage,
                    Status = StatusCodes.Status400BadRequest
                });
        }
        return StatusCode(StatusCodes.Status201Created, result.Value);
    }

    /// <summary>
    /// Full-document autosave replace for wizard answers.
    /// Returns 409 once ApprovedAt is set — approval is terminal and answers are immutable.
    /// </summary>
    [HttpPut("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<ActionResult> Autosave(
        string advertiserId,
        string id,
        [FromBody] OnboardingSessionUpdateDto dto
    ) {
        var result = await _service.ReplaceAsync(advertiserId, id, dto);
        if (!result.Successful && result.ErrorType == DataUpdateErrorType.Conflict) {
            return StatusCode(StatusCodes.Status409Conflict, new ProblemDetails {
                Title  = "Conflict",
                Detail = result.ErrorMessage,
                Status = StatusCodes.Status409Conflict
            });
        }
        return HandleDataUpdateResult(result);
    }

}
```

---

### Task 4 — Run tests (expect pass)

**4.1** Run only the new controller tests:

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionsController" 2>&1 | tail -20
```

All seven tests must be green:

- `GetLatest_WhenSessionExists_Returns200WithDto`
- `GetLatest_WhenNoSession_Returns200WithNull`
- `Create_WithValidPayload_Returns201WithDto`
- `Create_WhenAlreadyApproved_Returns409`
- `Autosave_WithValidPayload_Returns204`
- `Autosave_WhenApproved_Returns409`
- `Autosave_WhenSessionNotFound_Returns404`
- `Autosave_WhenAccessDenied_Returns403`

**4.2** Run the full suite to confirm no regressions:

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

---

### Task 5 — Commit

```bash
git add \
  Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs \
  Kochava.Aim.Portal.Models/ActionResults/DataUpdateResult.cs \
  Kochava.Aim.Portal.Models/ActionResults/DataUpdateErrorType.cs \
  Kochava.Aim.Portal.Api/Areas/App/Controllers/AppControllerBase.cs \
  Kochava.Aim.Portal.Tests/OnboardingSessionsControllerTests.cs

git commit -m "feat(onboarding): add OnboardingSessionsController GET/POST/PUT (B4)"
```

> Only include files that were actually modified — omit unchanged files from
> `git add`.
