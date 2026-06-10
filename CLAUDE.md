# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **SQL Server Integration Services (SSIS)** project (Project Deployment Model) that orchestrates database backup migrations and data movement for the Diamond360 platform across Dev, QA, and Live environments.

Built artifact: `bin/Development/Diamond360DeploymentPackages.ispac` (SSIS project archive, a ZIP format).

## Build

Open `Diamond360DeploymentPackages_Upgrade.slnx` in **Visual Studio with SQL Server Data Tools (SSDT)** installed, then:

- **Build:** Build > Build Solution — outputs to `bin/Development/Diamond360DeploymentPackages.ispac`
- **Run a single package:** Right-click any `.dtsx` file in Solution Explorer → Execute Package
- **Deploy to catalog:** Right-click project → Deploy, point at target SQL Server SSIS Catalog

No command-line build scripts exist; all build/run operations go through SSDT or `dtexec.exe`.

## Architecture

### Package Categories (15 total, all are entry points)

| Purpose | Packages |
|---|---|
| Diamond360 DB backups | `MoveDiamond360To{Dev,QA,Live}.dtsx` |
| Diamond360 raw data | `MoveDiamond360_RawDataTo{Dev,QA,Live}.dtsx` |
| Prospector exports | `MoveProspectorExportsTo{Dev,QA,Live}.dtsx` |
| Group Insurance exports | `MoveGroupInsuranceExportsTo{Dev,QA,Live}.dtsx` |
| Other exports | `MoveEINFinderToLive.dtsx`, `MoveFreeERISAToDev.dtsx` |
| **Orchestrator** | `DeployDataCycleDatabases.dtsx` |

### Orchestrator Pattern

`DeployDataCycleDatabases.dtsx` is the master orchestrator. It uses **Execute Package Task** nodes with precedence constraints to call the individual Move packages in controlled order (QA → Live). It groups calls into three sequence containers: Diamond360, GroupInsuranceExports, ProspectorExports.

All other packages can also run standalone (e.g., triggered by SQL Server Agent jobs).

### Task Types Used

- **Execute SQL Task** — T-SQL for backups, maintenance, index optimization
- **DbMaintenanceTSQLExecuteTask** — database backup operations
- **FileSystemTask** — file operations
- **ScriptTask** — custom .NET code embedded in packages
- **ExecutePackageTask** — calls child packages (used in orchestrator)

## Key Configuration Files

| File | Purpose |
|---|---|
| `Diamond360DeploymentPackages_Upgrade.dtproj` | Project manifest: all package metadata, connections, parameters, version info, deployment config |
| `CINPSQL20_PROCESSING.ReferenceData.conmgr` | Shared project-level connection manager |
| `Project.params` | Project-level parameter definitions (currently empty) |
| `*.dtproj.user` | User-local IDE settings (not committed to source control) |

## Server / Connection Topology

- **Source/control server:** `CINPSQL20\PROCESSING` (ReferenceData database)
- **Primary server:** `CINPSQL20.SBMEDIA.COM\PROCESSING`
- **Destination server:** `CINPSQL22.SBMEDIA.COM`
- **Backup file share:** `\\Sunvwin360db02\amc\D360Backups\`

## Security

Protection level is `EncryptSensitiveWithUserKey` — credentials are encrypted with the Windows user key of whoever last saved the project. If the project is opened on a different user account, sensitive connection string passwords will need to be re-entered.

## Technical Notes

- Target server version: **SQL Server 2025**
- SSIS schema version: 9.0.1.0 (compatible with SSDT for VS 2019/2022)
- Key package variables: `Diamond360DataDrive` (default `E`), `Diamond360LogDrive` (default `J`), `Diamond360FileName` (set at runtime)
- Package version builds are tracked in `.dtproj`; the highest build number indicates the most-iterated package
