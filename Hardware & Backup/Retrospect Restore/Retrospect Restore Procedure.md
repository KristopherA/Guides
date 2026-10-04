# Retrospect: Restore Files from Server Backup

Step-by-step procedure for restoring files or folders from the Retrospect (Archive) backup server. Example used throughout: restoring the daily MySQL backup folders from the `data` server (data.example.com).

Last reviewed: 2026-07-09 (procedure tested successfully on this date)

---

## Before You Start

- Restores are run from the Retrospect console on the Retrospect (Archive) machine.
- Each server has two media sets: **Server Backups 2026** (onsite) and **Server Backups Offsite 2026**. Restore from the **onsite** set — the offsite set may ask for a disk stored in the offsite safe.
- Backups run daily at 2:00 AM, so a given day's files land in the *next day's* backup. Example: to recover files from May 8–9, restore from the May 10 or 11 backup.
- Only a limited number of days is kept in the recent list; older backups must be retrieved from the media set catalog (Step 4).

## Procedure

### 1. Start the Restore Assistant

1. In the Retrospect console, click the **Restore** button in the top toolbar.
2. On the Getting Started screen, choose **Restore selected files and folders**.
3. Click **Continue**.

### 2. Find the Backup

1. On the "Where do you want to restore from?" screen, type the source machine name (e.g. `data`) in the search box at the top right.
2. The list shows recent point-in-time backups. Check the **Media Set** column and use the entry from **Server Backups 2026** (onsite).

### 3. If Your Date Is Listed

Select it and skip to Step 5.

### 4. If Your Date Is NOT Listed — Retrieve It

1. Select any backup from the correct machine and onsite media set, then click **More Backups…**
2. The "Retrieve a backup" dialog lists every backup in the media set. Find the date/volume you want (remember the +1 day rule) and click **Retrieve**.
3. A Retrieve job runs — watch **Activities** to see it (the running-jobs count goes up by one, then back down). "Accessing backup…" can take several minutes.
4. When the job finishes, re-run the search in the search box to refresh the list. The retrieved backup now appears (e.g. the 5/11 backup).

### 5. Select the Files to Restore

1. Select the backup and click **Browse** (or Continue).
2. Click the disclosure triangle to expand the folder tree.
3. Check the files/folders you need (e.g. the `8` and `9` daily backup folders).
4. Click **Select**, then **Continue**.

### 6. Select the Destination

1. Choose the **Test** drive (on the Retrospect (Archive) machine) as the destination.
2. **CRITICAL:** check **Restore to a new folder**. If you skip this, Retrospect will OVERWRITE the destination drive.
3. Click **Continue**.

### 7. Run the Restore

1. Choose **Start Now**.
2. Immediately check **Activities**. If the restore needs a disk from the safe, it will show there.
3. Jobs are multi-threaded, so the restore usually starts right away — it only queues if it must read and write the same media set as another running job.

### 8. Deliver and Clean Up

1. When the restore completes, copy the files from the Test drive to their destination. For user restores, use the **"Restore for <user>"** folder on the **Systems Admin** share.
2. Delete the restored files from the Test drive once copied.

## Notes

- The `data` machine is being renamed to `data.example.com` in Retrospect for clarity.
- If a restore from the Offsite media set is unavoidable, expect to be prompted for a disk from the offsite safe.
