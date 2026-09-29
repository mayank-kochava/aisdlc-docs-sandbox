---
id: plan-b1
title: "B1 — OnboardingSession Model"
---

## Goal

Create the `OnboardingSession` POCO and all embedded types with BSON attributes, add the collection name constant to `CollectionNames.cs`, and verify with a passing NUnit test.

---

## Files

**Create**

- `Kochava.Aim.Portal.DataAccess/Mongo/Models/OnboardingSession.cs`
- `Kochava.Aim.Portal.Tests/OnboardingSessionModelTests.cs`

**Modify**

- `Kochava.Aim.Portal.DataAccess/Mongo/CollectionNames.cs`

---

## Dependencies

None.

---

## Tasks

### Task 1 — Write the failing test

**1.1** Create the test file:

```text
Kochava.Aim.Portal.Tests/OnboardingSessionModelTests.cs
```

with these contents:

```csharp
using System;
using System.Collections.Generic;
using FluentAssertions;
using Kochava.Aim.Portal.DataAccess.Mongo;
using Kochava.Aim.Portal.DataAccess.Mongo.Models;
using NUnit.Framework;

namespace Kochava.Aim.Portal.Tests;

[TestFixture]
public class OnboardingSessionModelTests {

    [Test]
    public void OnboardingSession_DefaultsAreCorrect() {
        var session = new OnboardingSession();

        session.Status.Should().Be("not_started");
        session.Wizard.Should().NotBeNull();
        session.Wizard.ProjectLeads.Should().NotBeNull().And.BeEmpty();
        session.Wizard.DataLeads.Should().NotBeNull().And.BeEmpty();
        session.Wizard.Platforms.Should().NotBeNull().And.BeEmpty();
        session.Wizard.DigitalMediaTypes.Should().NotBeNull().And.BeEmpty();
        session.Wizard.AttrGapCategories.Should().NotBeNull().And.BeEmpty();
        session.Wizard.PaidSplit.Should().NotBeNull();
        session.Wizard.Funnel.Should().NotBeNull();
        session.Wizard.Funnel.Items.Should().NotBeNull().And.BeEmpty();
        session.Wizard.WebAttrSources.Should().NotBeNull().And.BeEmpty();
        session.Wizard.AdSpendSources.Should().NotBeNull().And.BeEmpty();
        session.Wizard.ExternalFactors.Should().NotBeNull().And.BeEmpty();
        session.Tier.Should().NotBeNull();
        session.ScopeOfWork.Should().NotBeNull();
        session.ScopeOfWork.SectionNotes.Should().NotBeNull().And.BeEmpty();
        session.ScopeOfWork.CsmApproved.Should().BeFalse();
        session.Timeline.Should().NotBeNull();
        session.Timeline.Milestones.Should().NotBeNull().And.BeEmpty();
        session.ChangeLog.Should().NotBeNull().And.BeEmpty();
    }

    [Test]
    public void CollectionNames_AimOnboardingSessions_IsCorrect() {
        CollectionNames.AimOnboardingSessions.Should().Be("aim_onboarding_sessions");
    }

    [Test]
    public void OnboardingSession_CanRoundTripProperties() {
        var session = new OnboardingSession {
            AdvertiserId = "adv-123",
            Status = "in_progress",
            CreatedByUserId = "user-abc",
            UpdatedByUserId = "user-abc",
            CreatedAt = DateTimeOffset.UtcNow,
            UpdatedAt = DateTimeOffset.UtcNow,
        };
        session.Wizard.CompanyName = "Acme";
        session.Wizard.BudgetMonthly = 50000;
        session.Wizard.BudgetPeriod = "monthly";
        session.Wizard.ProjectLeads.Add(new WizardLead { Name = "Alice", Email = "alice@acme.com" });
        session.Tier.Recommended = "aim_pro";
        session.Tier.Effective = "aim_pro";
        session.ScopeOfWork.ApproverName = "Bob";
        session.Timeline.Milestones.Add(new Milestone {
            Key = "kickoff",
            Name = "Kickoff",
            Responsible = "CSM",
            WeekOffset = 0,
            Duration = "1w",
            Status = "not_started"
        });
        session.ChangeLog.Add(new ChangeLogEntry {
            At = DateTimeOffset.UtcNow,
            UserEmail = "csm@kochava.com",
            Type = "tier_override",
            Field = "tier.override",
            OldValue = null,
            NewValue = "aim_pro"
        });

        session.AdvertiserId.Should().Be("adv-123");
        session.Status.Should().Be("in_progress");
        session.Wizard.CompanyName.Should().Be("Acme");
        session.Wizard.BudgetMonthly.Should().Be(50000);
        session.Wizard.ProjectLeads.Should().HaveCount(1);
        session.Tier.Recommended.Should().Be("aim_pro");
        session.ScopeOfWork.ApproverName.Should().Be("Bob");
        session.Timeline.Milestones.Should().HaveCount(1);
        session.Timeline.Milestones[0].Key.Should().Be("kickoff");
        session.ChangeLog.Should().HaveCount(1);
        session.ChangeLog[0].UserEmail.Should().Be("csm@kochava.com");
    }
}
```

