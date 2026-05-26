# Lab: Copying Files and Directories — `cp`, `cp -R`, `cp -a`

**Series:** linux-ops-mastery — RHCSA Essential Tools & File Operations
**Subjects covered:** `cp` for single-file copy, `-R`/`-r` for recursive directory copy, `-a` "archive" mode (preserves owner, group, mode, timestamps, symlinks, SELinux context), `-i` interactive overwrite prompts, `-n` no-clobber, `-u` update-only-if-newer, `-v` verbose, `-p` preserve mode/owner/timestamps, `--preserve=context` for SELinux, the difference between `cp -R src dst/` and `cp -R src/. dst/`, the trailing-slash gotcha, and the production reflex of "use `-a` unless you have a reason not to"
**Career arcs covered:** RHCSA (every "copy this config to that location" task), RHCE (Ansible `copy:` module — same semantics), SRE (backup snapshots before changes), DevOps (Dockerfile `COPY` inherits from cp semantics), AI/MLOps (dataset / checkpoint replication across nodes)
**Prerequisite:** Labs 05–07 (navigation, listing, timestamps)
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 foundation (single-file copy) · 2 recursive copy · 3 the `-a` preserve-everything pattern · 4 interactive/no-clobber/update flags · 5 the trailing-slash and `src/.` tricks · 6 RHCSA exam-realistic capstone

---

## Objective

Copy files and directories without losing metadata, without overwriting things you did not mean to, and without falling into the "did I copy the directory or only its contents?" trap. By the end of this lab you can perform any RHCSA-style copy task in two seconds — and you understand exactly why `-a` is the senior-engineer default.

The capstone is an exam-realistic prompt: *"Make a complete backup of `/etc/ssh/` to `/root/ssh-backup/` such that every file's owner, group, mode, timestamps, and SELinux context are preserved exactly. Verify by comparing one host key's metadata between source and destination."*

> **Lab safety note:** Every command in this lab reads from `/etc/ssh` and your sandbox under `/tmp/cp-lab`, and writes only into `/tmp/cp-lab` or `/root/...`. No system file is modified.

---

## Concept: A Copy Is a Choice About Metadata

The bytes of a file are easy to copy. The **metadata** is where copies get interesting:

| Metadata | Default `cp` preserves? | `cp -p` preserves? | `cp -a` preserves? |
|---|---|---|---|
| File contents | yes | yes | yes |
| Mode (permissions) | no (uses umask) | yes | yes |
| Owner / group | no (current user) | yes (if root) | yes (if root) |
| mtime | no (now) | yes | yes |
| atime | no (now) | yes | yes |
| ctime | no (now — kernel always sets) | no | no |
| Symlinks | followed (target copied) | followed | preserved as symlinks |
| Hard links | broken (two independent copies) | broken | preserved |
| SELinux context | no (inherits destination) | no | yes |
| ACLs | no | no | yes |
| Extended attributes | no | no | yes |

```
   ┌──────────────────────────────────────────────────────┐
   │                cp source dest                        │
   ├──────────────────────────────────────────────────────┤
   │  default   → bytes only; new inode; new metadata     │
   │  cp -p     → bytes + mode/owner/group/times          │
   │  cp -R/-r  → recurse into directories                │
   │  cp -a     → -dR --preserve=all   (the everything)   │
   └──────────────────────────────────────────────────────┘
```

> **Why this matters:** On the RHCSA exam, the prompt almost always says "preserve" or "with original permissions" or "as a backup." Each of those words is a hint to reach for `-a` instead of `-r`. Using the wrong flag silently loses metadata, and the grader checks the result.

---

## 📜 Why `cp -a` Exists — The Story

In **Unix v1 (1971)** `cp` did one job: copy bytes. Metadata? Use `chmod`, `chown`, `touch` separately. As Unix matured into a multi-user system, "make a backup of `/etc`" became a nightly chore — and the manual sequence "copy then chmod then chown then touch" became an error-prone ritual.

The Berkeley team responded with `-p` (preserve), shipped in **4.3 BSD (1986)**, which preserved owner/group/mode/timestamps in one operation. GNU coreutils later added `-d` (preserve symlinks), `--preserve=context` (SELinux), and finally `-a` (archive — equivalent to `-dR --preserve=all`) to combine every "do the right thing for a backup" flag into one letter.

