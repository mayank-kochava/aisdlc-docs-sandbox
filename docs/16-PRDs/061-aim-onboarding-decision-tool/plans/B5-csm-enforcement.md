---
id: plan-b5
title: "B5 — CSM Server-Side Enforcement"
---

## Goal

Resolve the caller's `is_as_admin` flag from mos-iam server-side (cached), surface it on
`ICurrentUserContext`, and expose a service-layer gate that returns
`DataUpdateResult.AccessDenied` (→ 403 via the existing `HandleDataUpdateResult`/`ForbidWithReason`
path) for non-admin callers on CSM-only actions. No controller code is added here; B6 (`approve`,
`tier-override`) consumes the gate.

> **OPEN ITEM — confirm with Satish before wiring:**
> The implementation below assumes an `is_as_admin` boolean returned from a mos-iam
> per-user lookup keyed by `userId`. The exact signal (`is_as_admin` vs `ROLE_CSM` keto
> role) and the mos-iam endpoint path are **not yet confirmed**. See eng-research Q2 and
> design-consolidated §12. The seam is `MosIamAdminStatusResolver.ResolveIsAsAdminAsync` —
> only that method changes when the endpoint/field is confirmed. Everything else stays.

---

## Files

**Create**

- `Kochava.Aim.Portal.Services/Interfaces/IAdminStatusResolver.cs`
- `Kochava.Aim.Portal.Services/MosIamAdminStatusResolver.cs`
- `Kochava.Aim.Portal.Tests/CsmEnforcementTests.cs`

**Modify**

- `Kochava.Aim.Portal.Common/Identity/Interfaces/ICurrentUserContext.cs` — add `bool IsAsAdmin`
- `Kochava.Aim.Portal.Common/Identity/CurrentUserContext.cs` — implement `IsAsAdmin`
- `Kochava.Aim.Portal.Api/ActionFilters/AdvertiserContextActionFilter.cs` — inject resolver, populate `IsAsAdmin`
- `Kochava.Aim.Portal.Common/Helpers/Interfaces/IAppConfigHelper.cs` — add `MosIamBaseUrl`, `AdminStatusCacheTtlSeconds`
- `Kochava.Aim.Portal.Common/Helpers/AppConfigHelper.cs` — read new config keys
- `Kochava.Aim.Portal.Api/Startup.cs` — add `services.AddMemoryCache()`, register named `"MosIam"` HttpClient
- `Kochava.Aim.Portal.Api/appsettings.json` — add `MosIam` config block

---

## Dependencies

B3 (DTOs + `OnboardingSessionService`) must exist so the gate can be wired into the service.
`ICurrentUserContext` and `AppControllerBase` are already in place.

---

## Tasks

### Task 1 — Write failing tests

**1.1** Create `Kochava.Aim.Portal.Tests/CsmEnforcementTests.cs`:

```csharp
using System;
using System.Threading.Tasks;
using FluentAssertions;
using Kochava.Aim.Portal.Common.Identity.Interfaces;
using Kochava.Aim.Portal.Models.ActionResults;
using Kochava.Aim.Portal.Services;
using Kochava.Aim.Portal.Services.Interfaces;
using Moq;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class CsmEnforcementTests {

    private Mock<ICurrentUserContext> _userContext;
    private OnboardingSessionService _service;

    [SetUp]
    public void SetUp() {
        _userContext = new Mock<ICurrentUserContext>();

        // Wire a minimal OnboardingSessionService with the user-context factory.
        // Pass null for the data layer — gate tests must short-circuit before touching it.
        _service = new OnboardingSessionService(
            onboardingSessionDataLayer: null,
            currentUser: () => _userContext.Object
        );
    }

    [Test]
    public async Task RequireCsmAsync_WhenIsAsAdminTrue_ReturnsSuccess() {
        _userContext.Setup(u => u.IsAsAdmin).Returns(true);

        var result = await _service.RequireCsmAsync();

        result.Successful.Should().BeTrue();
    }

    [Test]
    public async Task RequireCsmAsync_WhenIsAsAdminFalse_ReturnsAccessDenied() {
        _userContext.Setup(u => u.IsAsAdmin).Returns(false);

        var result = await _service.RequireCsmAsync();

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.AccessDenied);
    }

    [Test]
    public async Task RequireCsmAsync_WhenUserContextNull_ReturnsAccessDenied() {
        // Defensive: factory returns null (no HttpContext).
        var svc = new OnboardingSessionService(
            onboardingSessionDataLayer: null,
            currentUser: () => null
        );

        var result = await svc.RequireCsmAsync();

        result.Successful.Should().BeFalse();
        result.ErrorType.Should().Be(DataUpdateErrorType.AccessDenied);
    }

    [Test]
    public async Task MosIamAdminStatusResolver_EmptyUserId_ReturnsFalseWithoutCallingHttp() {
        // Guard: never call mos-iam for an empty userId.
        var resolver = new MosIamAdminStatusResolver(
            httpClientFactory: null,   // must not be touched
            cache: null,               // must not be touched
            config: null               // must not be touched
        );

        var result = await resolver.ResolveIsAsAdminAsync(string.Empty);

        result.Should().BeFalse();
    }
}
```

