# FSIyagi — Delete-Protection Filesystem

A Linux filesystem that keeps project folders from being deleted — by
accident, or by a runaway script. Reads, writes, and creation all pass
through untouched. **Only deletion is refused.**

```
os.remove()  shutil.rmtree()  find -delete  rm  perl unlink
                          │
                          ▼  they all collapse into the same syscall
                    unlink / rmdir
                          │
                    FS-AIIyagi  ──▶  refused (EPERM)
```

No matter which language or tool tries to delete, it passes through the
same point in the kernel, so blocking it there leaves no way around it.
This isn't a command-name check, so a script that imitates `rm` or calls
it under a different name is blocked just the same.

It also guards against wiping a file's *contents* without deleting it
(`> file.cpp`, a mistaken save in an editor, and so on) by keeping one
copy of a file's contents from just before it gets overwritten (the
Vault).

## Install

Download the deb package and install it:

```bash
sudo dpkg -i fs-aiiyagi_*.deb
sudo apt --fix-broken install   # if a dependency is missing
```

Nothing turns on automatically after install. Choosing the mount point
yourself is the point, so the service is started manually too.

## First use

### 1. Add your account to the group allowed to delete

Only processes in the `fsguard` group may delete. Everyone else — including
an admin account — has every delete attempt refused.

```bash
sudo gpasswd -a "$USER" fsguard
```

**Log in again for this to take effect.**

### 2. Prepare a partition with the management UI

```bash
fsai-manager
```

Pick a partition from the disk list, then **Format** and **Mount** it.
For a first try, start with an empty or disposable partition rather than
one you already rely on.

### 3. Use it as normal

Inside the mounted directory, creating, editing, and building files all
work exactly as they always have. The one difference: **trying to delete
is refused.**

```
$ rm important.txt
rm: cannot remove 'important.txt': Operation not permitted
```

When you genuinely need to delete something, borrow `fsguard` group
privileges explicitly through `fsai-admin`:

```bash
fsai-admin run rm important.txt
```

### 4. Check on things through the UI

The **Disks** tab in `fsai-manager` shows usage and mount state. The
**Tools** tab lets you find and restore preserved originals from the
Vault, inspect the policy, or undo a format if needed.

## Two storage modes

| | EXT4 MODE | NATIVE MODE |
|---|---|---|
| Storage format | Layered on standard ext4 | Own on-disk format |
| Recovery | Works with ordinary ext4 tools | Needs the dedicated fsck |
| Best for | Compatibility, existing setups | Performance, self-containment |

You choose which mode to use at mount time. Both modes share the same
delete-protection code, so the behavior is identical either way.

## Language

The UI follows your system language automatically. Supported: Korean,
English, Deutsch, Español, Français, Bahasa Indonesia, 日本語, Português,
Русский, Türkçe, Tiếng Việt, 中文. To force a specific language, add
`ui/language=<code>` to the config file (e.g. `ko`, `ja`).

## Troubleshooting

**Deletion fails with `Operation not permitted`**
That's the intended behavior. Run it through `fsai-admin run <command>`,
or check that your account is in the `fsguard` group (`groups`).

**The mount point shows up empty**
If the mount point already had content but the target partition is empty,
mounting is refused outright to prevent that content from silently
disappearing under a blank overlay. Double-check you picked the right
partition in `fsai-manager`.

**`Transport endpoint is not connected`**
The daemon died. Unmount and mount again.

**I want to undo a format**
If you enabled the "save a backup for undo" option when formatting, you
can undo it from the Tools tab in `fsai-manager`. The data area is never
touched by formatting, so as long as nothing new has been written to the
partition since, the original filesystem comes back intact.

## Learn more

For build instructions, on-disk format design, the reasoning behind the
policy decisions, and verification records, see `DEVNOTES.md`.
