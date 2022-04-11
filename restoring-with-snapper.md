# **Restoring / to its previous snapshot:**

> [Taken from arch wiki](https://wiki.archlinux.org/title/snapper#Restoring_/_to_its_previous_snapshot)

To restore / using one of snapper's snapshots, first boot into a live Arch Linux USB/CD.

Mount the toplevel subvolume (subvolid=5). That is, omit any subvolid or subvol mount flags.

```zsh
cryptsetup open /dev/nvme0n1p4 root
mount /dev/mapper/root /mnt
```

Find the number of the snapshot that you want to recover:

```zsh
grep -r '<date>' /mnt/@snapshots/\*/info.xml
```

Can also read the info for me deets:

```zsh
nano /mnt/@snapshots/5/info.xml
```

Remember the number.

Now, move @ to another location (e.g. /@.broken) to save a copy of the current system.

Alternatively, simply delete @ using btrfs subvolume delete /mnt/@

```zsh
rm -rf /mnt/@
```

Create a read-write snapshot of the read-only snapshot snapper took:

```zsh
btrfs subvolume snapshot /mnt/@snapshots/number/snapshot /mnt/@
```

Where number is the number of the snapper snapshot you wish to restore.

Your / has now been restored to the previous snapshot. Now just simply reboot.