**1.2** Run (expect compile failure — types do not exist yet):

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "CsmEnforcement" \
  --no-build 2>&1 | tail -20
```

---

### Task 2 — Add `IsAsAdmin` to `ICurrentUserContext` and `CurrentUserContext`

**2.1** Open
`Kochava.Aim.Portal.Common/Identity/Interfaces/ICurrentUserContext.cs`
and add one property after `ActiveAdvertiser`:

```csharp
    /// <summary>
    /// True when mos-iam confirms the caller is a super-admin (CSM).
    /// Populated by AdvertiserContextActionFilter via IAdminStatusResolver.
    /// </summary>
    bool IsAsAdmin { get; set; }
```

**2.2** Open
`Kochava.Aim.Portal.Common/Identity/CurrentUserContext.cs`
and add the backing property after `ActiveAdvertiser`:

```csharp
    public bool IsAsAdmin { get; set; }
```

---

### Task 3 — Create `IAdminStatusResolver` and `MosIamAdminStatusResolver`

**3.1** Create
`Kochava.Aim.Portal.Services/Interfaces/IAdminStatusResolver.cs`:

```csharp
namespace Kochava.Aim.Portal.Services.Interfaces;

/// <summary>
/// Resolves whether a user is a super-admin (CSM) from mos-iam.
/// Results are cached per userId to avoid a per-request hop.
/// </summary>
/// <remarks>
/// OPEN ITEM: confirm the mos-iam endpoint and response field
/// (is_as_admin vs ROLE_CSM) with Satish before deploying.
/// The seam is ResolveIsAsAdminAsync — swap only the impl, not the callers.
/// </remarks>
public interface IAdminStatusResolver : ISingleInstanceService {

    /// <summary>
    /// Returns true if <paramref name="userId"/> is a super-admin.
    /// Returns false for empty or null userId without calling mos-iam.
    /// </summary>
    Task<bool> ResolveIsAsAdminAsync(string userId);

}
```

**3.2** Create
`Kochava.Aim.Portal.Services/MosIamAdminStatusResolver.cs`:

```csharp
using System;
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;
using Kochava.Aim.Portal.Common.Helpers.Interfaces;
using Kochava.Aim.Portal.Services.Interfaces;
using Microsoft.Extensions.Caching.Memory;

namespace Kochava.Aim.Portal.Services;

/// <summary>
/// Calls mos-iam to resolve is_as_admin for a given userId.
///
/// *** OPEN ITEM — confirm with Satish before shipping: ***
///   - Endpoint:  assumed GET {MosIamBaseUrl}/users/{userId}/my-details
///   - Field:     assumed JSON path $.is_as_admin (bool)
///   - Signal:    is_as_admin (frontend-parity) vs ROLE_CSM keto role — not yet verified
///   - Auth:      assumed service-account header or mTLS (pending confirmation)
///
/// Change ONLY this file when the endpoint/field is confirmed.
/// </summary>
public class MosIamAdminStatusResolver : IAdminStatusResolver {

