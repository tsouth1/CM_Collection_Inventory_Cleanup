# CM Collection Inventory Cleanup

PowerShell scripts for finding and safely removing unused Configuration Manager (SCCM / MECM) collections.

The workflow has three steps:

1. **Inventory**: export every collection, with its rules, deployments and references, to CSV.
2. **Review**: open the CSV in Excel and put `Y` in the `to be deleted` column for each collection to remove.
3. **Delete**: preview what the deletion script would do, then run it with `-Execute`. Every collection is checked against the live site and backed up before it is deleted.

| Script | Purpose | Changes the site? |
|---|---|---|
| `Scripts\Collection-Inventory.v2.ps1` | Exports the collection inventory | No (read-only) |
| `Scripts\Delete-MarkedCollections.ps1` | Deletes collections marked `Y` in the inventory CSV | Only with `-Execute` |

## Requirements

- Windows PowerShell 5.1.
- An account with read access to the SMS Provider. Deleting also needs rights to delete collections.
- Network access to the site server over DCOM/RPC (the same access the CM console uses).
- **Deleting only:** the Configuration Manager console installed on the machine that runs `Delete-MarkedCollections.ps1`. The script loads the `ConfigurationManager` module from the console's install folder.

Running both scripts on the site server itself (with `-SiteServer localhost`) meets all of these.

## 1. Inventory

```powershell
cd .\Scripts
.\Collection-Inventory.v2.ps1 -SiteServer CM01.contoso.com -SiteCode PS1
```

| Parameter | Default | Description |
|---|---|---|
| `-SiteServer` | This computer | Server hosting the SMS Provider. |
| `-SiteCode` | `LEA` | Site code. **Always pass this** when targeting any site other than LEA. To detect the site code from the provider instead, pass `-SiteCode ''`. |
| `-OutputPath` | `Data\CollectionInventory_<yyyyMMdd-HHmm>.csv` | Inventory CSV to write. |
| `-ExcludeNameSuffix` | `-sec` | Leave out collections whose name ends with any of these values (not case-sensitive). `@()` turns it off. |
| `-ExcludeLimitedTo` | `LEA013D0` | Leave out collections limited to any of these collection IDs. `@()` turns it off. |

The script writes two files:

- **`CollectionInventory_<timestamp>.csv`**: one row per collection. The first columns are:

  | Column | Content |
  |---|---|
  | `to be deleted` | Blank, for you to fill in |
  | `CollectionName` | |
  | `ReferencedByCount` | Number of collections that depend on this one |
  | `ReferencedBy` | Those collections, with the type of reference |
  | `ObjectPath` | Folder in the console, e.g. `Device Collections\Workstations\Office` |
  | `CollectionID` | |

  These are followed by the type, member count, limiting collection, include/exclude/query/direct rules, deployments, evaluation schedule, incremental evaluation, last refresh/change times and comment.
- **`CollectionInventory_<timestamp>_References.csv`**: one row per collection-to-collection reference: `Limiting`, `Include`, `Exclude`, or `Query` (a collection ID found in a query rule's WQL text).

Cells with several values are separated by ` | `, and referenced collections are written as `Name (ID)`. A limiting collection that no longer exists shows as `<unknown> (ID)`.

### Exclusions

These collections are left out of the inventory CSV, so they can't be marked for deletion:

- built-in collections (IDs starting with `SMS`),
- collections whose name ends with `-sec` (`HR-sec`, `Finance-SEC`; not `App-Security`),
- collections limited to `LEA013D0`.

Excluded collections are still read for their references. If an excluded collection uses another collection, that collection's `ReferencedBy` still lists it and it won't look unused. The console summary shows how many collections were excluded and why.

Example: also exclude names ending in `-test`, and turn off the limiting collection exclusion:

```powershell
.\Collection-Inventory.v2.ps1 -SiteServer CM01.contoso.com -SiteCode PS1 -ExcludeNameSuffix '-sec', '-test' -ExcludeLimitedTo @()
```

## 2. Review

Open the inventory CSV and type `Y` in the `to be deleted` column for each collection to remove. Case and surrounding spaces are ignored. Any other value, including `Yes`, means "keep".

Good candidates are collections with no members, no deployments and a `ReferencedByCount` of 0. Save the file as CSV.

## 3. Delete

Always preview first. Without `-Execute`, nothing is deleted:

```powershell
.\Delete-MarkedCollections.ps1 -InputPath ..\Data\CollectionInventory_20261027-0900.csv -SiteServer CM01.contoso.com -SiteCode PS1
```

The preview lists the collections it would delete, plus a **Skipped** table with the reason for every marked row it will not delete. When the result looks right, run the same command with `-Execute`:

```powershell
.\Delete-MarkedCollections.ps1 -InputPath ..\Data\CollectionInventory_20261027-0900.csv -SiteServer CM01.contoso.com -SiteCode PS1 -Execute
```

The script lists the collections again and asks you to type `DELETE` (in capitals) to continue. Anything else cancels.

| Parameter | Default | Description |
|---|---|---|
| `-InputPath` | (required) | Inventory CSV with rows marked `Y`. |
| `-SiteServer` | (required) | Server hosting the SMS Provider. |
| `-SiteCode` | (required) | Site code. |
| `-BackupFolder` | `Data\DeletionBackups` | Folder for the JSON backups. |
| `-LogPath` | `Data\DeletionLog.csv` | Log CSV. Every run appends to it. |
| `-Execute` | Off | Actually delete. Without it, the script only previews. |

### Safety checks

The CSV can be out of date, so every marked row is checked against the live site first. A row is skipped if:

- it is a built-in collection (ID starts with `SMS`),
- the collection no longer exists, or its name differs from the CSV (re-run the inventory),
- it has any deployment,
- another collection references it as a limiting, include or exclude collection, or by ID in a query rule,
- its collection ID appears more than once in the CSV.

If you mark a collection and also the collections that depend on it, only the dependents are deleted on the first run. Run the inventory and deletion again to remove the rest.

### Backups and log

Before deleting each collection, the script writes `<CollectionID>_<timestamp>.json` to the backup folder with the collection's properties and membership rules. If the backup fails, that collection is not deleted.

Every preview, skip, deletion and failure is appended to the log CSV with a run ID and the mode (`Preview` or `Execute`).

## Output files

Everything is written under `Data\` by default. That folder is in `.gitignore`, so inventory exports, logs and backups are never committed.