So `cp -a` is the result of 35 years of "I forgot to preserve X." It is the convergent answer to "what do I want when I copy a directory tree for safekeeping?" The answer is **everything**.

> **The point of the story:** Default `cp` is the "give me a clean copy with the current user's defaults" tool. `cp -a` is the "treat this as a backup" tool. Pick the right one for the job.

---

## 👪 The cp Family — Who Lives There

### Common flags

| Flag | Meaning |
|---|---|
| `-r` / `-R` | Recursive (required for directories) |
| `-i` | Interactive — prompt before overwrite |
| `-n` | No-clobber — skip if destination exists |
| `-f` | Force overwrite (default) |
| `-u` | Update only — copy only if source is newer or destination missing |
| `-v` | Verbose — print each copy |
| `-p` | Preserve mode, owner, group, timestamps |
| `--preserve=mode,owner,group,timestamps,links,context,xattr,all` | Granular preserve |
| `-a` | Archive — `-dR --preserve=all` (the everything) |
| `-l` | Hard-link instead of copy (when possible) |
| `-s` | Make symlink instead of copy |
| `-L` | Follow symlinks in source |
| `-P` | Never follow symlinks (preserve as symlinks) |
| `-d` | `-P --preserve=links` (preserve symlinks and hardlinks) |
| `-t TARGET` | Specify destination first (so source args can be repeated/scripted) |
| `-T` | Treat destination as a file, not a directory |

### Related commands

| Command | Notes |
|---|---|
| `mv` | Rename / move (no metadata to copy, just rename) |
| `rsync -a` | Same idea as `cp -a` but resumable, networked, with `--delete` |
| `install -m MODE` | Copy with explicit mode; common in Makefiles |
| `tar c ... \| tar x ...` | Old-school metadata-preserving copy |
| `cpio -p` | The original "copy preserving everything" — predates `cp -a` |

> **The point of the family tree:** For RHCSA, `cp` + `-a`/`-r` covers everything. For multi-machine work, `rsync -a`. For "build artifacts," `install`. For one-time archive operations, `tar`.

---

## 🔬 The Anatomy of a Copy — In One Diagram

```
$ cp -a /etc/ssh /root/ssh-backup
       │       │ │
       │       │ └─ destination — a directory (will be CREATED if missing, or used as PARENT if exists)
       │       └─ source — a directory
       └─ -a flag = -dR --preserve=all

What the shell + cp actually do:
  1. stat the source to learn its type and metadata.
  2. If dest does not exist, create it (mkdir).
  3. For each entry in source:
       - regular file  → read source bytes, write to dest/name, copy metadata
       - directory     → recurse
       - symlink (-d)  → preserve as symlink, do NOT follow
       - hardlink (-d) → preserve the link count
  4. Apply final metadata (timestamps last so they are not bumped by writes).

Trailing-slash trap:
  cp -a src  dst    → if dst exists, places src AS dst/src
                       (final layout: dst/src/...)
  cp -a src/ dst    → SAME behavior as above (trailing / on src has no effect in cp)
  cp -a src/. dst   → copies the CONTENTS of src into dst (no extra level)
  cp -aT src  dst   → forces "treat dst as a file" — copies CONTENTS, dst becomes "src"
```

> **Reading rule:** The position of the slash on the **destination** rarely matters; the difference between `src` and `src/.` on the **source** is what controls whether you get `dst/src/...` or `dst/...`.

---

## 📚 cp Reference Table

