# Lab 08: Copying Files and Directories — `cp`, `cp -R`, `cp -a`

**Series:** File Operations & Shell Fundamentals · **Lab 8 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (Tasks 11, 16, 18, 20), RHCE EX294 (Ansible `copy` module), CKA (`/etc/kubernetes/*`, kubeconfig backups), RHCA building blocks (RH342, RH358, RH318)  
**Prerequisite:** Labs 05–07  
**Time Estimate:** 40–55 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–13 practical · 14–18 advanced · 19–20 exam-realistic

---

## 🎯 Objective

Master `cp` and its three workhorse modes: single-file copy, recursive copy with `-R`, and **archive** copy with `-a` that preserves everything (ownership, timestamps, permissions, symlinks, ACLs, and SELinux contexts). Choosing the wrong flag silently breaks services on the exam — Apache, sshd, MariaDB all expect specific metadata.

---

## 🧠 Concept: A Copy Is a New Inode

Unlike a hard link, `cp` creates a **brand-new inode** with **new data blocks**. By default, the new file inherits **fresh** metadata:

| Attribute | Default `cp` behavior | With `-a` |
|---|---|---|
| Inode | New | New |
| Data | Copied | Copied |
| Owner | The **caller** (`whoami`) | **Preserved** from source |
| Group | Caller's primary group | **Preserved** from source |
| Mode | Source mode AND-ed with umask | **Preserved** exactly |
| atime/mtime | Set to **now** | **Preserved** |
| Symlinks | **Dereferenced** (file copied) | **Preserved as links** |
| SELinux context | Inherited from **target directory** | **Preserved** with `--preserve=context` |
| ACLs | Lost | **Preserved** with `--preserve=xattr,all` |

> **Why this matters on RHCSA Task 16:** Move `/var/www/html` content with plain `cp` and the new files inherit the **wrong** SELinux type — Apache returns 403. Use `cp -a` plus `restorecon`, or `cp -aZ`.

### Diagram: what happens when you cp

```
  cp src dst
    │
    ├── allocate new inode in dst's filesystem
    ├── copy data blocks
    ├── set metadata (default: caller's, target dir's context)
    └── done

  cp -a src dst  (archive)
    │
    ├── allocate new inode
    ├── copy data blocks
    ├── PRESERVE metadata from source (mode, owner, times, context, links)
    └── done
```

---

## 📚 `cp` Reference