    // ── seam ─────────────────────────────────────────────────────────────
    // Replace these three constants once Satish confirms the real values.
    private const string EndpointTemplate = "/users/{0}/my-details"; // TODO: confirm path
    private const string AdminField = "is_as_admin";                  // TODO: confirm field
    private const string CacheKeyPrefix = "mos_iam_is_admin_";
    // ─────────────────────────────────────────────────────────────────────

    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IMemoryCache _cache;
    private readonly IAppConfigHelper _config;

    public MosIamAdminStatusResolver(
        IHttpClientFactory httpClientFactory,
        IMemoryCache cache,
        IAppConfigHelper config
    ) {
        _httpClientFactory = httpClientFactory;
        _cache = cache;
        _config = config;
    }

    public async Task<bool> ResolveIsAsAdminAsync(string userId) {
        if (string.IsNullOrWhiteSpace(userId)) { return false; }

        var cacheKey = $"{CacheKeyPrefix}{userId}";
        if (_cache != null && _cache.TryGetValue(cacheKey, out bool cached)) {
            return cached;
        }

        var result = await FetchFromMosIamAsync(userId);

        if (_cache != null && _config != null) {
            var ttl = TimeSpan.FromSeconds(_config.AdminStatusCacheTtlSeconds);
            _cache.Set(cacheKey, result, ttl);
        }

        return result;
    }

    // ── seam: replace the body below when endpoint/auth are confirmed ────
    private async Task<bool> FetchFromMosIamAsync(string userId) {
        try {
            var client = _httpClientFactory.CreateClient("MosIam");
            var path = string.Format(EndpointTemplate, Uri.EscapeDataString(userId));
            var response = await client.GetAsync(path);
            if (!response.IsSuccessStatusCode) { return false; }

            var json = await response.Content.ReadAsStringAsync();
            using var doc = JsonDocument.Parse(json);
            if (doc.RootElement.TryGetProperty(AdminField, out var prop)
                && prop.ValueKind == JsonValueKind.True) {
                return true;
            }
        } catch {
            // Fail closed: if mos-iam is unreachable, deny admin.
        }
        return false;
    }
    // ─────────────────────────────────────────────────────────────────────

}
```

---

### Task 4 — Add config keys to `IAppConfigHelper` and `AppConfigHelper`

**4.1** Open
`Kochava.Aim.Portal.Common/Helpers/Interfaces/IAppConfigHelper.cs`
and add after `ZenDeskPassword`:

```csharp
    /// <summary>Base URL for mos-iam API (e.g. https://mos-iam.internal). Confirm with Satish.</summary>
    string MosIamBaseUrl { get; }

    /// <summary>TTL in seconds for the per-user admin-status cache (default 300).</summary>
    int AdminStatusCacheTtlSeconds { get; }
```

**4.2** Open
`Kochava.Aim.Portal.Common/Helpers/AppConfigHelper.cs`
and add two lines in the constructor after the ZenDesk block:

```csharp
        MosIamBaseUrl = _configuration["MosIam:BaseUrl"];
        AdminStatusCacheTtlSeconds = int.TryParse(_configuration["MosIam:AdminStatusCacheTtlSeconds"], out var ttl) ? ttl : 300;
```

And add the backing properties after `ZenDeskPassword`:

```csharp
    public string MosIamBaseUrl { get; }

    public int AdminStatusCacheTtlSeconds { get; }
```

---

### Task 5 — Update `AdvertiserContextActionFilter`

Inject `IAdminStatusResolver` and populate `IsAsAdmin` after `UserId` is set.
The call is conditional: skip if `UserId` is empty (guard per eng-research).

Open
`Kochava.Aim.Portal.Api/ActionFilters/AdvertiserContextActionFilter.cs`
and apply these changes:

**Add to using block:**

```csharp
using Kochava.Aim.Portal.Services.Interfaces;
```

**New constructor parameter** (add after `_advertisersService`):

```csharp
    private readonly IAdminStatusResolver _adminStatusResolver;

    public AdvertiserContextActionFilter(
        Func<ICurrentUserContext> currentUserFactory,
        IAdvertisersService advertisersService,
        IAdminStatusResolver adminStatusResolver
    ) {
        _currentUserFactory = currentUserFactory;
        _advertisersService = advertisersService;
        _adminStatusResolver = adminStatusResolver;
    }