| Task | Command | Notes |
|---|---|---|
| Single-file copy | `cp /etc/hosts /tmp/hosts.bak` | Loses mode/owner/timestamps |
| Single-file with preservation | `cp -p /etc/hosts /tmp/hosts.bak` | Preserves mode/owner/times |
| Directory copy | `cp -r /etc/ssh /tmp/ssh.bak` | Recursive |
| Archive copy (recommended) | `cp -a /etc/ssh /tmp/ssh.bak` | Preserves everything |
| Copy contents (not the dir itself) | `cp -a /etc/ssh/. /tmp/ssh-contents/` | Use `/.` on source |
| Verbose | `cp -av /etc/ssh /tmp/ssh.bak` | Prints each copy |
| Interactive (prompt before clobber) | `cp -i a b` | Y/N per overwrite |
| No-clobber | `cp -n a b` | Skip if dest exists |
| Update-only-if-newer | `cp -u src dst` | Diff by mtime |
| Force overwrite | `cp -f a b` | Default; rarely needed |
| Multiple sources, one target dir | `cp f1 f2 f3 /target/` | Target must be a dir (or use `-t`) |
| Specify target first (-t form) | `cp -t /target/ f1 f2 f3` | Script-friendly |
| Make a hard link instead | `cp -l a b` | Same inode |
| Make a symlink instead | `cp -s a b` | Soft link |
| Preserve SELinux only | `cp --preserve=context a b` | Without `-a` |
| Preserve owner only | `cp --preserve=owner a b` | Needs root for owner change |

> **Rule one of cp:** When in doubt, `cp -a` and verify. The flag costs nothing extra; missing metadata costs you the grade or the incident.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | "Copy `/etc/foo` to `/tmp/foo-backup` preserving permissions and contexts" is the canonical exam phrasing. `cp -a` solves it. |
| **RHCE candidate** | Ansible's `copy:` module mirrors `cp -a` semantics through `mode:`, `owner:`, `group:`, and SELinux relabeling. |
| **SRE / Platform** | Before any risky config change: `cp -a /etc/sshd_config /root/sshd_config.bak-$(date +%s)`. Roll back with `cp -a` back. |
| **DevOps** | Dockerfile `COPY src dst` is the build-time analogue. Inside images, `cp -a` is used for layered initialization. |
| **AI / MLOps** | Dataset replication across NFS or local disks: `cp -a /shared/datasets/v1 /local/scratch/datasets/v1` keeps timestamps so cache-validation scripts work. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **copy → preserve → verify** habit.

---

### Task 1 — Single-file copy with and without `-p`

**Purpose:** Build the sandbox, copy a file with default `cp`, then re-copy with `-p`, and prove the difference in metadata.

```bash
mkdir -p /tmp/cp-lab && cd /tmp/cp-lab

cp /etc/hosts hosts-plain.copy
ls -l /etc/hosts hosts-plain.copy
stat -c 'm=%y u=%U g=%G mode=%a' /etc/hosts hosts-plain.copy

cp -p /etc/hosts hosts-preserved.copy
ls -l /etc/hosts hosts-preserved.copy
stat -c 'm=%y u=%U g=%G mode=%a' /etc/hosts hosts-preserved.copy
```

**Human-Readable Breakdown:** Make the sandbox, copy `/etc/hosts` two ways — plain (no metadata preservation) and `-p` (preserve mode/owner/group/timestamps). Compare the metadata side-by-side.

**Reading it left to right:** Plain `cp` performs a `read()` from the source and `write()` to a freshly created destination — the kernel assigns mtime/atime to **now** and owner/group to the **calling user**. `cp -p` issues a final `chmod`/`chown`/`utimes` to align metadata with the source.

**The story:** This is the experiment that converts you from "cp copies files" to "cp copies bytes; metadata is a choice." Once you have run it, you reach for `-p` (or `-a`) automatically.

**Expected output:**

```text
-rw-r--r--. 1 root     root     158 May 21 14:33 /etc/hosts
-rw-r--r--. 1 ec2-user ec2-user 158 May 26 13:40 hosts-plain.copy
m=2026-05-21 14:33:18 u=root g=root mode=644
m=2026-05-26 13:40:00 u=ec2-user g=ec2-user mode=644
-rw-r--r--. 1 root     root     158 May 21 14:33 /etc/hosts
-rw-r--r--. 1 ec2-user ec2-user 158 May 21 14:33 hosts-preserved.copy
m=2026-05-21 14:33:18 u=root g=root mode=644
m=2026-05-21 14:33:18 u=ec2-user g=ec2-user mode=644
```

**Switches**

| Token | Meaning |
|---|---|
| `cp SRC DST` | Plain copy |
| `cp -p SRC DST` | Preserve mode/owner/group/timestamps |
| `stat -c 'FMT'` | Custom stat output |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Owner did not transfer | You are not root — `-p` can only set owner if you have CAP_CHOWN |
| mtime is "now" with `-p` | Filesystem may not support setting mtime — rare |
| Mode differs by `0644` vs `664` | Source file mode was `664` — `-p` preserves it |

