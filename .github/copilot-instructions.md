# Windows Package Manager Community Repository - Copilot Instructions

## Repository Overview

This is the **Windows Package Manager (WinGet)** community repository containing **~415,000+ manifest files** (2.3GB) for software packages installable via `winget`. The repository is a **manifest-only** repository - no application code, just YAML metadata files describing how to install Windows applications.

**Key Facts:**
- **Primary Language:** YAML manifest files, PowerShell scripts for tooling
- **Target Runtime:** Windows 10/11, Windows Package Manager client
- **Size:** Large repository with alphabetically organized manifests
- **Schema:** Uses multi-file YAML manifests (version 1.10.0 recommended, 1.9.0 also supported)
- **Supported Installers:** MSIX, MSI, APPX, EXE only (scripts and fonts not supported)

## Critical: How This Repository Works

**This is NOT a traditional code repository.** You typically work with **manifest files only**, not application source code. Each PR should modify **exactly one package** (one manifest set).

### Manifest Structure (Multi-File Format Required)

Manifests are located in: `manifests/<first-letter>/<Publisher>/<Package>/<Version>/`

Example: `manifests/m/Microsoft/WindowsTerminal/1.0.1401.0/`

**Required files per version:**
1. `<PackageIdentifier>.yaml` - Version file (references other files)
2. `<PackageIdentifier>.installer.yaml` - Installer details, URLs, SHA256
3. `<PackageIdentifier>.locale.en-US.yaml` - Default locale metadata

**File Naming:** Must match `PackageIdentifier` exactly (case-sensitive).

## Validation and Testing - CRITICAL

**ALWAYS validate manifests before submitting.** The validation pipeline will reject invalid manifests.

### Local Validation (Required)

```bash
# Enable local manifest files (run once in admin PowerShell)
winget settings --enable LocalManifestFiles

# Validate syntax
winget validate --manifest <path-to-manifest-directory>

# Test installation
winget install --manifest <path-to-manifest-directory>
```

**Note:** `<path-to-manifest-directory>` is the VERSION directory containing all three YAML files.

### Windows Sandbox Testing (Highly Recommended)

The **preferred testing method** to ensure no system dependencies are required:

```powershell
.\Tools\SandboxTest.ps1 <path-to-manifest-directory>
```

**Prerequisites:**
- Windows 10/11 Pro/Enterprise/Education (NOT Home edition)
- Enable Windows Sandbox: `Enable-WindowsOptionalFeature -FeatureName "Containers-DisposableClientVM" -All -Online`
- Restart required after enabling

## GitHub Actions Workflows

**Two workflows run automatically on PRs:**

1. **PSScriptAnalyzer** (`.github/workflows/scriptAnalyzer.yaml`)
   - Runs on: Changes to `*.ps1` files
   - Validates: PowerShell script quality
   - Failures: Script errors only (rarely relevant for manifest PRs)

2. **Spell Checking** (`.github/workflows/spellCheck.yaml`)
   - Runs on: All PRs
   - Validates: Spelling in manifest fields
   - Common issues: Package names, technical terms may be flagged

## Azure DevOps Pipelines (Primary Validation)

**The main validation happens in Azure Pipelines** (not GitHub Actions):

**Validation Pipeline** (`DevOpsPipelineDefinitions/validation-pipeline.yaml`):
- Triggers on: PRs to master branch affecting `manifests/*`
- **Critical checks performed:**
  - Manifest syntax validation
  - Installer download and hash verification
  - SmartScreen and security scanning
  - Policy compliance checks
  - Installation verification
  - Domain URL validation
  - Catalog content verification

