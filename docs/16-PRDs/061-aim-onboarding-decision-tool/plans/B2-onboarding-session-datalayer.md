---
id: plan-b2
title: "B2 — OnboardingSessionDataLayer"
---

## Goal

Add `OnboardingSessionDataLayer` and `IOnboardingSessionDataLayer` to `mmm-portal-api`,
mirroring the `SavedViewsDataLayer` stack exactly. The layer is advertiser-scoped
(`advertiserId` passed as a method parameter), Autofac-registered automatically via the
assembly scan in `DataAccess/IocModule.cs`, and enforces the terminal-approval contract:
`ReplaceAsync` reads the **stored** document first and returns `DataUpdateResult.Locked`
when the persisted `ScopeOfWork.ApprovedAt` is set — never trusting the incoming payload
for lock state (§8a: "audit author is server-stamped, never client-typed" extends to lock
state). This signals the controller to respond 409. A new `Locked` variant is added to
`DataUpdateErrorType` so `AppControllerBase.HandleDataUpdateResult` can map it.
`AppendChangeLogAsync` appends one entry and stamps `UpdatedAt` server-side without
replacing the whole document. No `DeleteAsync` is implemented — the §6 API contract exposes
no DELETE endpoint for onboarding sessions.

---

## Files

**Create**

- `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs`
- `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs`
- `Kochava.Aim.Portal.Tests/OnboardingSessionDataLayerTests.cs`

**Modify**

- `Kochava.Aim.Portal.Models/ActionResults/DataUpdateErrorType.cs` — add `Locked`
- `Kochava.Aim.Portal.Models/ActionResults/DataUpdateResult.cs` — add static `Locked`

---

## Dependencies

- **B1** — `OnboardingSession`, `ChangeLogEntry`, and `CollectionNames.AimOnboardingSessions`
  must exist before any task here will compile.

---

## Tasks

### Task 1 — Write the failing tests

**1.1** Create the test file. No data layer tests exist in the project yet. The approach is to
subclass `OnboardingSessionDataLayer` in the test assembly, injecting a Moq
`IMongoCollection<OnboardingSession>` via a `protected` constructor added to the data layer.
`MongoDataLayerBase`'s constructor is bypassed by passing a valid dummy connection string
(`"mongodb://localhost"`) and a mock `IAppConfigHelper`; `MongoClient` is lazy so no real
connection opens, and the overridden `Collection()` method returns the injected mock before
the real client is ever touched.