```

**Add after the `x-user-name` block inside `OnActionExecutionAsync`**, before `await next()`:

```csharp
        // Resolve CSM (super-admin) status from mos-iam — cached per userId.
        // OPEN ITEM: confirm is_as_admin signal + endpoint with Satish (see MosIamAdminStatusResolver).
        if (currentUser != null && !string.IsNullOrEmpty(currentUser.UserId)) {
            currentUser.IsAsAdmin = await _adminStatusResolver.ResolveIsAsAdminAsync(currentUser.UserId);
        }
```

---

### Task 6 — Add `RequireCsmAsync` gate to `OnboardingSessionService`

Open
`Kochava.Aim.Portal.Services/OnboardingSessionService.cs`
(created in B3) and add one public method:

```csharp
    /// <summary>
    /// Returns AccessDenied when the caller is not a CSM super-admin.
    /// Call this at the top of every CSM-only action (approve, tier-override).
    /// </summary>
    public Task<DataUpdateResult> RequireCsmAsync() {
        var user = _currentUser();
        if (user == null || !user.IsAsAdmin) {
            return Task.FromResult(DataUpdateResult.AccessDenied);
        }
        return Task.FromResult(DataUpdateResult.Success);
    }
```

---

### Task 7 — Wire DI and config

**7.1** Open `Kochava.Aim.Portal.Api/Startup.cs`.

In `ConfigureServices`, add **before** `services.AddControllersWithViews`:

```csharp
        services.AddMemoryCache();

        services.AddHttpClient("MosIam", c => {
            // TODO: confirm auth header (service-account token or mTLS) with Satish.
            var mosIamBaseUrl = Configuration["MosIam:BaseUrl"];
            if (!string.IsNullOrEmpty(mosIamBaseUrl)) {
                c.BaseAddress = new Uri(mosIamBaseUrl);
            }
        });
```

`IAdminStatusResolver` auto-registers as a singleton because
`IAdminStatusResolver : ISingleInstanceService` and `Services.IocModule` scans the
assembly for `ISingleInstanceService` implementations. No manual registration needed.

**7.2** Open `Kochava.Aim.Portal.Api/appsettings.json` and add a new top-level key:

```json
"MosIam": {
  "BaseUrl": "",
  "AdminStatusCacheTtlSeconds": "300"
}
```

The real URL will be filled per environment (local/qa/prod k8s overlays); leave blank for now.

---

### Task 8 — Run tests (expect pass)

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "CsmEnforcement" 2>&1 | tail -20
```

All four tests must pass:

- `RequireCsmAsync_WhenIsAsAdminTrue_ReturnsSuccess`
- `RequireCsmAsync_WhenIsAsAdminFalse_ReturnsAccessDenied`
- `RequireCsmAsync_WhenUserContextNull_ReturnsAccessDenied`
- `MosIamAdminStatusResolver_EmptyUserId_ReturnsFalseWithoutCallingHttp`

Then run the full suite:

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

---

### Task 9 — Commit

```bash
git add \
  Kochava.Aim.Portal.Common/Identity/Interfaces/ICurrentUserContext.cs \
  Kochava.Aim.Portal.Common/Identity/CurrentUserContext.cs \
  Kochava.Aim.Portal.Common/Helpers/Interfaces/IAppConfigHelper.cs \
  Kochava.Aim.Portal.Common/Helpers/AppConfigHelper.cs \
  Kochava.Aim.Portal.Services/Interfaces/IAdminStatusResolver.cs \
  Kochava.Aim.Portal.Services/MosIamAdminStatusResolver.cs \
  Kochava.Aim.Portal.Services/OnboardingSessionService.cs \
  Kochava.Aim.Portal.Api/ActionFilters/AdvertiserContextActionFilter.cs \
  Kochava.Aim.Portal.Api/Startup.cs \
  Kochava.Aim.Portal.Api/appsettings.json \
  Kochava.Aim.Portal.Tests/CsmEnforcementTests.cs

git commit -m "feat(onboarding): add CSM server-side enforcement via mos-iam is_as_admin lookup (B5)"
```
