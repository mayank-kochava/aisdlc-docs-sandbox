---
id: plan-b9
title: "B9 — Book-my-meeting Email"
---

## Goal

Add `POST /{id}/book-meeting` to `OnboardingSessionsController`.
The endpoint calls the existing `IEmailService.SendAsync` (AWS SES) with a new
`aim-onboarding-book-meeting` template, reads the recipient address from a new
`CsmTeamEmail` top-level config key, and returns a clear `200 OK` on success or
`500` on failure so the frontend can render sent/retry (design-consolidated.md §8).

> **OPEN ITEM — pending Gary:** recipient list and email copy are not yet
> confirmed. Until Gary signs off, use the `CsmTeamEmail` config key (single
> address placeholder `csm-team@kochava.com`) and the placeholder template bodies
> marked `<!-- TODO: replace with Gary-approved copy -->` below.
> Do **not** ship to production until this item is resolved.

---

## Files

**Create**

- `Kochava.Aim.Portal.Services/Resources/EmailTemplates/aim-onboarding-book-meeting.html`
- `Kochava.Aim.Portal.Services/Resources/EmailTemplates/aim-onboarding-book-meeting.txt`
- `Kochava.Aim.Portal.Tests/BookMeetingEmailTests.cs`

**Modify**

- `Kochava.Aim.Portal.Services/Kochava.Aim.Portal.Services.csproj` — add two
  `<EmbeddedResource>` entries
- `Kochava.Aim.Portal.Common/Helpers/Interfaces/IAppConfigHelper.cs` — add
  `string CsmTeamEmail { get; }`
- `Kochava.Aim.Portal.Common/Helpers/AppConfigHelper.cs` — read
  `_configuration["CsmTeamEmail"]`
- `Kochava.Aim.Portal.Api/appsettings.example.json` — add `"CsmTeamEmail"` key
- `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs` — add
  `Task<bool> SendBookMeetingEmailAsync(string sessionId)`
- `Kochava.Aim.Portal.Services/OnboardingSessionService.cs` — implement the
  method
- `Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs`
  — add the `[HttpPost("{id}/book-meeting")]` action

---

## Dependencies

**B4** — `OnboardingSessionsController`, `IOnboardingSessionService`, and
`IEmailService` DI wiring must exist before this plan.

---

## Tasks

### Task 1 — Write the failing tests

**1.1** Create `Kochava.Aim.Portal.Tests/BookMeetingEmailTests.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using FluentAssertions;
using Kochava.Aim.Portal.Api.Areas.App.Controllers;
using Kochava.Aim.Portal.Common.Identity.Interfaces;
using Kochava.Aim.Portal.Services.Interfaces;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Routing;
using Moq;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class BookMeetingEmailTests {

    private const string SessionId = "session-abc";
    private const string AdvertiserId = "adv-123";

    private Mock<IOnboardingSessionService> _serviceMock;
    private Mock<ICurrentUserContext> _currentUserMock;
    private OnboardingSessionsController _controller;

    [SetUp]
    public void Setup() {
        _serviceMock = new Mock<IOnboardingSessionService>();
        _currentUserMock = new Mock<ICurrentUserContext>();
        _currentUserMock.Setup(x => x.UserId).Returns("user-1");

        _controller = new OnboardingSessionsController(
            () => _currentUserMock.Object,
            _serviceMock.Object
        );

        _controller.ControllerContext = new ControllerContext {
            HttpContext = new DefaultHttpContext(),
            RouteData = new RouteData()
        };
    }

    [Test]
    public async Task BookMeeting_EmailSent_Returns200() {
        // Arrange
        _serviceMock
            .Setup(x => x.SendBookMeetingEmailAsync(AdvertiserId, SessionId))
            .ReturnsAsync(true);

        // Act
        var result = await _controller.BookMeeting(AdvertiserId, SessionId);

        // Assert
        var ok = result as OkResult;
        ok.Should().NotBeNull();
        ok!.StatusCode.Should().Be(StatusCodes.Status200OK);
        _serviceMock.Verify(x => x.SendBookMeetingEmailAsync(AdvertiserId, SessionId), Times.Once);
    }

    [Test]
    public async Task BookMeeting_EmailFails_Returns500() {
        // Arrange
        _serviceMock
            .Setup(x => x.SendBookMeetingEmailAsync(AdvertiserId, SessionId))
            .ReturnsAsync(false);

        // Act
        var result = await _controller.BookMeeting(AdvertiserId, SessionId);

        // Assert
        var err = result as ObjectResult;
        err.Should().NotBeNull();
        err!.StatusCode.Should().Be(StatusCodes.Status500InternalServerError);
    }

    [Test]
    public async Task BookMeeting_ServiceThrows_Returns500() {
        // Arrange
        _serviceMock
            .Setup(x => x.SendBookMeetingEmailAsync(AdvertiserId, SessionId))
            .ThrowsAsync(new Exception("SES unavailable"));

        // Act
        var result = await _controller.BookMeeting(AdvertiserId, SessionId);

        // Assert
        var err = result as ObjectResult;
        err.Should().NotBeNull();
        err!.StatusCode.Should().Be(StatusCodes.Status500InternalServerError);
    }

}
```