---

### Task 2 — Recursive directory copy with `-r`

**Purpose:** Use `-r` (or `-R`) to copy a directory tree. Notice that plain `-r` does **not** preserve owner/timestamps/contexts.

```bash
cd /tmp/cp-lab

cp -r /etc/ssh ssh-r.copy
ls -ld ssh-r.copy
ls -l ssh-r.copy | head -n 5
stat -c 'mode=%a u=%U g=%G m=%y' ssh-r.copy ssh-r.copy/sshd_config
```

**Human-Readable Breakdown:** Recursively copy `/etc/ssh` into the sandbox with `-r`. List the copied directory and one file inside it, then compare metadata against the source.

**Reading it left to right:** `cp -r SRC DST` recurses into SRC's directories. For each file, the default (no preservation) applies — owner becomes you, timestamps become now, mode obeys your umask combined with source mode.

**The story:** Most beginners learn `cp -r` and think they have done a backup. They have copied bytes, not state. The metadata loss only shows up later — when SSH refuses to use the host key because permissions are too open, or when `find -mtime` returns no results because everything has the same recent mtime.

**Expected output:**

```text
drwxr-xr-x. 4 ec2-user ec2-user 175 May 26 13:42 ssh-r.copy
-rw-r--r--. 1 ec2-user ec2-user 4434 May 26 13:42 sshd_config
-rw-r--r--. 1 ec2-user ec2-user 1872 May 26 13:42 ssh_config
...
mode=755 u=ec2-user g=ec2-user m=2026-05-26 13:42:00
mode=644 u=ec2-user g=ec2-user m=2026-05-26 13:42:00
```

**Switches**

| Token | Meaning |
|---|---|
| `cp -r SRC DST` | Recursive copy |
| `cp -R SRC DST` | Same as `-r` (POSIX form) |
| `ls -ld DIR` | Long-list the directory itself |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cp: -r not specified; omitting directory '/etc/ssh'` | Add `-r` |
| Permission denied reading host key files | Run as root (`sudo cp -r ...`) — keys are mode 0600 root-only |
| Owner is wrong | Use `-p` or `-a` (and run as root) |

---

### Task 3 — Archive copy with `-a` (the everything)

**Purpose:** Use `-a` to preserve mode, owner, group, timestamps, symlinks, hard links, ACLs, and SELinux context all at once.

```bash
sudo -i
cd /tmp/cp-lab

cp -a /etc/ssh ssh-a.copy
ls -ldZ ssh-a.copy
ls -lZ  ssh-a.copy/sshd_config /etc/ssh/sshd_config
stat -c 'mode=%a u=%U g=%G C=%C m=%y' ssh-a.copy/sshd_config /etc/ssh/sshd_config
```

**Human-Readable Breakdown:** Become root. Use `cp -a` to copy `/etc/ssh` with everything preserved. Compare the destination's `sshd_config` against the source — mode, owner, group, SELinux context, and mtime should all match.

**Reading it left to right:** `cp -a` is `-dR --preserve=all`: `-d` preserves symlinks and hardlinks, `-R` recurses, `--preserve=all` keeps mode/owner/group/timestamps/links/context/xattrs. The result is byte-and-metadata identical to the source.

**The story:** Once you have used `-a` and verified the output, you stop typing `-r` for backups. The cost is zero. The benefit is "passes the exam grader, doesn't surprise the daemon."

**Expected output:**

```text
drwxr-xr-x. 4 root root system_u:object_r:etc_t:s0 175 May 21 14:33 ssh-a.copy
-rw-------. 1 root root system_u:object_r:etc_t:s0 4434 May 21 14:33 ssh-a.copy/sshd_config
-rw-------. 1 root root system_u:object_r:etc_t:s0 4434 May 21 14:33 /etc/ssh/sshd_config
mode=600 u=root g=root C=system_u:object_r:etc_t:s0 m=2026-05-21 14:33:18
mode=600 u=root g=root C=system_u:object_r:etc_t:s0 m=2026-05-21 14:33:18
```

**Switches**

| Token | Meaning |
|---|---|
| `cp -a SRC DST` | Archive copy |
| `--preserve=all` | Same as `-a`'s preservation set |
| `--preserve=context` | Just SELinux context |
| `ls -lZ` / `ls -ldZ` | Include SELinux context column |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Owner did not transfer | Need root (`sudo -i`) |
| Context shows `default_t` | Filesystem may not support xattrs, or `--preserve=context` failed — verify with `ls -Z` |
| Symlinks were followed and copied as files | Use `-a` (preserves symlinks), not `-r` |

---

### Task 4 — Safety flags: `-i`, `-n`, `-u`, `-v`

**Purpose:** Practice the four flags that protect you from accidental overwrites and noisy output: `-i` (interactive), `-n` (no-clobber), `-u` (update), `-v` (verbose).

```bash
cd /tmp/cp-lab