Create `Kochava.Aim.Portal.Tests/OnboardingSessionDataLayerTests.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using FluentAssertions;
using Kochava.Aim.Portal.Common.Helpers.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.DataLayers;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using Kochava.Aim.Portal.Models.ActionResults;
using Microsoft.Extensions.Logging;
using MongoDB.Driver;
using Moq;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class OnboardingSessionDataLayerTests {

    private Mock<IMongoCollection<OnboardingSession>> _collection;
    private TestableOnboardingSessionDataLayer _layer;

    [SetUp]
    public void SetUp() {
        _collection = new Mock<IMongoCollection<OnboardingSession>>();
        _layer = new TestableOnboardingSessionDataLayer(_collection.Object);
    }

    // ── GetLatestByAdvertiserAsync ────────────────────────────────────────────

    [Test]
    public async Task GetLatestByAdvertiserAsync_ReturnsNullWhenNoDocument() {
        var cursor = BuildCursor(new List<OnboardingSession>());
        _collection
            .Setup(c => c.FindAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<FindOptions<OnboardingSession, OnboardingSession>>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(cursor);

        var result = await _layer.GetLatestByAdvertiserAsync("adv-1");

        result.Should().BeNull();
    }

    [Test]
    public async Task GetLatestByAdvertiserAsync_ReturnsDocumentWhenFound() {
        var session = new OnboardingSession { AdvertiserId = "adv-1", Status = "in_progress" };
        var cursor = BuildCursor(new List<OnboardingSession> { session });
        _collection
            .Setup(c => c.FindAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<FindOptions<OnboardingSession, OnboardingSession>>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(cursor);

        var result = await _layer.GetLatestByAdvertiserAsync("adv-1");

        result.Should().NotBeNull();
        result!.AdvertiserId.Should().Be("adv-1");
    }

    // ── CreateAsync ───────────────────────────────────────────────────────────

    [Test]
    public async Task CreateAsync_InsertsDocumentAndReturnsIt() {
        var session = new OnboardingSession { AdvertiserId = "adv-2" };
        _collection
            .Setup(c => c.InsertOneAsync(
                session,
                It.IsAny<InsertOneOptions>(),
                It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        var result = await _layer.CreateAsync(session);

        result.Should().BeSameAs(session);
        _collection.Verify(
            c => c.InsertOneAsync(session, It.IsAny<InsertOneOptions>(), It.IsAny<CancellationToken>()),
            Times.Once);
    }

    // ── ReplaceAsync — unlocked ───────────────────────────────────────────────

    [Test]
    public async Task ReplaceAsync_ReturnsSuccess_WhenStoredDocIsNotApproved() {
        // FindAsync returns the stored doc with no ApprovedAt
        var stored = new OnboardingSession {
            Id = "6655443322110011aabbccdd",
            AdvertiserId = "adv-3",
            ScopeOfWork = new ScopeOfWork { ApprovedAt = null }
        };
        _collection
            .Setup(c => c.FindAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<FindOptions<OnboardingSession, OnboardingSession>>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(BuildCursor(new List<OnboardingSession> { stored }));

        var replaceResult = new Mock<ReplaceOneResult>();
        replaceResult.Setup(r => r.MatchedCount).Returns(1);
        _collection
            .Setup(c => c.ReplaceOneAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<OnboardingSession>(),
                It.IsAny<ReplaceOptions>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(replaceResult.Object);

        var incoming = new OnboardingSession {
            Id = "6655443322110011aabbccdd",
            AdvertiserId = "adv-3"
        };
        var result = await _layer.ReplaceAsync("adv-3", incoming);

        result.Successful.Should().BeTrue();
    }

    [Test]
    public async Task ReplaceAsync_ReturnsNotFound_WhenNoStoredDoc() {
        // FindAsync returns empty — document does not exist
        _collection
            .Setup(c => c.FindAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<FindOptions<OnboardingSession, OnboardingSession>>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(BuildCursor(new List<OnboardingSession>()));

        var incoming = new OnboardingSession {
            Id = "6655443322110011aabbccdd",
            AdvertiserId = "adv-3"
        };
        var result = await _layer.ReplaceAsync("adv-3", incoming);

        result.Should().Be(DataUpdateResult.NotFound);
        _collection.Verify(
            c => c.ReplaceOneAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<OnboardingSession>(),
                It.IsAny<ReplaceOptions>(),
                It.IsAny<CancellationToken>()),
            Times.Never);
    }

    // ── ReplaceAsync — locked (stored ApprovedAt set) ─────────────────────────

    [Test]
    public async Task ReplaceAsync_ReturnsLocked_WhenStoredApprovedAtIsSet() {
        // The stored doc is approved — lock must be read from DB, not the incoming payload
        var stored = new OnboardingSession {
            Id = "6655443322110011aabbccdd",
            AdvertiserId = "adv-4",
            ScopeOfWork = new ScopeOfWork { ApprovedAt = DateTimeOffset.UtcNow }
        };
        _collection
            .Setup(c => c.FindAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<FindOptions<OnboardingSession, OnboardingSession>>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(BuildCursor(new List<OnboardingSession> { stored }));

        // Incoming payload has no ApprovedAt (stale client buffer) — must still be rejected
        var incoming = new OnboardingSession {
            Id = "6655443322110011aabbccdd",
            AdvertiserId = "adv-4",
            ScopeOfWork = new ScopeOfWork { ApprovedAt = null }
        };
        var result = await _layer.ReplaceAsync("adv-4", incoming);

        result.Should().Be(DataUpdateResult.Locked);
        _collection.Verify(
            c => c.ReplaceOneAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<OnboardingSession>(),
                It.IsAny<ReplaceOptions>(),
                It.IsAny<CancellationToken>()),
            Times.Never);
    }

    // ── AppendChangeLogAsync ──────────────────────────────────────────────────

    [Test]
    public async Task AppendChangeLogAsync_ReturnsSuccess_WhenEntryPushed() {
        var entry = new ChangeLogEntry {
            At = DateTimeOffset.UtcNow,
            UserEmail = "csm@kochava.com",
            Type = "tier_override",
            Field = "tier.override",
            OldValue = null,
            NewValue = "aim_pro"
        };
        var updateResult = new Mock<UpdateResult>();
        updateResult.Setup(r => r.MatchedCount).Returns(1);
        _collection
            .Setup(c => c.UpdateOneAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<UpdateDefinition<OnboardingSession>>(),
                It.IsAny<UpdateOptions>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(updateResult.Object);

        var result = await _layer.AppendChangeLogAsync("adv-5", "6655443322110011aabbccdd", entry);

        result.Successful.Should().BeTrue();
        _collection.Verify(
            c => c.UpdateOneAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<UpdateDefinition<OnboardingSession>>(),
                It.IsAny<UpdateOptions>(),
                It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Test]
    public async Task AppendChangeLogAsync_ReturnsNotFound_WhenMatchedCountIsZero() {
        var entry = new ChangeLogEntry { UserEmail = "u@k.com", Type = "t" };
        var updateResult = new Mock<UpdateResult>();
        updateResult.Setup(r => r.MatchedCount).Returns(0);
        _collection
            .Setup(c => c.UpdateOneAsync(
                It.IsAny<FilterDefinition<OnboardingSession>>(),
                It.IsAny<UpdateDefinition<OnboardingSession>>(),
                It.IsAny<UpdateOptions>(),
                It.IsAny<CancellationToken>()))
            .ReturnsAsync(updateResult.Object);

        var result = await _layer.AppendChangeLogAsync("adv-5", "6655443322110011aabbccdd", entry);

        result.Should().Be(DataUpdateResult.NotFound);
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private static IAsyncCursor<OnboardingSession> BuildCursor(List<OnboardingSession> items) {
        var cursor = new Mock<IAsyncCursor<OnboardingSession>>();
        var called = false;
        cursor
            .Setup(c => c.MoveNextAsync(It.IsAny<CancellationToken>()))
            .ReturnsAsync(() => {
                if (called) { return false; }
                called = true;
                return true;
            });
        cursor.Setup(c => c.Current).Returns(items);
        return cursor.Object;
    }

}

/// <summary>
/// Bypasses MongoDataLayerBase's real MongoClient constructor for unit tests
/// by accepting a pre-built IMongoCollection directly. Calls the protected
/// constructor on OnboardingSessionDataLayer which passes a dummy connection
/// string and a stub config so MongoClient construction succeeds without a
/// real server (the overridden Collection() method is called instead).
/// </summary>
internal class TestableOnboardingSessionDataLayer : OnboardingSessionDataLayer {

    private readonly IMongoCollection<OnboardingSession> _collection;

    public TestableOnboardingSessionDataLayer(IMongoCollection<OnboardingSession> collection)
        : base(collection) {
        _collection = collection;
    }

    protected override IMongoCollection<OnboardingSession> Collection() => _collection;

}
```

