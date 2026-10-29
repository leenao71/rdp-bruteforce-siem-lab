# Teardown

To preserve the lab for future reference without running it continuously:

1. Shut down all VMs cleanly (`sudo shutdown now` on Linux VMs, Start → Power →
   Shut down on Windows).
2. In UTM, the `-BACKUP` clones can be kept as point-in-time snapshots, or deleted
   to reclaim disk space if no longer needed.
3. To fully remove: right-click each VM in UTM → Delete, confirm. This removes the
   virtual disks.
4. ISOs (Ubuntu, Windows, Kali) can be deleted from disk once VMs are no longer
   needed, or kept for rebuilding later.