mkdir -p safety && cd safety
echo "original" > target.txt

# Interactive — prompts before overwrite
echo "new" > src.txt
cp -i src.txt target.txt
# (answer 'n' to keep original, 'y' to overwrite)

# No-clobber — skips silently
cp -n src.txt target.txt
cat target.txt

# Update — only overwrites if source is newer
touch -d "1 hour ago" old-src.txt
cp -u old-src.txt target.txt
cat target.txt
sleep 1
touch new-src.txt
cp -u new-src.txt target.txt
cat target.txt

# Verbose
cp -v src.txt verbose-result.txt
```

**Human-Readable Breakdown:** Create a `target.txt`, then try to overwrite it under four different safety regimes. `-i` asks. `-n` silently skips. `-u` overwrites only if the source is newer. `-v` prints each copy operation.

**Reading it left to right:** `-i` causes `cp` to read from stdin before overwriting. `-n` is "no" — the destination already exists, do nothing. `-u` compares mtimes and acts only when the source is strictly newer. `-v` prints `'src' -> 'dst'` for each operation.

**The story:** These are the seatbelts. In production scripts that loop over many files, `-n` prevents disasters when sources and destinations overlap; `-u` provides cheap idempotence; `-v` aids debugging. In interactive use, `-i` is a paranoid safety net that nobody regrets.

**Expected output:**

```text
cp: overwrite 'target.txt'? n
original
original
                              ← (no change — source older with -u)
                              ← (new file created)
'src.txt' -> 'verbose-result.txt'
```

**Switches**

| Token | Meaning |
|---|---|
| `-i` | Interactive overwrite prompt |
| `-n` | No-clobber |
| `-u` | Update only if source newer |
| `-v` | Verbose |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-i` did not prompt | Output is not a TTY (e.g. in a script) — `-i` is suppressed |
| `-n` silently skipped what you wanted to overwrite | Wrong default — use `-f` or remove `-n` |
| `-u` did not overwrite a newer source | Verify mtimes with `stat`; clock skew between machines breaks `-u` |

---

### Task 5 — The trailing slash and `src/.` content-copy trick

**Purpose:** Master the most common confusion in `cp -r`: "do I want the directory itself at the destination, or its contents?"

```bash
cd /tmp/cp-lab
rm -rf foo bar baz qux

mkdir foo
echo "A" > foo/a.txt
echo "B" > foo/b.txt

# Case 1 — destination dir exists; cp places SOURCE INSIDE it
mkdir bar
cp -a foo bar
ls -R bar      # -> bar/foo/a.txt

# Case 2 — destination dir does NOT exist; cp makes it WITH source's contents
cp -a foo baz
ls -R baz      # -> baz/a.txt   (no extra `foo` level)

# Case 3 — copy CONTENTS of source into existing destination
mkdir qux
cp -a foo/. qux
ls -R qux      # -> qux/a.txt   (contents only — what you usually want)

# Case 4 — force "treat dest as a file" with -T
mkdir -p one
echo "X" > one/x.txt
cp -aT one two
ls -R two      # -> two/x.txt (same as case 2)
```

**Human-Readable Breakdown:** Build a small source tree, then copy it four ways to see what destination layout each one produces. The crucial distinction: when the destination exists, plain `cp -a foo bar` places `foo` inside `bar`; `cp -a foo/. bar` copies only the contents.