**1.2** Run the tests (expect compile failure — types don't exist yet):

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionModel" \
  --no-build 2>&1 | tail -20
```

---

### Task 2 — Add `AimOnboardingSessions` to `CollectionNames.cs`

**2.1** Open `Kochava.Aim.Portal.DataAccess/Mongo/CollectionNames.cs`. Read it, then append one line after `SavedViews`:

```csharp
    public const string AimOnboardingSessions = "aim_onboarding_sessions";
```

The file should look like this after the edit:

```csharp
namespace Kochava.Aim.Portal.DataAccess.Mongo;

public class CollectionNames {

    public const string Advertisers = "advertisers";

    public const string Apps = "apps";

    public const string AppsEnriched = "apps_enriched";

    public const string Metrics = "metrics";

    public const string Networks = "networks";

    public const string GeoLocations = "geo_locations";

    public const string CostCurves = "cost_curves";

    public const string UserInvitations = "user_invitations";

    public const string AppConfig = "app_config";

    public const string IncrementalReports = "incremental_reports";

    public const string SavedViews = "saved_views";

    public const string AimOnboardingSessions = "aim_onboarding_sessions";

}
```

---

### Task 3 — Create `OnboardingSession.cs`

**3.1** Create `Kochava.Aim.Portal.DataAccess/Mongo/Models/OnboardingSession.cs` with the full POCO and all embedded types.

Match the BSON attribute style from `SavedView.cs` (no class-level `[BsonNoId]`; `[BsonId]`/`[BsonRepresentation]`/`[BsonIgnoreIfDefault]` on the Id field; no other BSON attributes needed on plain properties; `= new()` initializers for embedded objects and collections).

Every field name matches the §5 JSON shape, in PascalCase:

```csharp
using System;
using System.Collections.Generic;
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

namespace Kochava.Aim.Portal.DataAccess.Mongo.Models;

public class OnboardingSession {

    [BsonId]
    [BsonRepresentation(BsonType.ObjectId)]
    [BsonIgnoreIfDefault]
    public string Id { get; set; }

    public string AdvertiserId { get; set; }

    /// <summary>not_started | in_progress | complete</summary>
    public string Status { get; set; } = "not_started";

    public DateTimeOffset CreatedAt { get; set; }

    public DateTimeOffset UpdatedAt { get; set; }

    public string CreatedByUserId { get; set; }

    public string UpdatedByUserId { get; set; }

    public Wizard Wizard { get; set; } = new();

    public Tier Tier { get; set; } = new();

    public ScopeOfWork ScopeOfWork { get; set; } = new();

    public TimelineState Timeline { get; set; } = new();

    public List<ChangeLogEntry> ChangeLog { get; set; } = new();

}

public class WizardLead {

    public string Name { get; set; }

    public string Email { get; set; }

}

public class WizardFunnel {

    public List<string> Items { get; set; } = new();

    public string Kpi { get; set; }

    public bool KpiConfirmed { get; set; }

    public Dictionary<string, string> Names { get; set; } = new();

}

public class WizardPaidSplit {

    public int Ios { get; set; } = 65;

    public int Android { get; set; } = 80;

    public int Web { get; set; } = 50;

}

public class Wizard {

    public string CompanyName { get; set; }

    public List<WizardLead> ProjectLeads { get; set; } = new();

    public List<WizardLead> DataLeads { get; set; } = new();

    public string AppName { get; set; }

    public List<string> Platforms { get; set; } = new();

    public Dictionary<string, double> PlatformShares { get; set; } = new();

    public string Modelling { get; set; }

    public string Region { get; set; }

    public string RegionOther { get; set; }

    public double BudgetMonthly { get; set; }

    public double BudgetAnnual { get; set; }

    /// <summary>monthly | annual</summary>
    public string BudgetPeriod { get; set; }

    public bool UsesOffline { get; set; }

    public int OfflineSplitPct { get; set; } = 20;

    public List<string> DigitalMediaTypes { get; set; } = new();

    public string HasAttrGaps { get; set; }

    public List<string> AttrGapCategories { get; set; } = new();

    public string CoverageConfidence { get; set; }

    public WizardPaidSplit PaidSplit { get; set; } = new();

    public bool Ua { get; set; }

    public string UaShare { get; set; }

    public bool Ue { get; set; }

    public string UeShare { get; set; }

    public bool Brand { get; set; }

    public string BrandShare { get; set; }

    public bool CampaignGrouping { get; set; }

    public string CampaignGroupingChoice { get; set; }

    public string Business { get; set; }

    public WizardFunnel Funnel { get; set; } = new();

    public Dictionary<string, object> WebFunnel { get; set; } = new();

    public bool WantsLtv { get; set; }

    public string LtvCohortAvail { get; set; }

    public string LtvPartialChoice { get; set; }

    public string Mmp { get; set; }

    public string MmpCollection { get; set; }

    public string MmpFileStorage { get; set; }

    public bool AppsflyerCohortAccess { get; set; }

    public List<string> WebAttrSources { get; set; } = new();

    public string SpendCollection { get; set; }

    public List<string> AdSpendSources { get; set; } = new();

    public Dictionary<string, object> History { get; set; } = new();

    public List<string> ExternalFactors { get; set; } = new();

    public string ExternalFactorsNotes { get; set; }

    public string ObjectivesGoal { get; set; }

    public string ObjectivesSuccess { get; set; }

    public string ObjectivesMarketing { get; set; }

    public string UaUeRoutingFlag { get; set; }

    public bool BrandRoutingFlag { get; set; }

}

public class Tier {

    /// <summary>aim_x | aim_pro</summary>
    public string Recommended { get; set; }

    /// <summary>CSM-only asymmetric override; null until set</summary>
    public string Override { get; set; }

    /// <summary>aim_x | aim_pro — resolved from Override ?? Recommended</summary>
    public string Effective { get; set; }

}

public class ScopeOfWork {

    public Dictionary<string, string> SectionNotes { get; set; } = new();

    public string ApproverName { get; set; }

    public string ApproverJobTitle { get; set; }

    public DateTimeOffset? ApprovedAt { get; set; }

    public bool CsmApproved { get; set; }

}

public class TimelineState {

    public DateTimeOffset? AnchorDate { get; set; }

    public List<Milestone> Milestones { get; set; } = new();

}

public class Milestone {

    public string Key { get; set; }

    public string Name { get; set; }

    public string Responsible { get; set; }

    public int WeekOffset { get; set; }

    public string Duration { get; set; }

    public string Status { get; set; }

    public DateTimeOffset? DelayedToDate { get; set; }

    public string Notes { get; set; }

}

public class ChangeLogEntry {

    public DateTimeOffset At { get; set; }

    public string UserEmail { get; set; }

    public string Type { get; set; }

    public string Field { get; set; }

    public string OldValue { get; set; }

    public string NewValue { get; set; }

}
```

---

### Task 4 — Run tests (expect pass)

```bash
cd /path/to/mmm-portal-api
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj \
  --filter "OnboardingSessionModel" 2>&1 | tail -20
```

All three tests (`DefaultsAreCorrect`, `CollectionNames_AimOnboardingSessions_IsCorrect`, `CanRoundTripProperties`) must pass.

Then run the full suite to confirm no regressions:

```bash
dotnet test Kochava.Aim.Portal.Tests/Kochava.Aim.Portal.Tests.csproj 2>&1 | tail -20
```

---

### Task 5 — Commit

```bash
git add \
  Kochava.Aim.Portal.DataAccess/Mongo/CollectionNames.cs \
  Kochava.Aim.Portal.DataAccess/Mongo/Models/OnboardingSession.cs \
  Kochava.Aim.Portal.Tests/OnboardingSessionModelTests.cs

git commit -m "feat(onboarding): add OnboardingSession POCO + CollectionNames entry (B1)"
```
