# iOS Worktree Bootstrap

## What This Is

A reusable bootstrap procedure that makes a fresh git worktree build-ready for the `build-mobile-clean-dev` iOS project. Run this **immediately after `git worktree add`** — before any Xcode or build work.

## Why This Is Needed

`make ios-build-ready` requires `ARTIFACTORY_USER`/`ARTIFACTORY_PASSWORD` environment variables that are not available in agent/CI context. Additionally, `art-toucan.autodesk.com` (the Autodesk SPM registry) requires VPN access — packages must be copied from an existing DerivedData cache, not resolved from the network.

## When to Use

- Any agent or workflow that creates a git worktree for iOS work
- Before running `xcodebuild`, opening a workspace in Xcode, or importing Swift modules
- When SourceKit shows `No such module 'AlloyUI'` or similar errors in a worktree

## The Bootstrap Script

Run this block immediately after `git worktree add`, replacing `<worktree-path>` and `<main-repo>`:

```bash
MAIN=<main-repo>          # e.g. /Users/mamand/Development/build-mobile-clean-dev
WT=<worktree-path>        # e.g. /Users/mamand/Development/build-mobile-clean-dev/.claude/worktrees/SCCOM-XXXXX

# 1. Bundler config (Artifactory token for gem installs)
cp "$MAIN/.bundle/config" "$WT/.bundle/config"

# 2. CocoaPods — copy from main repo to avoid re-running pod install
cp -R "$MAIN/ios/Pods" "$WT/ios/Pods"
cp "$WT/ios/Podfile.lock" "$WT/ios/Pods/Manifest.lock"

# 3. Bootstrap markers — signal that setup is complete, skip credential checks
touch "$WT/vendor"
touch "$WT/ios/.ios_bootstrapped"

# 4. Gemfile — main repo has newer sorbet-static (Apple Silicon compatible)
#    Committed Gemfile.lock has old sorbet-static 0.4.4755 that doesn't support arm64
mkdir -p "$WT/common/util"
cp "$MAIN/common/util/configure_pg_gems.rb" "$WT/common/util/"
cp "$MAIN/Gemfile" "$MAIN/Gemfile.lock" "$WT/"

# 5. SPM SourcePackages — copy from Xcode DerivedData cache (VPN required for fresh resolve)
SRC_SP=$(ls -td ~/Library/Developer/Xcode/DerivedData/PlanGrid-*/SourcePackages 2>/dev/null | head -1)
WT_SP="$WT/.derivedData/SourcePackages"
mkdir -p "$WT_SP"
for dir in checkouts artifacts prebuilts registry repositories; do
  [ -d "$SRC_SP/$dir" ] && cp -R "$SRC_SP/$dir" "$WT_SP/"
done
# Patch workspace-state.json so Xcode resolves packages from the worktree-local path
sed "s|$SRC_SP|$WT_SP|g" "$SRC_SP/workspace-state.json" > "$WT_SP/workspace-state.json"

# 6. Local package symlinks
# Xcode workspace references ios/<PackageName> but packages live at ios/Modules/<PackageName>
# Without these, Xcode errors with "The folder X doesn't exist"
IOS="$WT/ios"
for pkg in AlloyResources AlloyUI CorePlatform Filters FormsUI LMVViewer Localizer Markups MediaFlowKit MemberPicker ShareKit TranslationKit testing-extras; do
  [ ! -e "$IOS/$pkg" ] && [ -d "$IOS/Modules/$pkg" ] && ln -s "Modules/$pkg" "$IOS/$pkg"
done

echo "Bootstrap complete. Open $WT/ios/PlanGrid.xcworkspace in Xcode."
```

## What Each Step Does

| Step | Purpose |
|------|---------|
| `.bundle/config` | Bundler token for gem installs in the worktree |
| `ios/Pods` | Avoids re-running `pod install` — uses main repo's resolved pods |
| `Pods/Manifest.lock` | Keeps CocoaPods from complaining the Podfile.lock is out of sync |
| `vendor` + `.ios_bootstrapped` | Marker files that signal to `make` that bootstrap already ran |
| `Gemfile`/`Gemfile.lock` | Main repo has newer sorbet-static (0.5.x); committed version fails on Apple Silicon |
| `SourcePackages` | Copies resolved SPM packages — avoids VPN dependency for registry packages |
| `workspace-state.json` | Xcode uses this to locate package checkouts — must point to worktree-local path |
| Symlinks | `ios/<Pkg>` → `ios/Modules/<Pkg>` — required for Xcode package resolution |

## Critical Notes

- **NEVER share DerivedData between worktrees** — causes stale `Filters` module errors. Each worktree gets its own `.derivedData`.
- **NEVER run `make ios-build-ready`** — it requires Artifactory credentials not available in agent context.
- **If `SRC_SP` is empty** — no Xcode DerivedData exists yet. The user must first build the main repo in Xcode to populate the cache, then re-run step 5.
- **Registry packages** (`adlmvviewersdk`, `bim360offlineviewer`, `formsexpressionslib`) come from `art-toucan.autodesk.com` — VPN required for fresh resolution. Always use cache.

## Opening in Xcode

After bootstrap, open the workspace:
```
<worktree-path>/ios/PlanGrid.xcworkspace
```

For CLI builds, pass the cloned packages dir:
```bash
xcodebuild -workspace PlanGrid.xcworkspace \
  -scheme PlanGrid \
  -configuration Debug \
  -sdk iphonesimulator \
  -clonedSourcePackagesDirPath <worktree-path>/.derivedData/SourcePackages \
  build
```