**Reading it left to right:** Case 1 — dest exists → cp creates `bar/foo`. Case 2 — dest missing → cp creates `baz` as the new "foo." Case 3 — `foo/.` means "everything inside foo" → qux receives the contents, not the foo level. Case 4 — `-T` forces "destination is the renamed source," even if it already exists.

**The story:** Every `cp -r` bug in production scripts is this gotcha. Watch the layout once with your own hands and you stop guessing.

**Expected output:**

```text
bar:
foo

bar/foo:
a.txt  b.txt

baz:
a.txt  b.txt

qux:
a.txt  b.txt

two:
x.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `cp -a SRC DST` | Standard copy |
| `cp -a SRC/. DST` | Copy contents only (DST must exist) |
| `cp -aT SRC DST` | Treat DST as a file/dir to be created with SRC's contents |
| `rm -rf` | Remove old test data (Lab 11 deep dive) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Got `bar/foo/...` when expecting `bar/...` | Use `cp -a foo/. bar` |
| Got `bar/...` when expecting `bar/foo/...` | The dest did not exist; pre-create with `mkdir` |
| `-T` complains "Not a directory" | Destination exists as a file; remove it first |

---

### Task 6 — Capstone: RHCSA-realistic full-metadata backup

**Task statement:** *"Make a complete backup of `/etc/ssh/` to `/root/ssh-backup/` such that every file's owner, group, mode, timestamps, and SELinux context are preserved exactly. Verify by comparing `sshd_config` metadata between source and destination."*

**Purpose:** Execute a real exam-style preserved backup end-to-end, then verify metadata.

```bash
sudo -i

cp -a /etc/ssh /root/ssh-backup

ls -ldZ /etc/ssh /root/ssh-backup
diff -q /etc/ssh/sshd_config /root/ssh-backup/sshd_config

echo "--- source ---"
stat -c 'mode=%a u=%U g=%G C=%C m=%y' /etc/ssh/sshd_config
echo "--- backup ---"
stat -c 'mode=%a u=%U g=%G C=%C m=%y' /root/ssh-backup/sshd_config

test -d /root/ssh-backup && echo "VERIFY: backup directory exists"
test "$(stat -c '%C' /etc/ssh/sshd_config)" = "$(stat -c '%C' /root/ssh-backup/sshd_config)" \
  && echo "VERIFY: SELinux contexts match"
test "$(stat -c '%a' /etc/ssh/sshd_config)" = "$(stat -c '%a' /root/ssh-backup/sshd_config)" \
  && echo "VERIFY: modes match"
```

**Human-Readable Breakdown:** Become root, run `cp -a` to clone `/etc/ssh` to `/root/ssh-backup`. Verify with `ls -ldZ`, `diff -q` (no differences), and a side-by-side `stat` of `sshd_config`. The final `test` lines compare specific fields to fail loudly if anything drifted.

**Layer stack you built:**

```text
/etc/ssh                  ─── original config dir
   │
   ▼  cp -a
/root/ssh-backup          ─── preserved copy
   ├── identical contents (diff -q empty)
   ├── identical owner/group (root:root)
   ├── identical mode (per file)
   ├── identical mtime (per file)
   ├── identical SELinux context (etc_t / sshd_key_t)
   └── symlinks preserved as symlinks (if any)
```

**The story:** This is the **canonical 60-second exam answer** for any "preserved backup" task. Memorize the spine: `cp -a /src /dst` → `diff -q` for content → `stat -c '%a %U %G %C'` for metadata. The grader script does the same comparison.

**Expected verification output:**

```text
drwxr-xr-x. 4 root root system_u:object_r:etc_t:s0 175 May 21 14:33 /etc/ssh
drwxr-xr-x. 4 root root system_u:object_r:etc_t:s0 175 May 21 14:33 /root/ssh-backup
--- source ---
mode=600 u=root g=root C=system_u:object_r:etc_t:s0 m=2026-05-21 14:33:18
--- backup ---
mode=600 u=root g=root C=system_u:object_r:etc_t:s0 m=2026-05-21 14:33:18
VERIFY: backup directory exists
VERIFY: SELinux contexts match
VERIFY: modes match
```

**Cleanup**

```bash
rm -rf /tmp/cp-lab /root/ssh-backup
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Owner is wrong | Need root (`sudo -i`) before running `cp -a` |
| Mode differs | Used `-r` instead of `-a` — re-run with `-a` |
| `diff -q` reports differences | Bytes do not match — re-run; verify no edits between |
| SELinux context wrong | xattrs not preserved on target FS — confirm `xattr` mount option |