**Publish Pipeline:**
- Runs after: PR is merged
- Timeline: Changes published within ~1 hour after merge
- Status: Check [WinGetSvc-Publish Pipeline](https://dev.azure.com/shine-oss/winget-pkgs/_build?definitionId=12)

## PowerShell Tools (Located in `Tools/`)

### YamlCreate.ps1 - Manifest Creation Tool

**Primary tool for creating manifests.** Interactive script that prompts for package details.

```powershell
.\Tools\YamlCreate.ps1 [-PackageIdentifier <id>] [-PackageVersion <ver>] [-Mode <1-5>]
```

**Features:**
- Downloads installer to calculate SHA256
- Validates manifest automatically
- Can test in Windows Sandbox
- Can auto-submit PR via GitHub CLI

### SandboxTest.ps1 - Testing Tool

**Tests manifests in isolated Windows Sandbox environment.**

```powershell
.\Tools\SandboxTest.ps1 <path-to-manifest> [-Prerelease] [-Clean]
```

**Common options:**
- `-Prerelease` - Use preview WinGet version
- `-SkipManifestValidation` - Skip validation if already validated
- `-Clean` - Force re-download of WinGet

### Other Tools

- `CheckDependencies.ps1` - Verify package dependencies
- `CheckDisplayVersions.ps1` - Validate version matching
- `PRTest.ps1` - Test PRs before submission
- `WingetVersionManager.ps1` - Manage WinGet versions

## Manifest Authoring Guidelines

### PackageVersion vs DisplayVersion

**Critical distinction:**
- `PackageVersion` - Marketing/user-facing version (sortable)
- `DisplayVersion` - Registry ARP entry version (what installer writes)
- **When to use AppsAndFeaturesEntries:** When these don't match, or DisplayVersion is missing/unsortable

### Version Sorting in WinGet

**Important:** WinGet strips leading non-digit characters before first `.` (e.g., `v1.0.1` → `1.0.1`)

**Comparison rules:**
1. Split on `.` into Parts (integer + string suffix)
2. Compare integers first
3. If equal, Part WITHOUT string suffix is GREATER (e.g., `1.2.34` > `1.2.34-beta`)
4. If both have suffix, lexicographic comparison

### Installer Architecture

**Set architecture based on INSTALLED binaries, not installer itself.** An x86 installer that installs x64 binaries should have `Architecture: x64`.

## Common Issues and Solutions

### Issue: "DisplayVersion contains version but is unsortable"
**Solution:** Use `AppsAndFeaturesEntries` with `PackageVersion` (sortable) and `DisplayVersion` (actual registry value).

### Issue: "Manifest validation fails locally"
**Cause:** Not enabling local manifests OR wrong path
**Solution:** 
```powershell
winget settings --enable LocalManifestFiles
# Pass directory containing all 3 YAML files, not individual file
```

### Issue: "Installer hash mismatch"
**Cause:** Installer URL changed or re-uploaded with different binary
**Solution:** Re-download installer and recalculate SHA256. Never fabricate hashes.

### Issue: "Package upgrades in a loop"
**Cause:** 
1. `DisplayVersion` missing from registry (shows as "Unknown")
2. `PackageVersion` ≠ `DisplayVersion` without `AppsAndFeaturesEntries`
**Solution:** Add `AppsAndFeaturesEntries` with correct `DisplayVersion` to every manifest version.

### Issue: "PR bot assigns back to me with validation error"
**Action Required:** You have 7 days to fix before auto-close. Check bot comment for specific issue.

## File Encodings and Line Endings

**Critical for Windows compatibility:**

From `.editorconfig`:
- **All files:** UTF-8, CRLF line endings, trim trailing whitespace
- **YAML files:** 2-space indentation (no tabs)
- **Markdown files:** Tab indentation (accessibility)
- **PowerShell files:** UTF-8 with BOM

## PR Guidelines

**From `PULL_REQUEST_TEMPLATE.md`:**

**Checklist for every manifest PR:**
- [ ] Signed [Contributor License Agreement](https://cla.opensource.microsoft.com/microsoft/winget-pkgs)
- [ ] No other open PRs for same package
- [ ] Only ONE manifest modified per PR
- [ ] Validated locally with `winget validate --manifest <path>`
- [ ] Tested locally with `winget install --manifest <path>`
- [ ] Conforms to schema 1.10.0 or 1.9.0

**Manifest Testing Requirements:**
- Application installs **unattended** (no user interaction)
- Application version matches `PackageVersion` or `AppsAndFeaturesEntries` present
- Application publisher matches manifest `Publisher` or `AppsAndFeaturesEntries` present
- Application name matches `PackageName` or `AppsAndFeaturesEntries` present

## Working with Git in This Repo

**Standard workflow:**
1. Fork repository (use `--depth=1` for shallow clone to save time/space)
2. Clone your fork locally
3. Create feature branch: `git checkout -b <publisher>/<package>/<version>`
4. Make changes to manifests
5. Validate and test locally
6. Commit and push to your fork
7. Create PR to `microsoft/winget-pkgs:master`

**Important:** PRs merge to `master` branch only. Path must affect `manifests/*` to trigger validation.

## Schema Documentation

**Current recommended versions:**
- **1.10.0** - Latest, full feature support
- **1.9.0** - Also supported, use if 1.10.0 features not needed

**Schema locations:**
- JSON schemas: `schemas/JSON/`
- Documentation: `doc/manifest/schema/<version>/`

**Key schema files:**
- `version.yaml` - Package identifier, version, manifest type
- `installer.yaml` - InstallerType, Architecture, InstallerUrl, InstallerSha256
- `defaultLocale.yaml` - Publisher, PackageName, License, ShortDescription, etc.

## VSCode Recommended Setup

**Extensions** (`.vscode/extensions.json`):
- `redhat.vscode-yaml` - YAML validation and intellisense
- `EditorConfig.EditorConfig` - Enforce .editorconfig settings

**Settings** (`.vscode/settings.json`):
- PowerShell files: UTF-8 with BOM, auto-guess encoding

## What NOT to Do

**Never:**
- Submit multiple packages in one PR
- Modify files outside `manifests/` unless fixing tools (rare)
- Submit script-based installers or fonts (not supported)
- Fabricate installer hashes or skip validation
- Include secrets or credentials in manifests
- Submit without testing in Windows Sandbox (if available)

## Trust These Instructions

**This document is comprehensive and tested.** Only search for additional information if:
- You encounter an error not covered here
- You need details on specific schema field behavior
- Instructions are incomplete for your specific case

**For more details, refer to:**
- `CONTRIBUTING.md` - Contribution process
- `doc/Authoring.md` - Detailed manifest authoring guide
- `doc/README.md` - Tool usage and submission process
- `doc/FAQ.md` - Common questions and issues