**1.2** Run the tests — expect compile failure because the types do not exist yet:

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionDataLayer" \
  --no-build 2>&1 | tail -30
```

---

### Task 2 — Add `Locked` to `DataUpdateErrorType` and `DataUpdateResult`

**2.1** Open `Kochava.Aim.Portal.Models/ActionResults/DataUpdateErrorType.cs` and add
`Locked` after `UpdateInProgress`:

```csharp
namespace Kochava.Aim.Portal.Models.ActionResults;

public enum DataUpdateErrorType {

    None,

    NotFound,

    AccessDenied,

    NameConflict,

    InvalidModel,

    UpdateInProgress,

    Locked,

}
```

**2.2** Open `Kochava.Aim.Portal.Models/ActionResults/DataUpdateResult.cs` and add the static
sentinel after `AccessDenied` in both `DataUpdateResult` and `DataUpdateResult<T>`:

In `DataUpdateResult` (non-generic), after `public static readonly DataUpdateResult AccessDenied`:

```csharp
    public static readonly DataUpdateResult Locked = new("This session has been approved and its answers are locked", DataUpdateErrorType.Locked);
```

In `DataUpdateResult<T>` (generic), after `public new static readonly DataUpdateResult<T> AccessDenied`:

```csharp
    public new static readonly DataUpdateResult<T> Locked = new("This session has been approved and its answers are locked", DataUpdateErrorType.Locked);
```

---

### Task 3 — Create `IOnboardingSessionDataLayer`

Create `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs`:

```csharp
using System.Threading.Tasks;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using Kochava.Aim.Portal.Models.ActionResults;

namespace Kochava.Aim.Portal.DataAccess.Mongo.DataLayers.Interfaces;

public interface IOnboardingSessionDataLayer : IMongoDataLayer {

    Task<OnboardingSession> GetLatestByAdvertiserAsync(string advertiserId);

    Task<OnboardingSession> CreateAsync(OnboardingSession session);

    /// <summary>
    /// Full-document replace. Returns <see cref="DataUpdateResult.Locked"/> when
    /// <see cref="ScopeOfWork.ApprovedAt"/> is already set (answers are terminal after
    /// approval — §8a). Returns <see cref="DataUpdateResult.NotFound"/> when no document
    /// matches the advertiser+id filter. Returns <see cref="DataUpdateResult.Success"/>
    /// on a clean replace.
    /// </summary>
    Task<DataUpdateResult> ReplaceAsync(string advertiserId, OnboardingSession session);