---

## 🔍 cp Decision Guide

```
Got files to copy?
  │
  ├── "Single file, just the bytes"
  │       └── ✅ cp src dst
  │
  ├── "Single file, preserve metadata"
  │       └── ✅ cp -p src dst
  │
  ├── "Whole directory, just contents"
  │       └── ✅ cp -r src dst             (loses metadata)
  │
  ├── "Whole directory, preserve EVERYTHING (recommended)"
  │       └── ✅ cp -a src dst
  │
  ├── "Copy the CONTENTS of a dir (no extra parent level)"
  │       └── ✅ cp -a src/. dst/
  │
  ├── "Be paranoid about overwrites"
  │       └── ✅ cp -i src dst             (prompt)
  │       └── ✅ cp -n src dst             (skip)
  │       └── ✅ cp -u src dst             (only if newer)
  │
  ├── "Bandwidth- or atomicity-sensitive across machines"
  │       └── ✅ rsync -a src/ host:/dst/
  │
  └── "Build artifacts with explicit mode"
          └── ✅ install -m 0755 src /usr/local/bin/dst
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Set up `/tmp/cp-lab` and compare `cp` vs `cp -p` metadata
- [ ] 02 Recursively copy a directory with `cp -r` and inspect lost metadata
- [ ] 03 Recopy with `cp -a` and verify mode/owner/group/timestamps/context all match
- [ ] 04 Practice `-i`, `-n`, `-u`, `-v` safety flags
- [ ] 05 Work the trailing-slash and `src/.` rules until layout is predictable
- [ ] 06 Execute the RHCSA capstone — `cp -a /etc/ssh /root/ssh-backup` and verify metadata

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `cp -r` for backups | Lost metadata | Use `cp -a` |
| Forgot `-r` on a dir copy | `omitting directory` error | Add `-r` (or `-a`) |
| `cp -r src dst` when dst exists | Got `dst/src/...` instead of `dst/...` | Use `cp -r src/. dst/` |
| Plain `cp` of a config under `/etc` | Owner becomes calling user | `cp -a` (and run as root) |
| Symlinks copied as files | Loss of link semantics | `-a` preserves symlinks (`-d`) |
| Hard links became two independent files | Disk doubled | `cp -a` preserves hard links |
| SELinux context lost | Daemon cannot read copy | `cp -a` or `cp --preserve=context` |
| Used `cp -i` in a script | Interactive prompt hangs the script | Use explicit `-n` or `-f` instead |
| Used `cp -u` across machines | Clock skew confuses mtime comparisons | Use `rsync -a --checksum` |
| Used `cp src /target` repeatedly in a loop | Verbose silent overwrites | Add `-v` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Train the reflex: when the task says "backup," "preserve permissions," or "with original ownership," type `cp -a`. Verify with `ls -lZ` on both sides.

**RHCE candidate**
- Ansible: `copy:` with `mode:`, `owner:`, `group:`, and `setype:` reproduces `cp -a` semantics declaratively.

**SRE / Platform interview**
- "Always back up before you change." → `sudo cp -a /etc/CONFIG_FILE /root/CONFIG_FILE.bak-$(date +%F-%H%M)` before any edit. Rollback is a one-line `cp -a` in reverse.

**DevOps**
- Dockerfile `COPY` preserves mode only by default; use `--chown=user:group` and `--chmod` (BuildKit) for parity with `cp -a`.

**AI / MLOps**
- `cp -a` on a 1 TB dataset is faster than `rsync -a` locally; cross-host, `rsync -aP --partial` resumes.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Directory Navigation | You must know the paths before you copy them |
| Lab 06 — Listing & SELinux Contexts | The verification side of `cp -a` |
| Lab 09 — Hard and Soft Links | Behavior of `cp -d` / `cp -a` with links |
| Lab 10 — Moving and Renaming Files | The sibling of `cp`: same inode, new name |
| Lab 11 — Safe Deletion | The undo for accidental copies |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
