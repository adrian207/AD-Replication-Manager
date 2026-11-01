# AD-Replication-Manager

PowerShell tool that audits Active Directory replication across domain controllers, repairs common failures with `repadmin`, and verifies the result. It produces CSV and JSON reports and stable exit codes, so it can run on a schedule or in a pipeline.

## What it does

`Invoke-ADReplicationManager.ps1` runs in one of four modes:

| Mode | What happens |
|---|---|
| `Audit` (default) | Collects inbound replication partner metadata and failures for each DC and classifies issues |
| `Repair` | Audit, then repair actionable issues (`repadmin /syncall`). Asks for confirmation unless `-AutoRepair` is set |
| `Verify` | Checks current state with `repadmin /showrepl` |
| `AuditRepairVerify` | All three in sequence |

Issues are classified as **ReplicationFailure** (from `Get-ADReplicationFailure`), **StaleReplication** (no successful inbound replication from a partner for more than 24 hours) and **Connectivity** (DC unreachable). Connectivity issues are reported for manual follow-up, not repaired.

**Scope:** a list of DCs (`-DomainControllers DC01,DC02`), every DC in a site (`-Scope Site:<name>`) or the whole forest (`-Scope Forest`, which asks for confirmation).

**Optional features:**

- **Auto-heal** (`-AutoHeal -HealingPolicy Conservative|Moderate|Aggressive`): repairs only the issue types the policy allows, with a cooldown per DC and a cap on actions per run. Every action is logged to a healing history folder
- **Delta mode** (`-DeltaMode`): if the last run (within `-DeltaThresholdMinutes`) found issues, checks only those DCs; otherwise scans all. `-ForceFull` overrides
- **Health score** (`-EnableHealthScore`): a 0–100 score per run, saved to a history file
- **Notifications:** Slack, Teams or email, sent on errors, on issues, always or never (`-EmailNotification`)
- **Scheduled task** (`-CreateScheduledTask -TaskSchedule Hourly|Every4Hours|Daily|Weekly`): registers itself with Task Scheduler (runs under `pwsh.exe`)

Repairs go through `-WhatIf` / `-Confirm` (ConfirmImpact is High), so you can preview any run first.

## Requirements

- Windows with PowerShell 5.1 or 7. DCs are queried in parallel; on PowerShell 7 with `ForEach-Object -Parallel`, on 5.1 with a runspace pool
- RSAT Active Directory module (`ActiveDirectory`) and `repadmin` on the machine running the script
- A domain-joined machine. Domain Admin (or delegated replication rights) for repairs
- PowerShell 7 (`pwsh.exe`) installed if you use `-CreateScheduledTask`

## Quick start

```powershell
git clone https://github.com/adrian207/AD-Replication-Manager.git
cd AD-Replication-Manager

# Audit two DCs (read-only)
.\Invoke-ADReplicationManager.ps1 -Mode Audit -DomainControllers DC01,DC02

# Preview what a repair would do, without changing anything
.\Invoke-ADReplicationManager.ps1 -Mode AuditRepairVerify -DomainControllers DC01,DC02 -WhatIf

# Repair every DC in a site and keep a transcript
.\Invoke-ADReplicationManager.ps1 -Mode Repair -Scope Site:Default-First-Site-Name -AuditTrail
```

Run `Get-Help .\Invoke-ADReplicationManager.ps1 -Detailed` for every parameter.

## Output

Each run writes to `-OutputPath`, or to a new `ADRepl-<timestamp>` folder in the current directory:

| File | Contents |
|---|---|
| `ReplicationSnapshot.csv` | Per-DC replication state |
| `IdentifiedIssues.csv` | Issues found, with category and severity |
| `RepairActions.csv` | Repair actions taken and their results |
| `VerificationResults.csv` | Post-repair verification |
| `summary.json` | Machine-readable run summary |
| `execution.log` | Full log |

**Exit codes:** `0` healthy, `2` issues found, `3` one or more DCs unreachable, `4` error.

## Repository layout

```
Invoke-ADReplicationManager.ps1   The tool
Tests/                            Pester tests
Test-ADReplManager.ps1            Demo harness against real DCs (Audit mode and -WhatIf only)
docs/                             Operations, troubleshooting, auto-heal and delta mode guides
archive/                          The two v2 scripts this replaced
```

## Documentation

- [docs/OPERATIONS-MANUAL.md](docs/OPERATIONS-MANUAL.md)
- [docs/TROUBLESHOOTING-GUIDE.md](docs/TROUBLESHOOTING-GUIDE.md)
- [docs/AUTO-HEALING-GUIDE.md](docs/AUTO-HEALING-GUIDE.md)
- [docs/DELTA-MODE-GUIDE.md](docs/DELTA-MODE-GUIDE.md)
- [docs/MIGRATION-GUIDE.md](docs/MIGRATION-GUIDE.md) (from the v2 scripts)
- [CHANGELOG.md](CHANGELOG.md)

## Tests

```powershell
Invoke-Pester .\Tests
```

Requires Pester 5 and the ActiveDirectory module. See [TESTING-GUIDE.md](TESTING-GUIDE.md).

## License

MIT. See [LICENSE](LICENSE).

## Author

Adrian Johnson · [github.com/adrian207](https://github.com/adrian207)