    /// <summary>
    /// Atomically pushes one entry to <c>ChangeLog[]</c> and stamps <c>UpdatedAt</c>
    /// server-side. Author (userEmail) must already be set on <paramref name="entry"/>
    /// by the caller (service layer) before this call.
    /// </summary>
    Task<DataUpdateResult> AppendChangeLogAsync(string advertiserId, string id, ChangeLogEntry entry);

}
```

---

### Task 4 — Create `OnboardingSessionDataLayer`

Create `Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Kochava.Aim.Portal.Common.Helpers.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.DataLayers.Interfaces;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using Kochava.Aim.Portal.Models.ActionResults;
using Microsoft.Extensions.Logging;
using MongoDB.Driver;
using Moq;

namespace Kochava.Aim.Portal.DataAccess.Mongo.DataLayers;

public class OnboardingSessionDataLayer : MongoDataLayerBase, IOnboardingSessionDataLayer {

    private readonly IAppConfigHelper _config;

    public OnboardingSessionDataLayer(
        IAppConfigHelper config,
        ILogger logger
    ) : base(config.MongoConnectionString, config, logger) {
        _config = config;
    }

    /// <summary>
    /// Constructor for test subclasses only. Passes a valid dummy connection string so
    /// MongoDataLayerBase construction succeeds without a real server. The overridden
    /// <see cref="Collection"/> method is always called instead of the real client.
    /// </summary>
    protected OnboardingSessionDataLayer(IMongoCollection<OnboardingSession> _)
        : base("mongodb://localhost", StubConfig(), Mock.Of<ILogger>()) { }

    public async Task<OnboardingSession> GetLatestByAdvertiserAsync(string advertiserId) {
        var filter = AdvertiserFilter(advertiserId);
        var cursor = await Collection()
            .FindAsync(filter, new FindOptions<OnboardingSession> {
                Sort = Builders<OnboardingSession>.Sort.Descending(x => x.UpdatedAt),
                Limit = 1
            });
        return await cursor.FirstOrDefaultAsync();
    }

    public async Task<OnboardingSession> CreateAsync(OnboardingSession session) {
        await Collection().InsertOneAsync(session);
        return session;
    }

    /// <summary>
    /// Full-document replace. Reads the stored document first and returns
    /// <see cref="DataUpdateResult.Locked"/> if the stored <c>ScopeOfWork.ApprovedAt</c>
    /// is set — the lock is always derived from the persisted state, never from the
    /// incoming payload (§8a).
    /// </summary>
    public async Task<DataUpdateResult> ReplaceAsync(string advertiserId, OnboardingSession session) {
        var filter = AdvertiserFilter(advertiserId)
            & Builders<OnboardingSession>.Filter.Eq(x => x.Id, session.Id);

        var existingCursor = await Collection().FindAsync(filter, new FindOptions<OnboardingSession> { Limit = 1 });
        var existing = await existingCursor.FirstOrDefaultAsync();
        if (existing == null) { return DataUpdateResult.NotFound; }
        if (existing.ScopeOfWork?.ApprovedAt.HasValue == true) { return DataUpdateResult.Locked; }

        session.UpdatedAt = DateTimeOffset.UtcNow;
        var result = await Collection().ReplaceOneAsync(filter, session);
        return result.MatchedCount >= 1 ? DataUpdateResult.Success : DataUpdateResult.NotFound;
    }

    public async Task<DataUpdateResult> AppendChangeLogAsync(
        string advertiserId,
        string id,
        ChangeLogEntry entry) {

        var filter = AdvertiserFilter(advertiserId)
            & Builders<OnboardingSession>.Filter.Eq(x => x.Id, id);
        var update = Builders<OnboardingSession>.Update
            .Push(x => x.ChangeLog, entry)
            .Set(x => x.UpdatedAt, DateTimeOffset.UtcNow);

        var result = await Collection().UpdateOneAsync(filter, update);
        return result.MatchedCount >= 1 ? DataUpdateResult.Success : DataUpdateResult.NotFound;
    }

    protected virtual IMongoCollection<OnboardingSession> Collection() =>
        GetCollection<OnboardingSession>(_config.MongoDatabaseName, CollectionNames.AimOnboardingSessions);

    private static FilterDefinition<OnboardingSession> AdvertiserFilter(string advertiserId) =>
        Builders<OnboardingSession>.Filter.Eq(x => x.AdvertiserId, advertiserId);

