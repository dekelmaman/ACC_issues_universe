# Quick Create — Tablet Auto-Select After Review

## Behavior

When a user taps **Review** in Quick Create on a **tablet** (split master-detail layout), the newly created issue is automatically selected in the issues list and its detail pane is opened — without requiring any additional tap.

This matches the iOS iPad behavior (`IssuesListFeature.swift:867–868`).

## Scope

- Applies to **both** AI-enabled and AI-disabled Review flows (same code path).
- Does **not** apply when Quick Create is opened from Project Home (`fromProjectHome = true`), because the issues list pane is not visible in that context.
- Does **not** apply on phones (no split-master-detail layout).

## Android Implementation

Already implemented in `QuickCreateFragment.kt:135–157`:

```kotlin
onNavigateToIssueTab = { issueUid ->
    if (featureFlag.isSplitMasterDetailsEnabled &&
        isMasterDetailSplit &&
        issueUid != null &&
        issuesViewModel != null &&
        masterDetailViewModel != null &&
        !fromProjectHome) {
        issuesViewModel.onIssueClicked(issueUid = issueUid, selectIssueUid = true)
        masterDetailViewModel.openDetails(issueUid)
    }
    // navigate back (both branches)
    ...
}
```

### Event chain

```
ReviewTapped (QuickCreateActionCard:290)
  → Effect.FinalizeIssue(mode=REVIEW)       [QuickCreateUpdater:201]
  → Action.Internal.ReviewComplete           [QuickCreateProcessor:371]
  → Event.NavigateToIssueTab(issueUid)      [QuickCreateUpdater:403]
  → QuickCreateScreenHost:325               (dispatches to eventHandler)
  → QuickCreateFragment.onNavigateToIssueTab [Fragment:135]
```

### Guards explained

| Guard | Source | Reason |
|-------|--------|--------|
| `isSplitMasterDetailsEnabled` | `IssuesFeatureFlag` | Feature flag |
| `isMasterDetailSplit` | `R.bool.is_master_detail_split` (true on w≥768dp) | Device is tablet-width |
| `issueUid != null` | State | Issue was successfully created |
| `issuesViewModel != null && masterDetailViewModel != null` | Fragment injection | VMs must be available |
| `!fromProjectHome` | Fragment arg | Issues list not visible when opened from Project Home |

### What the two calls do

- `issuesViewModel.onIssueClicked(issueUid, selectIssueUid = true)` — emits `_selectedIssueUid` which drives the highlight on the list row
- `masterDetailViewModel.openDetails(issueUid)` — opens the detail pane with the issue