**1.2** Run (expect compile failure — types don't exist yet):

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "BookMeetingEmail" \
  --no-build 2>&1 | tail -20
```

---

### Task 2 — Add `CsmTeamEmail` to config

**2.1** Read `Kochava.Aim.Portal.Common/Helpers/Interfaces/IAppConfigHelper.cs`.
Append after `string OpenAiApiKey { get; }`:

```csharp
    /// <summary>Email address(es) for the CSM team — recipient for book-meeting notifications.
    /// OPEN ITEM: final value pending Gary.</summary>
    string CsmTeamEmail { get; }
```

**2.2** Read `Kochava.Aim.Portal.Common/Helpers/AppConfigHelper.cs`.
In the constructor, after the `OpenAiApiKey` line, add:

```csharp
        CsmTeamEmail = _configuration["CsmTeamEmail"];
```

In the properties section, after `public string OpenAiApiKey { get; }`, add:

```csharp
    /// <summary>Email address(es) for the CSM team — recipient for book-meeting notifications.
    /// OPEN ITEM: final value pending Gary.</summary>
    public string CsmTeamEmail { get; }
```

**2.3** Read `Kochava.Aim.Portal.Api/appsettings.example.json`.
Add after the `"SupportEmail"` line:

```json
  "CsmTeamEmail": "csm-team@kochava.com",
```

---

### Task 3 — Add placeholder email templates

**3.1** Create
`Kochava.Aim.Portal.Services/Resources/EmailTemplates/aim-onboarding-book-meeting.html`:

```html
<!-- TODO: replace with Gary-approved copy before production -->
<!doctype html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <meta content="text/html; charset=UTF-8" http-equiv="Content-Type">
</head>
<body style="background:#f8f8f8;margin:0;padding:0;">
<div style="max-width:640px;margin:0 auto;background:#fff;padding:40px 50px;font-family:'Source Sans Pro',sans-serif;font-size:15px;color:#333;line-height:24px;">
  <h2 style="font-size:20px;font-weight:500;">New onboarding meeting request</h2>
  <p>A customer has requested a meeting to discuss their AIM Onboarding session.</p>
  <p><strong>Advertiser ID:</strong> {ADVERTISER_ID}</p>
  <p><strong>Session ID:</strong> {SESSION_ID}</p>
  <p><strong>Requested at:</strong> {REQUESTED_AT}</p>
  <hr style="border:none;border-top:1px solid #dcddde;margin:30px 0;">
  <p style="font-size:13px;color:#747f8d;">Sent by {COMPANY_NAME} &bull; <a href="{COMPANY_WEBSITE_URL}" style="color:#e77a35;">{COMPANY_WEBSITE_DOMAIN}</a></p>
</div>
</body>
</html>
```

**3.2** Create
`Kochava.Aim.Portal.Services/Resources/EmailTemplates/aim-onboarding-book-meeting.txt`:

```text
<!-- TODO: replace with Gary-approved copy before production -->
New onboarding meeting request

A customer has requested a meeting to discuss their AIM Onboarding session.

Advertiser ID: {ADVERTISER_ID}
Session ID: {SESSION_ID}
Requested at: {REQUESTED_AT}

---------------------------------------------------------------------

Sent by {COMPANY_NAME}
{COMPANY_WEBSITE_DOMAIN}
```

**3.3** Read
`Kochava.Aim.Portal.Services/Kochava.Aim.Portal.Services.csproj`.
Add two `<EmbeddedResource>` entries directly below the existing
`invitation.txt` entry:

```xml
    <EmbeddedResource Include="Resources\EmailTemplates\aim-onboarding-book-meeting.html"/>
    <EmbeddedResource Include="Resources\EmailTemplates\aim-onboarding-book-meeting.txt"/>
```

---

### Task 4 — Extend `IOnboardingSessionService`

**4.1** Read `Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs`.
Add the method signature:

```csharp
    /// <summary>Sends a book-meeting notification email to the CSM team via SES.
    /// Returns true on success, false on failure (never throws).</summary>
    Task<bool> SendBookMeetingEmailAsync(string advertiserId, string sessionId);
```

---

### Task 5 — Implement `SendBookMeetingEmailAsync` in the service

**5.1** Read `Kochava.Aim.Portal.Services/OnboardingSessionService.cs`.

Inject `IEmailService` and `IAppConfigHelper` via the constructor if not already
present (mirror the same constructor-injection pattern used throughout the
SavedView stack).

Add the method:

```csharp
public async Task<bool> SendBookMeetingEmailAsync(string advertiserId, string sessionId) {
    try {
        var recipient = _config.CsmTeamEmail;
        var tokens = new Dictionary<string, (string Value, string HtmlValue)> {
            { "ADVERTISER_ID", (advertiserId, System.Web.HttpUtility.HtmlEncode(advertiserId)) },
            { "SESSION_ID",    (sessionId,    System.Web.HttpUtility.HtmlEncode(sessionId)) },
            { "REQUESTED_AT",  (DateTimeOffset.UtcNow.ToString("u"), System.Web.HttpUtility.HtmlEncode(DateTimeOffset.UtcNow.ToString("u"))) }
        };
        await _emailService.SendAsync(
            "aim-onboarding-book-meeting",
            "AIM Onboarding: meeting request",
            tokens,
            recipient);
        return true;
    } catch {
        return false;
    }
}
```

> `_config` is the `IAppConfigHelper` field; `_emailService` is the
> `IEmailService` field — both injected via constructor, matching the
> `OnboardingSessionService` constructor established in B3/B4.

---

### Task 6 — Add the controller action

**6.1** Read
`Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs`.

Add after the existing `PUT /{id}` action:

```csharp
[HttpPost("{id}/book-meeting")]
[ProducesResponseType(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status500InternalServerError)]
public async Task<ActionResult> BookMeeting(string advertiserId, string id) {
    try {
        var sent = await _onboardingSessionService.SendBookMeetingEmailAsync(advertiserId, id);
        return sent
            ? Ok()
            : StatusCode(StatusCodes.Status500InternalServerError, new ProblemDetails {
                Title = "Email delivery failed",
                Detail = "The book-meeting notification could not be sent. Please try again.",
                Status = StatusCodes.Status500InternalServerError
            });
    } catch (Exception ex) {
        return StatusCode(StatusCodes.Status500InternalServerError, new ProblemDetails {
            Title = "Internal Server Error",
            Detail = ex.Message,
            Status = StatusCodes.Status500InternalServerError
        });
    }
}
```

The route resolves to
`POST /mmm/advertisers/{advertiserId}/onboarding-sessions/{id}/book-meeting`
(inherits the `[Route("[area]/advertisers/{advertiserId}/[controller]")]`
class-level attribute from B4, matching §6 of design-consolidated.md).

---

### Task 7 — Run failing tests, then pass

**7.1** Build to confirm compile errors are gone:

```bash
dotnet build Kochava.Aim.Portal.Api/Kochava.Aim.Portal.Api.csproj 2>&1 | tail -20
```

**7.2** Run the new tests (expect pass):

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "BookMeetingEmail" 2>&1 | tail -20
```

All three tests must pass: `BookMeeting_EmailSent_Returns200`,
`BookMeeting_EmailFails_Returns500`, `BookMeeting_ServiceThrows_Returns500`.

**7.3** Run the full suite to confirm no regressions:

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

---

### Task 8 — Commit

```bash
git add \
  Kochava.Aim.Portal.Services/Resources/EmailTemplates/aim-onboarding-book-meeting.html \
  Kochava.Aim.Portal.Services/Resources/EmailTemplates/aim-onboarding-book-meeting.txt \
  Kochava.Aim.Portal.Services/Kochava.Aim.Portal.Services.csproj \
  Kochava.Aim.Portal.Common/Helpers/Interfaces/IAppConfigHelper.cs \
  Kochava.Aim.Portal.Common/Helpers/AppConfigHelper.cs \
  Kochava.Aim.Portal.Api/appsettings.example.json \
  Kochava.Aim.Portal.Services/Interfaces/IOnboardingSessionService.cs \
  Kochava.Aim.Portal.Services/OnboardingSessionService.cs \
  Kochava.Aim.Portal.Api/Areas/App/Controllers/OnboardingSessionsController.cs \
  Kochava.Aim.Portal.Tests/BookMeetingEmailTests.cs

git commit -m "feat(onboarding): add POST /book-meeting SES email endpoint (B9)"
```