| Flag | Long form | Purpose |
|---|---|---|
| `-r` / `-R` | `--recursive` | Copy directories recursively |
| `-a` | `--archive` | `-dR --preserve=all` — preserve everything |
| `-i` | `--interactive` | Prompt before overwrite |
| `-n` | `--no-clobber` | Never overwrite an existing file |
| `-f` | `--force` | Remove unwritable destination first |
| `-u` | `--update` | Copy only when source is newer (or dest missing) |
| `-v` | `--verbose` | Print each file copied |
| `-p` | `--preserve=mode,ownership,timestamps` | Preserve those three attributes |
| `--preserve=all` | — | Preserve mode, ownership, timestamps, links, xattr, context |
| `-d` | — | Preserve symlinks (don't follow) |
| `-L` | `--dereference` | Always follow symlinks (default for non-`-a`) |
| `-P` | `--no-dereference` | Never follow symlinks |
| `-Z` | — | Set destination context to **target dir's default** |
| `--parents` | — | Recreate the source path under the destination |
| `--backup[=METHOD]` | — | Make a backup of an existing destination |
| `-l` | `--link` | Create hard links instead of copying |
| `-s` | `--symbolic-link` | Create symbolic links instead of copying |
| `--sparse=WHEN` | — | Control sparse-file handling (`auto`, `always`, `never`) |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **RHCSA EX200** | Tasks 11 (archive), 16 (web deploy), 18 (configs), 20 (rollback copy) |
| **RHCE EX294** | `ansible.builtin.copy`/`fetch`/`template` mirror `cp` semantics |
| **CKA** | Backups: kubeconfig, etcd snapshots, CNI configs |
| **RHCA — RH342** | Pre-change snapshots of `/etc/*.conf` |
| **RHCA — RH358** | Service-config deployments need correct context |
| **RHCA — RH318 (Virt)** | `cp --sparse=always` for VM disk images |

---

## 🔧 The 20 Tasks

---

### Task 1 — Set up source and destination

**Purpose:** Build a realistic test bed with files, subdirs, and a symlink.

```bash
mkdir -p ~/cp-lab/src ~/cp-lab/dst
cd ~/cp-lab
echo "alpha content"  > src/alpha.txt
echo "beta content"   > src/beta.txt
mkdir -p src/sub
echo "deep file"      > src/sub/deep.txt
ln -s alpha.txt        src/alpha.link
ls -lR src
```

**Expected output:**

```
src:
total 8
-rw-r--r--. 1 ec2-user ec2-user 14 Sep 12 12:00 alpha.txt
lrwxrwxrwx. 1 ec2-user ec2-user  9 Sep 12 12:00 alpha.link -> alpha.txt
-rw-r--r--. 1 ec2-user ec2-user 13 Sep 12 12:00 beta.txt
drwxr-xr-x. 2 ec2-user ec2-user 22 Sep 12 12:00 sub

src/sub:
total 4
-rw-r--r--. 1 ec2-user ec2-user 10 Sep 12 12:00 deep.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | Create with missing parents |
| `echo "X" > file` | Write `X` to file (with newline) |
| `ln -s` | Create symlink (Lab 09) |
| `ls -lR` | Long, recursive listing |

**Output decoded**

| Element | Meaning |
|---|---|
| `src:` | Directory header |
| `alpha.link -> alpha.txt` | Symlink that targets the regular file |
| `src/sub:` | Subdirectory header |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mkdir: cannot create` | Use a path inside your home |

---

### Task 2 — Copy a single file into a directory

**Purpose:** Most basic case. Source name is preserved at the destination.

```bash
cp src/alpha.txt dst/
ls -l dst/
```

**Expected output:**

```
-rw-r--r--. 1 ec2-user ec2-user 14 Sep 12 12:05 alpha.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `cp src dst` | Source then destination |
| `dst/` | Trailing `/` → destination is a directory (explicit) |

**Output decoded**

| Column | Meaning |
|---|---|
| `-rw-r--r--.` | Default mode (umask applied) |
| `Sep 12 12:05` | mtime = **now** (no `-a`, so timestamps are fresh) |
| `alpha.txt` | Name preserved at destination |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cp: target 'dst/' is not a directory` | `dst/` doesn't exist — `mkdir -p dst/` first |

---

### Task 3 — Copy AND rename in one step

**Purpose:** Saves a separate `mv` if you want the destination to have a different name.

```bash
cp src/alpha.txt dst/alpha-renamed.txt
ls dst/
```

**Expected output:**

```
alpha.txt  alpha-renamed.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `cp src dst/newname` | Destination has explicit filename, not a trailing `/` |

**Output decoded**

| Token | Meaning |
|---|---|
| `alpha-renamed.txt` | New name at destination |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wanted both files at destination but only one | Without trailing `/`, the last arg is treated as a target — be explicit |

---

### Task 4 — Interactive overwrite with `-i`

**Purpose:** Defense against accidental overwrites.

```bash
cp -i src/alpha.txt dst/alpha.txt
```

**Expected interaction:**

```
cp: overwrite 'dst/alpha.txt'? y
```

**Switches**

| Flag | Meaning |
|---|---|
| `-i` | Interactive — prompt before each overwrite |

**Output decoded**

| Prompt | Action |
|---|---|
| `overwrite 'dst/alpha.txt'?` | Type `y` (yes) or `n` (no) |

**Why a sysadmin needs this:** Many distros alias `cp` to `cp -i` for root only. Type the full command on the exam to be sure.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Got prompt unexpectedly | Your shell has `alias cp='cp -i'` — type `\cp` to bypass once, or unalias |

---

### Task 5 — Never overwrite with `-n`

**Purpose:** Idempotent deploys — copy only when destination is missing.

```bash
echo "I should survive" > dst/alpha.txt
cp -n src/alpha.txt dst/alpha.txt
cat dst/alpha.txt
```

**Expected output:**

```
I should survive
```

**Switches**

| Flag | Meaning |
|---|---|
| `-n` | No-clobber — silently skip if destination exists |

**Output decoded**

| Line | Meaning |
|---|---|
| `I should survive` | Destination was not overwritten — proves `-n` worked |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Combined `-n -i` | `-n` wins — silent skip, no prompt |

---

### Task 6 — Force overwrite with `-f`

**Purpose:** Bypass an alias-induced prompt; remove an unwritable destination before writing.

```bash
chmod 000 dst/alpha.txt
cp -f src/alpha.txt dst/alpha.txt
ls -l dst/alpha.txt
```

**Expected output:**

```
-rw-r--r--. 1 ec2-user ec2-user 14 Sep 12 12:10 dst/alpha.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `-f` | Force — remove destination first if it can't be opened |

**Output decoded**

| Token | Meaning |
|---|---|
| New mode `-rw-r--r--.` | `cp` removed the unwritable target and created a new one with default mode |

**Why a sysadmin needs this:** Production scripts that overwrite read-only flag files.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Still `Permission denied` | `-f` doesn't bypass directory perms — fix the parent's mode |

---

### Task 7 — Recursive copy with `-r`

**Purpose:** Whole directory trees. Without `-r`, `cp` refuses directories.

```bash
rm -rf dst/*
cp -r src dst/
ls -lR dst/
```

**Expected output:**

```
dst/:
drwxr-xr-x. 3 ec2-user ec2-user 78 Sep 12 12:10 src

dst/src:
-rw-r--r--. 1 ec2-user ec2-user 14 Sep 12 12:10 alpha.txt
-rw-r--r--. 1 ec2-user ec2-user 13 Sep 12 12:10 alpha.link
-rw-r--r--. 1 ec2-user ec2-user 13 Sep 12 12:10 beta.txt
drwxr-xr-x. 2 ec2-user ec2-user 22 Sep 12 12:10 sub
```

**Switches**

| Flag | Meaning |
|---|---|
| `-r` / `-R` | Recursive — same behavior; `-R` is POSIX-correct |

**Output decoded**

| Observation | Meaning |
|---|---|
| `dst/src/` was created | `cp -r src dst/` puts a copy of `src` **inside** `dst` |
| `alpha.link` is now a **regular file** | `-r` followed the symlink |
| Timestamps say `Sep 12 12:10` | Set to **now** (not preserved) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `omitting directory 'src'` without `-r` | Add `-r` |

---

### Task 8 — The `src` vs `src/.` distinction

**Purpose:** Critical for "copy CONTENTS into target, not the dir itself."

```bash
rm -rf dst/*
cp -r src/. dst/
ls dst/
```

**Expected output:**

```
alpha.link  alpha.txt  beta.txt  sub
```

**Patterns table**

| Pattern | Result |
|---|---|
| `cp -r src dst/` | `dst/src/...` |
| `cp -r src/. dst/` | `dst/...` (contents directly inside dst) |
| `cp -r src/* dst/` | Same as above but **misses dotfiles** — be careful |

**Output decoded**

| Token | Meaning |
|---|---|
| `alpha.txt` is directly in `dst/` | `src/.` told cp to copy the contents, not the dir |

**Why a sysadmin needs this on RHCSA Task 16:** "Move web content to `/var/www/html`" — usually means contents, not the parent dir.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Dotfiles missed with `*` | Use `src/.` form, or enable dotglob: `shopt -s dotglob` |

---

### Task 9 — Archive mode: `cp -a`

**Purpose:** The single most important `cp` flag for RHCSA/RHCE/CKA. Preserves everything.

```bash
rm -rf dst/*
cp -a src dst/
ls -l dst/src/
```

**Expected output:**

```
-rw-r--r--. 1 ec2-user ec2-user 14 Sep 12 12:00 alpha.txt
lrwxrwxrwx. 1 ec2-user ec2-user  9 Sep 12 12:00 alpha.link -> alpha.txt
-rw-r--r--. 1 ec2-user ec2-user 13 Sep 12 12:00 beta.txt
drwxr-xr-x. 2 ec2-user ec2-user 22 Sep 12 12:00 sub
```

**Switches — what's inside `-a`**

| Flag inside `-a` | Preserves |
|---|---|
| `-d` | Symlinks (don't follow) |
| `-R` | Recurse |
| `--preserve=mode` | Permissions |
| `--preserve=ownership` | User and group |
| `--preserve=timestamps` | atime, mtime |
| `--preserve=links` | Hard-link relationships |
| `--preserve=context` | SELinux contexts |
| `--preserve=xattr` | Extended attributes (includes ACLs) |

**Output decoded**

| Element | Meaning |
|---|---|
| `alpha.link` | **Still a symlink** (preserved) |
| `Sep 12 12:00` | Original mtime kept (not "now") |
| Permissions, owner, group | Exact copies |

> **Memorize this:** **`cp -a` is the safe default for anything under `/etc`, `/var/www`, `/var/lib/<service>`.**

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Ownership not preserved | You need root for owner change across users — `sudo cp -a` |

---

### Task 10 — Verify SELinux context preservation

**Purpose:** `cp -a` keeps the source's SELinux type. Plain `cp` inherits target dir's type.

```bash
ls -Z src/alpha.txt dst/src/alpha.txt
cp src/alpha.txt /tmp/alpha.txt
ls -Z src/alpha.txt /tmp/alpha.txt
```

**Expected output:**

```
unconfined_u:object_r:user_home_t:s0 src/alpha.txt
unconfined_u:object_r:user_home_t:s0 dst/src/alpha.txt
unconfined_u:object_r:user_home_t:s0 src/alpha.txt
unconfined_u:object_r:user_tmp_t:s0  /tmp/alpha.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `-Z` (on ls) | Show SELinux context |

**Output decoded**

| Row | Meaning |
|---|---|
| Both `dst/src/alpha.txt` rows match `src` | `cp -a` preserved context ✓ |
| `/tmp/alpha.txt` shows `user_tmp_t` | Plain `cp` inherited `/tmp`'s default — wrong for service deploys |

**Why a sysadmin needs this on RHCSA Task 16:** Plain `cp` → Apache 403. `cp -a` + `restorecon` → 200.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Both same | Both source and target dir share the type — not always wrong |

---

### Task 11 — Apply target context with `-Z`

**Purpose:** Sometimes you WANT the destination to take its directory's default context (e.g., deploying into `/var/www/html`).

```bash
cp -Z src/alpha.txt /tmp/alpha-Z.txt
ls -Z /tmp/alpha-Z.txt
```

**Expected output:**

```
unconfined_u:object_r:user_tmp_t:s0 /tmp/alpha-Z.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `-Z` | Set destination context to the **default for the target directory** (think: inline `restorecon`) |

**Output decoded**

| Token | Meaning |
|---|---|
| `user_tmp_t` | Target's default — different from source's `user_home_t` |

**Decision summary**

| Goal | Flag |
|---|---|
| Preserve source context | `cp -a` or `cp --preserve=context` |
| Apply target dir's default | `cp -Z` |
| Recompute from policy | `cp` (plain), then `restorecon -Rv <path>` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-Z` ignored | Old coreutils — use `restorecon` separately |

---

### Task 12 — Verbose with `-v`

**Purpose:** See exactly what `cp` did. Indispensable for batch jobs.

```bash
cp -av src dst/verbose-copy
```

**Expected output:**

```
'src' -> 'dst/verbose-copy'
'src/alpha.txt' -> 'dst/verbose-copy/alpha.txt'
'src/alpha.link' -> 'dst/verbose-copy/alpha.link'
'src/beta.txt' -> 'dst/verbose-copy/beta.txt'
'src/sub' -> 'dst/verbose-copy/sub'
'src/sub/deep.txt' -> 'dst/verbose-copy/sub/deep.txt'
```

**Switches**

| Flag | Meaning |
|---|---|
| `-a` | Archive (preserve everything) |
| `-v` | Verbose — print every operation |

**Output decoded**

| Line | Meaning |
|---|---|
| `'src/...' -> 'dst/verbose-copy/...'` | Each source → destination pair |

**Why a sysadmin needs this:** On the exam, `-v` gives you instant proof during demos.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Too much output | Pipe through `tail -20` or filter with `grep` |

---

### Task 13 — Update-only with `-u`

**Purpose:** Copy only when source is newer. Useful in sync-style scripts.

```bash
echo "newer content" > src/alpha.txt
cp -uv src/alpha.txt dst/src/alpha.txt
echo "still newer? maybe not" > dst/src/alpha.txt
cp -uv src/alpha.txt dst/src/alpha.txt
```

**Expected output:**

```
'src/alpha.txt' -> 'dst/src/alpha.txt'
(no output the second time — source not newer)
```

**Switches**

| Flag | Meaning |
|---|---|
| `-u` | Update — copy only when source mtime > destination mtime, or destination missing |
| `-v` | Verbose |

**Output decoded**

| Run | Outcome |
|---|---|
| First | Source newer → copied |
| Second | Destination newer (just edited) → silently skipped |

**Why a sysadmin needs this:** Lightweight sync. For real sync, use `rsync`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Skipped when expected to copy | mtime drift across timezones — verify with `stat` |

---

### Task 14 — Backup before overwrite with `--backup`

**Purpose:** Auto-keep a copy of the destination before clobbering.

```bash
cp --backup=numbered src/alpha.txt dst/src/alpha.txt
cp --backup=numbered src/alpha.txt dst/src/alpha.txt
ls dst/src/
```

**Expected output:**

```
alpha.txt  alpha.txt.~1~  alpha.txt.~2~  alpha.link  beta.txt  sub
```

**Switches**

| Flag | Meaning |
|---|---|
| `--backup` (= `existing`) | Backup as `alpha.txt~`, then numbered if more |
| `--backup=simple` | Always `alpha.txt~` |
| `--backup=numbered` | `alpha.txt.~1~`, `~2~`, ... |
| `--backup=none` (default) | No backup |
| `-b` | Short form of `--backup=existing` |

**Output decoded**

| File | Meaning |
|---|---|
| `alpha.txt` | The newly copied file |
| `alpha.txt.~1~` | First backup (the previous content) |
| `alpha.txt.~2~` | Second backup (from the second `cp`) |

**Why a sysadmin needs this:** Never edit `/etc/*.conf` without a backup. `cp --backup=numbered original` before editing.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Backups not appearing | You may have run with `-n` (no-clobber) — destination was skipped, no backup needed |

---

### Task 15 — Recreate source paths with `--parents`

**Purpose:** Copy a deep file while reconstructing its directory chain.

```bash
mkdir -p ~/cp-lab/landing
cp --parents src/sub/deep.txt ~/cp-lab/landing/
ls -lR ~/cp-lab/landing/
```

**Expected output:**

```
landing:
src

landing/src:
sub

landing/src/sub:
deep.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `--parents` | Recreate the source's directory chain under destination |

**Output decoded**

| Path | Meaning |
|---|---|
| `landing/src/sub/deep.txt` | Full chain preserved |

**Why a sysadmin needs this:** Selective backups that retain directory structure.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cp: cannot copy a directory ... into itself` | Don't put destination under source |

---

### Task 16 — Make hard links instead of copies with `-l`

**Purpose:** "Copy" without using more disk — `cp -l` creates hard links (same inode, no data duplication).

```bash
mkdir -p ~/cp-lab/hardlink-dst
cp -l src/alpha.txt ~/cp-lab/hardlink-dst/
ls -li src/alpha.txt ~/cp-lab/hardlink-dst/alpha.txt
```

**Expected output (same inode number):**

```
1048577 -rw-r--r--. 2 ec2-user ec2-user 14 Sep 12 12:00 src/alpha.txt
1048577 -rw-r--r--. 2 ec2-user ec2-user 14 Sep 12 12:00 /home/ec2-user/cp-lab/hardlink-dst/alpha.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `-l` | Hard-link instead of copying |

**Output decoded**

| Token | Meaning |
|---|---|
| `1048577` (both rows) | **Same inode** — both names point to one file |
| Link count `2` | Two names exist for this inode |

**Why a sysadmin needs this:** Snapshot-style backups across a single filesystem — zero extra disk usage.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cannot create hard link 'cross-fs'` | Hard links can't cross filesystems — use `cp -a` or `ln -s` |

---

### Task 17 — Make symlinks instead of copies with `-s`

**Purpose:** Same idea, but cross-filesystem capable. The destination becomes a symlink.

```bash
mkdir -p ~/cp-lab/symlink-dst
cp -s "$PWD/src/alpha.txt" ~/cp-lab/symlink-dst/
ls -l ~/cp-lab/symlink-dst/
```

**Expected output:**

```
lrwxrwxrwx. 1 ec2-user ec2-user 31 Sep 12 13:00 alpha.txt -> /home/ec2-user/cp-lab/src/alpha.txt
```

**Switches**

| Flag | Meaning |
|---|---|
| `-s` | Symbolic-link instead of copying |
| `$PWD` | Absolute path of current dir (symlinks need absolute targets to survive moves) |

**Output decoded**

| Token | Meaning |
|---|---|
| `lrwxrwxrwx.` | First char `l` = symlink |
| `alpha.txt -> /home/.../alpha.txt` | Points back to the source |

**Why a sysadmin needs this:** Cross-filesystem "logical copy" without duplicating data.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cp: '...' is not in the same directory` and relative paths break | Use absolute paths with `-s` |

---

### Task 18 — Sparse files with `--sparse`

**Purpose:** VM disk images and database files have huge "holes" of zeros. `--sparse=always` makes the copy sparse.

```bash
truncate -s 100M ~/cp-lab/big_sparse.img
cp --sparse=always ~/cp-lab/big_sparse.img ~/cp-lab/big_sparse.copy.img
ls -lh ~/cp-lab/big_sparse*.img
du -h ~/cp-lab/big_sparse*.img
```

**Expected output:**

```
-rw-r--r--. 1 ec2-user ec2-user 100M Sep 12 13:10 /home/ec2-user/cp-lab/big_sparse.img
-rw-r--r--. 1 ec2-user ec2-user 100M Sep 12 13:10 /home/ec2-user/cp-lab/big_sparse.copy.img
0	/home/ec2-user/cp-lab/big_sparse.img
0	/home/ec2-user/cp-lab/big_sparse.copy.img
```

**Switches**

| Flag | Meaning |
|---|---|
| `--sparse=always` | Always treat zero runs as holes |
| `--sparse=auto` (default) | Detect when likely useful |
| `--sparse=never` | Allocate every block, even zeros |
| `truncate -s 100M` | Create a 100 MiB sparse file (no blocks allocated) |

**Output decoded**

| Tool | Reports |
|---|---|
| `ls -lh` | Logical size (100M) |
| `du -h` | **Disk** size (0 — no blocks actually used) |

**Why a sysadmin needs this on RHCA RH318 (Virt):** A 100 GB VM disk on disk takes 100 GB unless sparse — and you have many of them.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Copied disk is huge | Source wasn't sparse — `--sparse=always` re-introduces holes |

---

### Task 19 — Combine `cp -a` with `restorecon` for service deploys

**Purpose:** The canonical exam pattern: archive-copy plus context fix.

```bash
sudo mkdir -p /web-deploy
sudo cp -av src/. /web-deploy/
sudo ls -lZ /web-deploy/
sudo restorecon -Rv /web-deploy
sudo ls -lZ /web-deploy/
```

**Expected output (excerpt):**

```
'src/./alpha.txt' -> '/web-deploy/./alpha.txt'
'src/./beta.txt' -> '/web-deploy/./beta.txt'
'src/./alpha.link' -> '/web-deploy/./alpha.link'
'src/./sub' -> '/web-deploy/./sub'
'src/./sub/deep.txt' -> '/web-deploy/./sub/deep.txt'

-rw-r--r--. 1 ec2-user ec2-user unconfined_u:object_r:user_home_t:s0 ... alpha.txt
...
Relabeled /web-deploy/alpha.txt from unconfined_u:object_r:user_home_t:s0 to system_u:object_r:default_t:s0
...
```

**Switches**

| Token | Meaning |
|---|---|
| `cp -av` | Archive + verbose |
| `restorecon -R` | Recompute contexts recursively from policy |
| `restorecon -v` | Verbose — print every change |

**Output decoded**

| Phase | What happened |
|---|---|
| After cp | All files at `/web-deploy/` still labeled `user_home_t` (source's type) |
| After restorecon | Files relabeled to `/web-deploy`'s policy-defined type |

**Why on RHCSA Task 16:** Without `restorecon`, Apache returns 403. With it, you pass.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `restorecon` says no change | Either source and target dir share type, or no policy defines this path — use `semanage fcontext -a` |

---

### Task 20 — Exam-style scenario

**Task statement (RHCSA-style):** *"Deploy `/home/admin/site/` into `/var/www/html`, preserving ownership/timestamps but with the correct Apache SELinux context. Verify."*

```bash
sudo mkdir -p /home/admin/site
echo "Hello, RHCSA" | sudo tee /home/admin/site/index.html > /dev/null
sudo ls -lZ /home/admin/site/index.html

sudo cp -av /home/admin/site/. /var/www/html/
sudo restorecon -Rv /var/www/html
sudo ls -lZ /var/www/html/index.html

sudo systemctl restart httpd 2>/dev/null || true
curl -sI http://localhost/ | head -1 2>/dev/null || echo "(httpd not running — skip)"
```

**Expected output (excerpts):**

```
unconfined_u:object_r:admin_home_t:s0 /home/admin/site/index.html
'/home/admin/site/./index.html' -> '/var/www/html/./index.html'
Relabeled /var/www/html/index.html from unconfined_u:object_r:admin_home_t:s0 to system_u:object_r:httpd_sys_content_t:s0
-rw-r--r--. 1 root root system_u:object_r:httpd_sys_content_t:s0 14 Sep 12 13:30 /var/www/html/index.html
HTTP/1.1 200 OK
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `cp -av /home/admin/site/. /var/www/html/` | Archive (preserve metadata) + verbose; `.` = contents |
| `restorecon -Rv /var/www/html` | Fix SELinux types to `httpd_sys_content_t` |
| `ls -lZ` | Verify final state |
| `curl -sI` | Functional verification — Apache returns 200 |

**Cleanup**

```bash
cd ~
rm -rf ~/cp-lab
sudo rm -rf /web-deploy /home/admin/site
sudo rm -f /tmp/alpha.txt /tmp/alpha-Z.txt
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `403 Forbidden` from curl | Skipped `restorecon` — run it |
| `httpd` not installed | `sudo dnf install httpd && sudo systemctl enable --now httpd` |

---

## 🔍 `cp` Decision Guide

```
Single file?                                  → cp src dst
Whole directory?                              → cp -r src dst
Whole directory + preserve EVERYTHING?        → cp -a src dst
Copy CONTENTS into target dir?                → cp -a src/. dst/
Copying INTO a service dir (correct context)? → cp -aZ src dst   OR cp -a + restorecon
Need to keep existing destination intact?     → cp -n
Want to be prompted before overwrite?         → cp -i
Want a backup before overwriting?             → cp --backup=numbered
Only copy if source is newer?                 → cp -u
Need to debug what was copied?                → add -v
Recreate source's directory tree under dst?   → cp --parents
Hard-link in place of copy (same FS)?         → cp -l
Symlink in place of copy (any FS)?            → cp -s
Sparse VM image / large zero-runs?            → cp --sparse=always
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Set up `~/cp-lab/src` + `~/cp-lab/dst` with files, subdir, symlink
- [ ] 02 Copy a single file into a directory
- [ ] 03 Copy and rename in one step
- [ ] 04 Interactive overwrite with `-i`
- [ ] 05 Skip overwrite with `-n`
- [ ] 06 Force overwrite with `-f`
- [ ] 07 Recursive copy with `-r`
- [ ] 08 Use `src/.` to copy contents (not dir itself)
- [ ] 09 Archive mode `cp -a` preserves everything
- [ ] 10 Compare `cp` vs `cp -a` SELinux context behavior
- [ ] 11 Apply target context with `-Z`
- [ ] 12 Verbose audit with `-v`
- [ ] 13 Update-only with `-u`
- [ ] 14 Backups with `--backup=numbered`
- [ ] 15 Reconstruct paths with `--parents`
- [ ] 16 Hard-link in place of copy with `-l`
- [ ] 17 Symlink in place of copy with `-s`
- [ ] 18 Sparse-file copy with `--sparse=always`
- [ ] 19 `cp -a` + `restorecon` for service deploys
- [ ] 20 Exam scenario: deploy into `/var/www/html` and verify with `curl`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Plain `cp` into `/var/www/html` | Apache returns 403 | Use `cp -a` then `restorecon -Rv /var/www/html` |
| `cp -r src dst` when you wanted contents | `dst/src/...` instead of `dst/...` | Use `cp -r src/. dst/` |
| Using `*` glob — misses dotfiles | `.htaccess`, `.env` lost | Use `src/.` or `cp -a` |
| Forgetting `-a` for `/etc/<service>/` | Service starts but behaves oddly | Replace `cp -r` with `cp -a` and re-test |
| Following a symlink unintentionally | Copy is regular file, not link | Add `-d` (or `-a`) |
| Overwriting a config without backup | Cannot revert | `cp --backup=numbered` before edit |
| `cp -Z` when you wanted source context | Wrong context | Use `cp -a` |
| Cross-FS hard link attempt | `cannot create hard link` | Use `cp -a` or `ln -s` |

---

## 📌 Exam Strategy

**RHCSA EX200**
- Default to `cp -a` for any "copy into service directory" task.
- Always `ls -lZ` on both source and destination after copying.
- After deploying content under SELinux-aware paths, **always** chase with `restorecon -Rv`.

**RHCE EX294 (Ansible)**
- `ansible.builtin.copy` parameters map to `cp` semantics:
  - `mode:`, `owner:`, `group:` ↔ `cp -a`
  - `setype:`, `seuser:`, `serole:` ↔ context preservation/override
  - `backup: yes` ↔ `cp --backup`

**CKA**
- Common copy operations:
  - `cp -a /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/backup/`
  - `cp -a /etc/cni/net.d /tmp/cni-backup`
  - `cp ~/.kube/config ~/.kube/config.bak` before editing context

**RHCA**
- RH342: pre-change snapshot `cp -a /etc/conf-file{,.$(date +%s).bak}`
- RH358: deploy service configs with `cp -a` + `restorecon`
- RH318 (Virt): `cp --sparse=always` for VM images

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 06 — SELinux contexts | Why `cp -a` vs `cp -Z` matters |
| Lab 09 — Hard/soft links | `cp -a` preserves; `cp` (plain) dereferences |
| Lab 10 — `mv` | Cross-FS `mv` = `cp -a` + `rm`; same metadata rules apply |
| Lab 11 — `rm -rf` | Pair with `cp --backup` for safe edits |
| Task 16 — Apache document root | Direct application of `cp -a` + `restorecon` |
| Task 11 — `tar` archive | `cp -a` to stage files before archiving |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
