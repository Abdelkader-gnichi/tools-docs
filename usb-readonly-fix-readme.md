# Troubleshooting a Read-Only USB Drive in Linux

If your USB drive is mounted as read-only, it is usually due to a physical lock or file system corruption. Follow these steps to diagnose and fix the issue.

---

## 1. Check for Physical Write-Protection
Check if your USB stick has a physical **lock switch** on the side. Ensure it is toggled to the **unlocked** position.

## 2. Identify the Device Name
Open your terminal and run the following command to find your USB device:

```bash
lsblk
```

**Example Output:**
Look for your device based on size (e.g., 14.6G). In this case, the device is `/dev/sdb1`, mounted at `/media/piva/ACHICH`.

```text
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sdb           8:16   1  14,6G  0 disk 
└─sdb1        8:17   1  14,6G  0 part /media/piva/ACHICH
```

---

## 3. Unmount the Drive
Before repairing the file system, you must unmount the partition:

```bash
sudo umount /media/piva/ACHICH
```

---

## 4. Repair File System Errors (`fsck`)
Run the file system check tool. Replace `/dev/sdb1` with the identifier you found in step 2.

```bash
sudo fsck /dev/sdb1
```

### Recommended responses for common prompts:
During the process, you may see several prompts. Based on your logs, here is how to handle them:

1.  **Boot sector differences:** Choose `3) No action` (or `1` to sync if preferred).
2.  **Dirty bit is set:** Choose `1) Remove dirty bit`.
3.  **Free cluster summary wrong:** Choose `1) Correct`.
4.  **Write changes:** Choose `1) Write changes` to save the repairs.

**Output Summary:**
```text
fsck.fat 4.2 (2021-01-31)
Performing repairs...
Reclaimed unused clusters.
Writing changes...
/dev/sdb1: 30 files, 439700/1917993 clusters
```

---

## 5. Verify the Fix
After the repair is complete, re-insert the USB or mount it manually. Test if you can now write to the drive by copying a folder:

```bash
cp -r /home/piva/Pictures/Screenshots /media/piva/ACHICH/
```

If the command completes without a "Read-only file system" error, your USB drive is successfully repaired.