    private static IAppConfigHelper StubConfig() {
        var m = new Mock<IAppConfigHelper>();
        m.Setup(c => c.MongoConnectionString).Returns("mongodb://localhost");
        m.Setup(c => c.MongoSslCAFilePath).Returns((string)null);
        m.Setup(c => c.LogMongoQueries).Returns(false);
        return m.Object;
    }

}
```

> **Scoping note:** `SavedViewsDataLayer.GetUserScopedFilter()` scopes by both
> `AdvertiserId` and `UserId` because saved views are per-user. Onboarding sessions are
> one-per-advertiser (§5: "one document per `advertiserId`"), so the filter uses only
> `AdvertiserId`. The `advertiserId` is passed as a method parameter rather than pulled from
> `ICurrentUserContext` because it comes from the controller route parameter
> `{advertiserId}` (§6 API contract).
>
> **`Moq` in production assembly:** `StubConfig()` uses `Moq` which is a test dependency.
> Before shipping, replace `StubConfig()` with a dedicated `StubAppConfigHelper` inner
> class that implements `IAppConfigHelper` with only the two fields `MongoDataLayerBase`
> reads during construction (`MongoConnectionString`, `MongoSslCAFilePath`,
> `LogMongoQueries`). Alternatively, extract a protected virtual `BuildClient()` method
> on `MongoDataLayerBase` and override it to return a no-op client in tests. Either
> refactor is equivalent — the current form is the simplest to follow in this plan.

---

### Task 5 — Build and run the failing tests

```bash
dotnet build Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

Expect compilation to succeed now. Run the targeted tests:

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionDataLayer" 2>&1 | tail -30
```

All 8 tests must pass:

- `GetLatestByAdvertiserAsync_ReturnsNullWhenNoDocument`
- `GetLatestByAdvertiserAsync_ReturnsDocumentWhenFound`
- `CreateAsync_InsertsDocumentAndReturnsIt`
- `ReplaceAsync_ReturnsSuccess_WhenStoredDocIsNotApproved`
- `ReplaceAsync_ReturnsNotFound_WhenNoStoredDoc`
- `ReplaceAsync_ReturnsLocked_WhenStoredApprovedAtIsSet`
- `AppendChangeLogAsync_ReturnsSuccess_WhenEntryPushed`
- `AppendChangeLogAsync_ReturnsNotFound_WhenMatchedCountIsZero`

If `GetLatestByAdvertiserAsync_ReturnsNullWhenNoDocument` fails because
`FirstOrDefaultAsync` does not invoke `FindAsync` with the expected args, check the cursor
mock setup — the `MoveNextAsync` sequence must return `true` once then `false`.

---

### Task 6 — Run full test suite (no regressions)

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

All pre-existing tests must continue to pass.

---

### Task 7 — Commit

```bash
git add \
  Kochava.Aim.Portal.Models/ActionResults/DataUpdateErrorType.cs \
  Kochava.Aim.Portal.Models/ActionResults/DataUpdateResult.cs \
  Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/Interfaces/IOnboardingSessionDataLayer.cs \
  Kochava.Aim.Portal.DataAccess/Mongo/DataLayers/OnboardingSessionDataLayer.cs \
  Kochava.Aim.Portal.Tests/OnboardingSessionDataLayerTests.cs

git commit -m "feat(onboarding): add OnboardingSessionDataLayer + Locked result type (B2)"
```

---

## Autofac registration

No manual registration is required. `DataAccess/IocModule.cs` scans the executing assembly
for all types assignable to `IMongoDataLayer` and registers them
`.AsImplementedInterfaces().AsSelf().SingleInstance()`. Because
`IOnboardingSessionDataLayer : IMongoDataLayer` (inherited via the interface chain), the
assembly scan picks up `OnboardingSessionDataLayer` automatically.

---

## Notes for downstream plans

- **B3 (OnboardingSessionService):** inject `IOnboardingSessionDataLayer`; call
  `ReplaceAsync` and branch on `DataUpdateResult.Locked` to surface the lock to the
  controller.
- **B4 (controller):** extend `AppControllerBase.HandleDataUpdateResult` to map
  `DataUpdateErrorType.Locked → StatusCode(409)`. The static `DataUpdateResult.Locked`
  added in Task 2 is the sentinel the controller switch should match.
- The `protected` constructor on `OnboardingSessionDataLayer` (Task 4) is only for test
  subclasses. It currently imports `Moq` into the production assembly — replace with a
  lightweight `StubAppConfigHelper` inner class before the PR is merged (see the note in
  Task 4).
