# Lab 08: Copying Files and Directories — `cp`, `cp -R`, `cp -a`, `cp --preserve`

- **Series:** linux-ops-mastery — File Operations & Shell Fundamentals
- **Career arcs covered:** RHCSA EX200 (backup/restore, `/etc/skel`, config staging), RHCE EX294 (`ansible.builtin.copy` with `mode:` `owner:` `preserve:`), CKA (`kubectl cp`, ConfigMap staging), RHCA — RH358 (preserving SELinux + ACLs on copies)
- **Prerequisite:** Lab 00 (Ansible control node) + Lab 07 (timestamps, `stat`)
- **Time Estimate:** 35–50 minutes
- **Tasks:** 5 (ADHD 3-1-1 spec — 3 RHCSA + 1 Ansible + 1 Verification capstone)
- **Practice Directory (lab-wide rotation #08):** `/etc/skel`
- **Sandbox:** `/srv/cp-lab` (writable, separate from `/etc/skel` so we never touch real user defaults)
- **Traps rehearsed this lab:** **T08-A** (`cp` resets mtime and SELinux context on the destination unless you use `-a` or `--preserve`) · **T08-B** (`cp -R` does NOT preserve symlinks as symlinks by default — they get followed; `cp -a` does preserve them) · **T08-C** (Ansible's `ansible.builtin.copy` `src:` is RESOLVED ON THE CONTROL NODE — for in-place copies you need `remote_src: true`)

> **This lab's practice directory is: `/etc/skel`** — every task references it for inspiration (it's where new-user defaults live). We **read** `/etc/skel` only; we **write** only inside `/srv/cp-lab`.

---

## 🖥️ LAB HEADER BLOCK — run this FIRST

```bash
echo "🖥️  ENV:   ${ENV:-DECLARE_ME}"
echo "💿  DISK:  $(lsblk 2>/dev/null | awk '$NF=="disk"{print "/dev/"$1}' | paste -sd, -)"
echo "🔐  SE:    $(getenforce 2>/dev/null || echo n/a)"
echo "📦  OS:    $(cat /etc/redhat-release 2>/dev/null || grep PRETTY_NAME /etc/os-release)"
echo "🕒  TIME:  $(date -Is)"
echo "👤  USER:  $(whoami)@$(hostname)"
echo "⚠️  TRAP REMINDERS THIS LAB: T08-A T08-B T08-C"
echo "📁  PRACTICE DIR: /etc/skel"
echo ""
echo "💡 /etc/skel contents (read-only):"
ls -laZ /etc/skel
```

> **STOP — paste header output before running setup.**

---

## 🎯 Objective

Copy single files, copy directory trees, and **preserve everything that matters** — timestamps, owner, group, SELinux context, ACLs, and symlinks. By the end you will:

- Choose between `cp`, `cp -R`, `cp -a`, and `cp --preserve=...` like a senior admin
- Know exactly what `cp -a` preserves (mode, ownership, timestamps, links, contexts, xattrs)
- Replicate the same operations with `ansible.builtin.copy` declaratively
- Verify with `diff -r`, `getfacl`, `ls -lZ` that the copy is byte-identical AND metadata-identical

---

## 🛠️ Setup — run once before Task 1

```bash
sudo mkdir -p /srv/cp-lab/src /srv/cp-lab/dst
sudo mkdir -p /root/rhcsa_journal/lab08

# Build a sample source tree with mixed file types
echo "hello"          | sudo tee /srv/cp-lab/src/file.txt
echo "secret"         | sudo tee /srv/cp-lab/src/secret.txt
sudo chmod 600 /srv/cp-lab/src/secret.txt
sudo mkdir -p /srv/cp-lab/src/sub
echo "deep"           | sudo tee /srv/cp-lab/src/sub/deep.txt
sudo ln -sfn /srv/cp-lab/src/file.txt /srv/cp-lab/src/link-to-file
sudo touch -t 202001151200 /srv/cp-lab/src/file.txt
ls -laR /srv/cp-lab/src/
```

---

## Task 1 — `cp` a Single File, Watch What Changes

**Practice directory this task:** `/etc/skel` (read), `/srv/cp-lab` (write)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab08/task1
date -Is | sudo tee /root/rhcsa_journal/lab08/task1/start.txt
ls -laZ /etc/skel | sudo tee -a /root/rhcsa_journal/lab08/task1/start.txt
echo "exit was: $?"
```

### Purpose

`cp src dst` is the base case. Compare timestamps, mode, and SELinux context BEFORE and AFTER to see what `cp` resets to default versus copies from the source.

### Main Command Block

```bash
# Source state — note mtime is Jan 15, 2020
ls -lZ /srv/cp-lab/src/file.txt
stat -c 'mode=%a mtime=%y owner=%U:%G' /srv/cp-lab/src/file.txt

# Plain cp — destination gets NEW mtime (now), inherits parent's SELinux context
sudo cp /srv/cp-lab/src/file.txt /srv/cp-lab/dst/file.txt
ls -lZ /srv/cp-lab/dst/file.txt
stat -c 'mode=%a mtime=%y owner=%U:%G' /srv/cp-lab/dst/file.txt

# cp -i — interactive (prompts before overwrite)
sudo cp -i /srv/cp-lab/src/file.txt /srv/cp-lab/dst/file.txt   # answer y when prompted

# cp -v — verbose
sudo cp -v /srv/cp-lab/src/file.txt /srv/cp-lab/dst/file2.txt

# cp -n — no-clobber (don't overwrite existing files)
sudo cp -n /srv/cp-lab/src/file.txt /srv/cp-lab/dst/file2.txt   # silent skip

# Capture
{
  echo "=== source ===";    ls -lZ /srv/cp-lab/src/file.txt
  echo "=== plain cp ==="; sudo cp /srv/cp-lab/src/file.txt /srv/cp-lab/dst/file.txt; ls -lZ /srv/cp-lab/dst/file.txt
  echo "=== mtime drift ==="; stat -c 'src=%y' /srv/cp-lab/src/file.txt; stat -c 'dst=%y' /srv/cp-lab/dst/file.txt
} 2>&1 | sudo tee /root/rhcsa_journal/lab08/task1/transcript.txt
```

### Human-Readable Breakdown

Plain `cp src dst` does the following:

| Attribute | What happens |
|---|---|
| Data bytes | Copied exactly |
| Mode | Inherits source's mode, then masked by `umask` for *new* files (RHEL default: 022 → 0644) |
| Owner | Set to **calling user**, NOT source's owner |
| Group | Set to **calling user's primary group** |
| mtime | Set to **now** |
| atime | Set to **now** |
| SELinux context | Set to **parent directory's default context** |
| ACLs | Lost |
| xattrs | Lost |
| Symlinks | **Followed** (you get the file pointed to, not the link) |

This is the source of `T08-A` — `cp` is destructive to metadata by default. The mtime jumps to now, the SELinux context resets, the owner becomes you. That's almost never what you want when staging a real config file.

### Reading It Left to Right

`cp SRC DST`

- `cp` — copy
- `SRC` — source path (file)
- `DST` — destination path (file or directory)

If `DST` is an existing directory, `cp` copies `SRC` into it as `DST/$(basename SRC)`. If `DST` is a file path, `cp` overwrites it (with `-i` it prompts first).

`cp -v SRC DST`

- `-v` — verbose; print `'src' -> 'dst'` for each copy

### The Story

A grader's question: "Copy `/etc/skel/.bashrc` to `/root/backup.bashrc`." If you write `cp /etc/skel/.bashrc /root/backup.bashrc`, that's RHCSA-correct for the **data**, but the timestamps and SELinux context of the destination won't match the source. For most exam questions that's fine; for "preserve the original timestamps," you need `-p` (next task).

### Expected Output

```
$ ls -lZ /srv/cp-lab/src/file.txt
-rw-r--r--. 1 root root unconfined_u:object_r:var_t:s0 6 Jan 15  2020 /srv/cp-lab/src/file.txt

$ sudo cp /srv/cp-lab/src/file.txt /srv/cp-lab/dst/file.txt
$ ls -lZ /srv/cp-lab/dst/file.txt
-rw-r--r--. 1 root root unconfined_u:object_r:var_t:s0 6 May 27 15:01 /srv/cp-lab/dst/file.txt
                                                                ^^^^^^^^^^
                                                                mtime jumped to now
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `cp SRC DST` | Base copy | Data only — metadata reset to defaults |
| `-i` | Interactive (prompt before overwrite) | Safer for ad-hoc shell work |
| `-n` | No-clobber (skip existing) | Idempotent without prompting |
| `-v` | Verbose | Script debugging |
| `-f` | Force (overwrite without prompt) | Pair with `-i` careful |
| `-u` | Update only (skip if dst is newer) | Backup tools |
| `-b` | Backup existing dst | Safety net |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| Plain `cp` | Data only — metadata reset to defaults |
| `-i` vs `-n` | Interactive prompt vs silent skip |
| Default mode | Inherits source's mode, masked by umask for new files |
| Default owner | Calling user, NOT source's owner |
| Default SELinux | Parent directory's context — that's why labels reset |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T08-A** | `cp` resets mtime + SELinux + owner | Use `-a` or `--preserve=` (Task 2) |
| Overwrite-blindness | `cp src dst` silently overwrites existing dst | Use `-i` for interactive, `-n` for skip |

### 🔁 Persistence Check

```bash
test -f /srv/cp-lab/dst/file.txt && echo "file ok"
diff /srv/cp-lab/src/file.txt /srv/cp-lab/dst/file.txt && echo "data identical"
stat -c '%y' /srv/cp-lab/src/file.txt
stat -c '%y' /srv/cp-lab/dst/file.txt   # different
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab08/task1/done.txt > /dev/null <<EOF
lab=08 task=1
when=$(date -Is)
practice_dir=/etc/skel
src_mtime=$(stat -c '%y' /srv/cp-lab/src/file.txt)
dst_mtime=$(stat -c '%y' /srv/cp-lab/dst/file.txt)
mtime_preserved=$([ "$(stat -c '%Y' /srv/cp-lab/src/file.txt)" = "$(stat -c '%Y' /srv/cp-lab/dst/file.txt)" ] && echo yes || echo no)
EOF
cat /root/rhcsa_journal/lab08/task1/done.txt
```

### 🧹 Cleanup

Leave files; Task 2 reuses the tree.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `cp: cannot create regular file: Permission denied` | Use `sudo` (this whole lab uses root-owned `/srv/cp-lab`) |
| `cp -i` doesn't prompt | An alias may have stripped it; `\cp -i` or `/usr/bin/cp -i` |

> **STOP — confirm `mtime_preserved=no` in done.txt (that's correct — plain `cp` does NOT preserve mtime).**

---

## Task 2 — `cp -R` (Recursive) and `cp -a` (Archive — Preserve Everything)

**Practice directory this task:** `/etc/skel` (read), `/srv/cp-lab` (write)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab08/task2
date -Is | sudo tee /root/rhcsa_journal/lab08/task2/start.txt
ls -laR /srv/cp-lab/src/ | sudo tee -a /root/rhcsa_journal/lab08/task2/start.txt
echo "exit was: $?"
```

### Purpose

Copy a directory tree (including subdirs and a symlink) with `cp -R` and observe that mtime + SELinux + symlinks behave differently. Then re-do with `cp -a` and observe that EVERYTHING (mode, owner, mtime, symlinks, SELinux, xattrs) is preserved.

### Main Command Block

```bash
# Clean prior dst
sudo rm -rf /srv/cp-lab/dst-R /srv/cp-lab/dst-a

# cp -R — recursive but does NOT preserve metadata (mtime reset, symlinks followed)
sudo cp -R /srv/cp-lab/src /srv/cp-lab/dst-R
ls -laZR /srv/cp-lab/dst-R | head -15
echo "--- mtime check ---"
stat -c 'src=%y' /srv/cp-lab/src/file.txt
stat -c 'dst-R=%y' /srv/cp-lab/dst-R/file.txt

echo "--- symlink check on -R ---"
ls -l /srv/cp-lab/dst-R/link-to-file        # NOT a symlink in dst-R (because -R follows by default)

# cp -a — archive (preserves: mode, ownership, mtime, symlinks-as-symlinks, SELinux, xattrs)
sudo cp -a /srv/cp-lab/src /srv/cp-lab/dst-a
ls -laZR /srv/cp-lab/dst-a | head -15
echo "--- mtime check ---"
stat -c 'src=%y' /srv/cp-lab/src/file.txt
stat -c 'dst-a=%y' /srv/cp-lab/dst-a/file.txt

echo "--- symlink check on -a ---"
ls -l /srv/cp-lab/dst-a/link-to-file        # IS a symlink

# Capture
{
  echo "=== cp -R mtime ==="
  stat -c 'src=%y' /srv/cp-lab/src/file.txt
  stat -c 'dst-R=%y' /srv/cp-lab/dst-R/file.txt
  echo "=== cp -R symlink ==="; ls -l /srv/cp-lab/dst-R/link-to-file
  echo "=== cp -a mtime ==="
  stat -c 'src=%y' /srv/cp-lab/src/file.txt
  stat -c 'dst-a=%y' /srv/cp-lab/dst-a/file.txt
  echo "=== cp -a symlink ==="; ls -l /srv/cp-lab/dst-a/link-to-file
  echo "=== cp -a SELinux ==="; ls -dZ /srv/cp-lab/src; ls -dZ /srv/cp-lab/dst-a
} 2>&1 | sudo tee /root/rhcsa_journal/lab08/task2/transcript.txt
```

### Human-Readable Breakdown

`cp -R` (capital R) is recursive. It walks subdirectories. But it does NOT preserve metadata — mtime resets, owner becomes you, symlinks get DE-referenced (you get the file pointed to, not the link).

`cp -a` is shorthand for `-dR --preserve=all`. It:

- Preserves mode, ownership, timestamps (`-p`)
- Preserves symlinks as symlinks (`-d`)
- Recurses (`-R`)
- Preserves SELinux context, xattrs, ACLs (`--preserve=all`)

`cp -a` is the **default tool** when staging configuration or doing backups. It's the closest the userspace `cp` gets to a true `rsync -a`.

There's also `cp -p` (preserve mode/ownership/timestamps but NOT symlinks/context/xattrs) and `cp -d` (preserve symlinks, no other preservation). `-a` bundles them all.

### Reading It Left to Right

`cp -R SRC DST`

- `cp` — copy
- `-R` — recursive (also `-r`, same thing on GNU cp)
- `SRC` — source directory
- `DST` — destination

`cp -a SRC DST`

- `cp` — copy
- `-a` — archive = `-dR --preserve=all`

### The Story

A grader: "Restore the contents of `/var/lib/oldservice/` from `/backup/oldservice/` preserving everything." Plain `cp -R` would break the service (wrong owner, wrong SELinux). `cp -a` is the right tool. RHCSA expects you to reach for `-a` when "preserve everything" is in the question.

### Expected Output

```
=== cp -R mtime ===
src=2020-01-15 12:00:00.000000000 -0500
dst-R=2026-05-27 15:01:00.000000000 -0400          <-- reset to now

=== cp -R symlink ===
-rw-r--r--. 1 root root 6 May 27 15:01 /srv/cp-lab/dst-R/link-to-file
                                                    ^^^^^^^^^^^^^^^^^
                                                    NO `l` — symlink was followed (became a file)

=== cp -a mtime ===
src=2020-01-15 12:00:00.000000000 -0500
dst-a=2020-01-15 12:00:00.000000000 -0500          <-- PRESERVED

=== cp -a symlink ===
lrwxrwxrwx. 1 root root 22 Jan 15  2020 /srv/cp-lab/dst-a/link-to-file -> /srv/cp-lab/src/file.txt
^                                                                       ^^^^^^^^^^^^^^^^^^^^^^^^
preserved as symlink                                                    arrow shows the link target
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `-R` / `-r` | Recursive copy | Required for directories |
| `-a` | Archive (= `-dR --preserve=all`) | The "preserve everything" answer |
| `-p` | Preserve mode + ownership + timestamps | A subset of `-a` |
| `-d` | Preserve symlinks as symlinks | Subset of `-a` |
| `-L` | Always follow symlinks | Opposite of `-d` |
| `-P` | Never follow symlinks (default for `-R` on RHEL) | Subset of `-d` |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| `-R` | Recursive; metadata RESET by default |
| `-a` | Archive; preserves mode + owner + mtime + symlinks + SELinux + xattrs |
| `-a` = | `-dR --preserve=all` |
| When to use `-a` | Backups, config staging, exam-grade "preserve everything" |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T08-A** | `cp -R` resets mtime — you wanted backup but the backup looks "new" | Use `cp -a` |
| **T08-B** | `cp -R` follows symlinks by default — your backup has the file's contents in place of the link | Use `cp -a` (which implies `-d`) |
| SELinux drift | `cp -R` resets contexts to parent's default | Use `cp -a` to preserve, or `restorecon -Rv` on destination |

### 🔁 Persistence Check

```bash
test -f /srv/cp-lab/dst-a/file.txt && echo "dst-a/file ok"
test -L /srv/cp-lab/dst-a/link-to-file && echo "dst-a/link is symlink"
test -L /srv/cp-lab/dst-R/link-to-file && echo "dst-R/link is symlink" || echo "dst-R/link is NOT a symlink (followed by -R)"
[ "$(stat -c '%Y' /srv/cp-lab/src/file.txt)" = "$(stat -c '%Y' /srv/cp-lab/dst-a/file.txt)" ] && echo "mtime preserved by -a"
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab08/task2/done.txt > /dev/null <<EOF
lab=08 task=2
when=$(date -Is)
dst_R_symlink_preserved=$(test -L /srv/cp-lab/dst-R/link-to-file && echo yes || echo no)
dst_a_symlink_preserved=$(test -L /srv/cp-lab/dst-a/link-to-file && echo yes || echo no)
dst_a_mtime_match=$([ "$(stat -c '%Y' /srv/cp-lab/src/file.txt)" = "$(stat -c '%Y' /srv/cp-lab/dst-a/file.txt)" ] && echo yes || echo no)
EOF
cat /root/rhcsa_journal/lab08/task2/done.txt
```

### 🧹 Cleanup

Leave both `dst-R` and `dst-a` — Task 3 compares them.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `cp: -R not specified; omitting directory` | You're copying a directory without `-R` — add it |
| Symlink in `dst-a` points at non-existent target | The symlink was already broken in `src/` — `ls -l src/` to verify |

> **STOP — confirm `dst_a_symlink_preserved=yes` and `dst_a_mtime_match=yes` in done.txt before Task 3.**

---

## Task 3 — Fine-Grained Control: `cp --preserve=` and `--no-preserve=`

**Practice directory this task:** `/srv/cp-lab` (write)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab08/task3
date -Is | sudo tee /root/rhcsa_journal/lab08/task3/start.txt
echo "exit was: $?"
```

### Purpose

Pick the exact preservation set you want. Use `--preserve=ATTRIBUTES` to enable only those, and `--no-preserve=ATTRIBUTES` to disable them. Valid attributes: `mode`, `ownership`, `timestamps`, `links`, `context`, `xattr`, `all`.

### Main Command Block

```bash
sudo rm -rf /srv/cp-lab/dst-pref /srv/cp-lab/dst-nopref

# Preserve only mode + timestamps (skip ownership)
sudo cp -R --preserve=mode,timestamps /srv/cp-lab/src /srv/cp-lab/dst-pref
ls -lZ /srv/cp-lab/dst-pref/file.txt
stat -c 'src=%U:%G %a %y' /srv/cp-lab/src/file.txt
stat -c 'pref=%U:%G %a %y' /srv/cp-lab/dst-pref/file.txt

# cp -a but explicitly DROP ownership preservation
sudo cp -a --no-preserve=ownership /srv/cp-lab/src /srv/cp-lab/dst-nopref
ls -lZ /srv/cp-lab/dst-nopref/file.txt
stat -c 'nopref=%U:%G' /srv/cp-lab/dst-nopref/file.txt

# --preserve=context — SELinux only (useful when DAC is fine but you need labels copied)
sudo rm -rf /srv/cp-lab/dst-ctx
sudo cp -R --preserve=context /srv/cp-lab/src /srv/cp-lab/dst-ctx
ls -lZ /srv/cp-lab/dst-ctx/file.txt
ls -lZ /srv/cp-lab/src/file.txt

# Capture
{
  echo "=== --preserve=mode,timestamps ==="
  stat -c 'src=%U:%G %a %y' /srv/cp-lab/src/file.txt
  stat -c 'pref=%U:%G %a %y' /srv/cp-lab/dst-pref/file.txt
  echo "=== cp -a --no-preserve=ownership ==="
  stat -c 'src_owner=%U:%G' /srv/cp-lab/src/file.txt
  stat -c 'nopref_owner=%U:%G' /srv/cp-lab/dst-nopref/file.txt
  echo "=== --preserve=context (SELinux) ==="
  ls -Z /srv/cp-lab/src/file.txt
  ls -Z /srv/cp-lab/dst-ctx/file.txt
} 2>&1 | sudo tee /root/rhcsa_journal/lab08/task3/transcript.txt
```

### Human-Readable Breakdown

`--preserve=ATTRIBUTES` is the *only* way to copy SOME but not ALL metadata. The full attribute list:

| Attribute | What it covers |
|---|---|
| `mode` | Permission bits (rwx + setuid/setgid/sticky) |
| `ownership` | uid + gid |
| `timestamps` | atime + mtime (not ctime — that always updates) |
| `links` | Symlinks as symlinks (= `-d`) |
| `context` | SELinux security context |
| `xattr` | Extended attributes (capabilities, ACLs go through here too) |
| `all` | Everything above (= what `-a` enables) |

`--no-preserve=` is the negation. Useful when you want `-a` minus one specific thing.

### Reading It Left to Right

`cp -R --preserve=mode,timestamps SRC DST`

- `cp -R` — recursive copy
- `--preserve=mode,timestamps` — only preserve permission bits and atime/mtime
- (everything else — owner, symlinks, SELinux — resets to default)

`cp -a --no-preserve=ownership SRC DST`

- `cp -a` — archive (preserve everything by default)
- `--no-preserve=ownership` — turn OFF ownership preservation
- (everything else from `-a` still preserved)

### The Story

A grader: "Copy `/etc/X/conf` to `/var/lib/X/conf-backup` preserving timestamps and SELinux labels but allowing the destination to be owned by the calling user." Plain `-a` would also copy ownership; you need `--preserve=timestamps,context`. The fine-grained switches are exam-grade.

### Expected Output

```
=== --preserve=mode,timestamps ===
src=root:root 644 2020-01-15 12:00:00.000000000 -0500
pref=root:root 644 2020-01-15 12:00:00.000000000 -0500
                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                  mtime + mode preserved; owner happens to be root because we ran with sudo

=== cp -a --no-preserve=ownership ===
src_owner=root:root
nopref_owner=root:root           <-- still root because we ran with sudo and source owner is root

=== --preserve=context (SELinux) ===
unconfined_u:object_r:var_t:s0   /srv/cp-lab/src/file.txt
unconfined_u:object_r:var_t:s0   /srv/cp-lab/dst-ctx/file.txt
                                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                  context preserved
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `--preserve=mode,timestamps` | Only mode + atime + mtime | Subset of `-a` |
| `--preserve=context` | Only SELinux context | Useful when DAC reset is fine |
| `--preserve=xattr` | xattrs (carries ACLs) | Required when source has ACLs |
| `--preserve=all` | Everything (= `-a`) | Explicit version of `-a` |
| `--no-preserve=ownership` | Turn off one attribute | Pair with `-a` for "everything except X" |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| `--preserve=X,Y,Z` | Enumerate exactly what to preserve |
| `--no-preserve=X` | Negation; pair with `-a` |
| Attribute set | `mode`, `ownership`, `timestamps`, `links`, `context`, `xattr`, `all` |
| `cp -a` ≡ | `cp -dR --preserve=all` |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| Default cp loses ACLs | If source has setfacl entries, plain `cp -p` won't copy them | Use `--preserve=xattr` or `-a` |
| `--preserve=context` requires SELinux | On disabled-SELinux systems it's a no-op | Check `getenforce` |

### 🔁 Persistence Check

```bash
diff -r /srv/cp-lab/src /srv/cp-lab/dst-a >/dev/null && echo "dst-a is bit-identical to src"
[ "$(stat -c '%Y' /srv/cp-lab/src/file.txt)" = "$(stat -c '%Y' /srv/cp-lab/dst-pref/file.txt)" ] && echo "mtime preserved on dst-pref"
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab08/task3/done.txt > /dev/null <<EOF
lab=08 task=3
when=$(date -Is)
preserve_demo=mode+timestamps,no-ownership,context
diff_clean=$(diff -r /srv/cp-lab/src /srv/cp-lab/dst-a >/dev/null && echo yes || echo no)
mtime_pref_match=$([ "$(stat -c '%Y' /srv/cp-lab/src/file.txt)" = "$(stat -c '%Y' /srv/cp-lab/dst-pref/file.txt)" ] && echo yes || echo no)
EOF
cat /root/rhcsa_journal/lab08/task3/done.txt
```

### 🧹 Cleanup

Leave the four `dst-*` directories — Task 5 verifies them.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `cp: invalid argument --preserve=foo` | `foo` is not a valid attribute — see Switches Table |
| Context not preserved on RHEL | Confirm SELinux enabled (`getenforce`) — Disabled drops the operation silently |

> **STOP — confirm `diff_clean=yes` in done.txt before Task 4.**

---

## Task 4 — Ansible: `ansible.builtin.copy` with Preserve / Mode / Owner

**Practice directory this task:** `/srv/cp-lab` (sandbox)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab08/task4/playbooks
date -Is | sudo tee /root/rhcsa_journal/lab08/task4/start.txt
ansible --version | head -1 | sudo tee -a /root/rhcsa_journal/lab08/task4/start.txt
echo "exit was: $?"
```

### Purpose

Replicate Tasks 1–3 with `ansible.builtin.copy`. Set explicit `mode:`, `owner:`, `group:`. Use `remote_src: true` because the source is already on the target machine (localhost), not on the control node. Prove idempotence: second run = `changed=0`.

### Main Command Block

Write the playbook:

```bash
sudo tee /root/rhcsa_journal/lab08/task4/playbooks/copy.yml > /dev/null <<'EOF'
---
- name: Lab 08 Task 4 — copy a file via ansible.builtin.copy
  hosts: localhost
  become: true
  gather_facts: false

  vars:
    src_file: /srv/cp-lab/src/file.txt
    dst_file: /srv/cp-lab/dst-ansible/file.txt
    dst_dir: /srv/cp-lab/dst-ansible

  tasks:
    - name: Ensure destination directory exists
      ansible.builtin.file:
        path: "{{ dst_dir }}"
        state: directory
        mode: '0755'

    - name: Copy file with explicit mode + owner + group
      ansible.builtin.copy:
        src: "{{ src_file }}"
        dest: "{{ dst_file }}"
        remote_src: true       # source is on the target, not the control node
        owner: root
        group: root
        mode: '0644'
      register: copy_result

    - name: Show what changed
      ansible.builtin.debug:
        msg:
          - "copy changed: {{ copy_result.changed }}"
          - "dest: {{ copy_result.dest }}"
          - "checksum: {{ copy_result.checksum | default('n/a') }}"
EOF
```

Check-mode first:

```bash
ansible-playbook --check --diff /root/rhcsa_journal/lab08/task4/playbooks/copy.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab08/task4/check.log
```

Apply:

```bash
ansible-playbook /root/rhcsa_journal/lab08/task4/playbooks/copy.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab08/task4/apply.log
```

Idempotence proof:

```bash
ansible-playbook /root/rhcsa_journal/lab08/task4/playbooks/copy.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab08/task4/rerun.log
grep '^localhost' /root/rhcsa_journal/lab08/task4/rerun.log
```

Second run must show `changed=0` on the copy task. If it's `changed=1`, the file is genuinely different — investigate with `diff`.

### Human-Readable Breakdown

`ansible.builtin.copy` is the file-copy module. The most-used arguments:

| Argument | Maps to |
|---|---|
| `src:` | Source path (control node by default — see `remote_src:`) |
| `dest:` | Destination path on the target |
| `mode:`, `owner:`, `group:` | Explicit DAC — set unconditionally on dest |
| `remote_src: true` | Source is on the target, not the control node |
| `preserve: true` | Preserve mode and timestamps from source (like `cp -p`) |
| `force: false` | Don't overwrite if dest exists (like `cp -n`) |
| `backup: true` | Save backup of overwritten file (like `cp -b`) |
| `checksum:` | If you provide a checksum, the module only copies when current dest != checksum |

`remote_src: true` is the critical RHCE-grade switch on localhost-style labs. Without it, Ansible would look for `/srv/cp-lab/src/file.txt` on the **control node**, not on the **target**. Since they are the same machine in this lab, it usually still works — but writing `remote_src: true` is the correct and RHCE-defensible form.

### Reading It Left to Right

```yaml
ansible.builtin.copy:
  src: /srv/cp-lab/src/file.txt
  dest: /srv/cp-lab/dst-ansible/file.txt
  remote_src: true
  owner: root
  group: root
  mode: '0644'
```

- `ansible.builtin.copy:` — FQCN of the copy module
- `src:` — source path (resolved as remote because of `remote_src:`)
- `dest:` — destination path on the target
- `remote_src: true` — "src is already on the target"
- `owner:`, `group:` — DAC
- `mode:` — quoted octal

### The Story

A grader: "On host X, copy `/etc/template.conf` to `/etc/myservice/conf` owned by root:root, mode 0640." The Ansible answer is `ansible.builtin.copy` with `src:`, `dest:`, `owner:`, `group:`, `mode:`, and `remote_src: true` (because the template lives on the target, not on the controller). Second run reports `changed=0` because the file already matches.

### Expected Output

First apply:

```
TASK [Ensure destination directory exists] ***
changed: [localhost]

TASK [Copy file with explicit mode + owner + group] ***
changed: [localhost]

TASK [Show what changed] ***
ok: [localhost] => msg: ["copy changed: True", "dest: /srv/cp-lab/dst-ansible/file.txt", ...]

PLAY RECAP ***
localhost : ok=3 changed=2 unreachable=0 failed=0
```

Idempotence rerun:

```
TASK [Copy file with explicit mode + owner + group] ***
ok: [localhost]                    <-- NOT changed; content + mode + owner all match

PLAY RECAP ***
localhost : ok=3 changed=0 unreachable=0 failed=0
```

### Switches Table

| Switch / Key | Meaning | Why it matters |
|---|---|---|
| `ansible.builtin.copy:` | FQCN of the copy module | RHCE answer for `cp` |
| `src:` | Source path | Control node by default |
| `dest:` | Destination path on target | Always required |
| `remote_src: true` | Source is on the target | Required when src already on target |
| `owner:`, `group:`, `mode:` | DAC, explicit | Set unconditionally |
| `preserve: true` | Preserve mode + timestamps from src | Equivalent to `cp -p` |
| `backup: true` | Save backup of overwritten dest | Safety net |
| `force: false` | Don't overwrite existing | Equivalent to `cp -n` |
| `validate:` | Run a command to validate before final move | Best practice for `sshd_config` etc |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| `ansible.builtin.copy` | RHCE module for file copy + permission set + ownership set |
| `remote_src: true` | "src is on the target, not on the control node" |
| Idempotence | Module checksums dest; second run = changed=0 if content + meta match |
| `preserve:` | Equivalent of `cp -p` (mode + timestamps from source) |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **Wrapping `command: cp` instead of using `ansible.builtin.copy`** | RHCE cardinal sin | Use the module |
| **T08-C** | Forgetting `remote_src: true`, source resolved on control node and not found | Always set `remote_src: true` when the source is on the target |
| `mode: 0644` unquoted | YAML parses as decimal | Quote it: `mode: '0644'` |
| Not setting `validate:` for service configs | Bad sshd_config locks you out | Pass `validate: '/usr/sbin/sshd -tf %s'` on sshd_config copies |

### 🔁 Persistence Check

```bash
test -f /srv/cp-lab/dst-ansible/file.txt && echo "dst-ansible/file.txt ok"
diff /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt && echo "content matches"
stat -c 'mode=%a owner=%U:%G' /srv/cp-lab/dst-ansible/file.txt
grep -c 'changed=0' /root/rhcsa_journal/lab08/task4/rerun.log
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab08/task4/done.txt > /dev/null <<EOF
lab=08 task=4
when=$(date -Is)
playbook=/root/rhcsa_journal/lab08/task4/playbooks/copy.yml
dst_mode=$(stat -c '%a' /srv/cp-lab/dst-ansible/file.txt)
dst_owner=$(stat -c '%U:%G' /srv/cp-lab/dst-ansible/file.txt)
diff_clean=$(diff /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt >/dev/null && echo yes || echo no)
idempotent=$(grep -c 'changed=0' /root/rhcsa_journal/lab08/task4/rerun.log)
EOF
cat /root/rhcsa_journal/lab08/task4/done.txt
```

### 🧹 Cleanup

Leave files; Task 5 verifies them.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `Could not find or access SRC` | `src:` resolved on control node — add `remote_src: true` |
| Second run shows `changed=1` | mode/owner/group probably don't match — `stat` dest manually |
| `Permission denied` writing dest | `become: true` missing |

> **STOP — confirm `idempotent=1` (count of `changed=0` lines) in done.txt before Task 5.**

---

## Task 5 — RHCSA Verification Capstone: Prove the Copy is Byte- and Metadata-Identical

**Practice directory this task:** `/srv/cp-lab` (sandbox)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab08/task5
date -Is | sudo tee /root/rhcsa_journal/lab08/task5/start.txt
echo "exit was: $?"
```

### Purpose

Use **only** RHCSA inspection commands (no `ansible` CLI) to prove:

1. The destination is **byte-identical** to the source (`diff` or `cmp`)
2. The mode + owner + group match the playbook
3. The SELinux context is appropriate for the destination

### Main Command Block

Three+ RHCSA inspection commands:

```bash
# 1) Byte-identical comparison
diff /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt && echo "DIFF_CLEAN"
cmp /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt && echo "CMP_CLEAN"
md5sum /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt

# 2) Metadata comparison
stat -c 'mode=%a owner=%U:%G size=%s' /srv/cp-lab/src/file.txt
stat -c 'mode=%a owner=%U:%G size=%s' /srv/cp-lab/dst-ansible/file.txt

# 3) SELinux context comparison
ls -lZ /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt
matchpathcon /srv/cp-lab/dst-ansible/file.txt

# 4) ACL comparison (RHCSA-grade — usually no ACLs but verify)
getfacl /srv/cp-lab/src/file.txt
getfacl /srv/cp-lab/dst-ansible/file.txt

# 5) Recursive tree comparison (for the cp -a backup)
diff -r /srv/cp-lab/src /srv/cp-lab/dst-a >/dev/null && echo "TREE_DIFF_CLEAN"

# Capture
{
  echo "=== diff src vs dst-ansible ==="
  diff /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt && echo "CLEAN" || echo "DIFFERENT"
  echo "=== md5 ==="
  md5sum /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt
  echo "=== mode+owner ==="
  echo "src: $(stat -c 'mode=%a owner=%U:%G' /srv/cp-lab/src/file.txt)"
  echo "dst: $(stat -c 'mode=%a owner=%U:%G' /srv/cp-lab/dst-ansible/file.txt)"
  echo "=== SELinux ==="; ls -lZ /srv/cp-lab/src/file.txt /srv/cp-lab/dst-ansible/file.txt
  echo "=== tree (cp -a) ==="
  diff -r /srv/cp-lab/src /srv/cp-lab/dst-a >/dev/null 2>&1 && echo "TREE_MATCH" || echo "TREE_DIFFER"
} 2>&1 | sudo tee /root/rhcsa_journal/lab08/task5/evidence.txt
```

### Human-Readable Breakdown

`diff FILE1 FILE2` — line-level diff; clean exit if identical
`cmp FILE1 FILE2` — byte-level binary comparison; clean exit if identical
`md5sum` — checksum compare; same hash = same content
`stat -c FMT` — metadata snapshot
`ls -lZ` — DAC + SELinux
`diff -r DIR1 DIR2` — recursive directory diff (the big one for `cp -a`)

These are the five tools a grader uses to audit a copy operation. Run them yourself first.

### Reading It Left to Right

`diff -r /srv/cp-lab/src /srv/cp-lab/dst-a >/dev/null`

- `diff` — diff tool
- `-r` — recursive
- `/srv/cp-lab/src` — left side
- `/srv/cp-lab/dst-a` — right side
- `>/dev/null` — discard output; just check exit code

`md5sum FILE1 FILE2`

- `md5sum` — MD5 checksum
- Two arguments → prints both checksums, one per line

### The Story

You hand a grader `evidence.txt` and it reads: "diff is clean, md5 matches, mode is identical (644), owner is identical (root:root), SELinux contexts agree, and the recursive tree diff for `cp -a` is also clean." That's the auditor's full report.

### Expected Output

```
=== diff src vs dst-ansible ===
CLEAN

=== md5 ===
b1946ac92492d2347c6235b4d2611184  /srv/cp-lab/src/file.txt
b1946ac92492d2347c6235b4d2611184  /srv/cp-lab/dst-ansible/file.txt

=== mode+owner ===
src: mode=644 owner=root:root
dst: mode=644 owner=root:root

=== SELinux ===
-rw-r--r--. 1 root root unconfined_u:object_r:var_t:s0 6 ... src/file.txt
-rw-r--r--. 1 root root unconfined_u:object_r:var_t:s0 6 ... dst-ansible/file.txt

=== tree (cp -a) ===
TREE_MATCH
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `diff FILE1 FILE2` | Line-level diff | Quick content check |
| `cmp FILE1 FILE2` | Byte-level compare | Binary-safe |
| `md5sum FILE` | Checksum | Network-safe verification |
| `diff -r DIR1 DIR2` | Recursive directory diff | For `cp -a` style copies |
| `ls -lZ` | DAC + SELinux in one row | Auditor primary view |
| `getfacl PATH` | ACL inspection | If source had ACLs |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| Verification triangle | `diff` (content) + `stat` (metadata) + `ls -lZ` (DAC + SELinux) |
| Reboot reasoning | `/srv/` survives reboot (real filesystem); state changes persist |
| Auditor reflex | Always verify with ≥3 RHCSA inspection commands; check content AND meta |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **Trusting `ansible-playbook`'s changed=1 without inspecting state** | Whole point of the verification capstone | Always run `diff` + `stat` + `ls -lZ` |
| Skipping `diff -r` for tree copies | Tree differ in one nested file you don't notice | Always `diff -r` for `cp -a` style backups |

### 🔁 Persistence Check (Reboot Reasoning)

```bash
echo "REBOOT REASONING:"                                                                | sudo tee /root/rhcsa_journal/lab08/task5/reboot.txt
echo "1. /srv/ is a normal filesystem path. Files persist across reboot."              | sudo tee -a /root/rhcsa_journal/lab08/task5/reboot.txt
echo "2. SELinux context survives because policy applies it at boot."                  | sudo tee -a /root/rhcsa_journal/lab08/task5/reboot.txt
echo "3. The Ansible playbook itself persists in /root/rhcsa_journal/."                | sudo tee -a /root/rhcsa_journal/lab08/task5/reboot.txt
test -f /root/rhcsa_journal/lab08/task4/playbooks/copy.yml && echo "playbook persists" | sudo tee -a /root/rhcsa_journal/lab08/task5/reboot.txt
test -f /srv/cp-lab/dst-ansible/file.txt && echo "dst persists"                        | sudo tee -a /root/rhcsa_journal/lab08/task5/reboot.txt
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab08/task5/done.txt > /dev/null <<EOF
lab=08 task=5
when=$(date -Is)
evidence=/root/rhcsa_journal/lab08/task5/evidence.txt
reboot=/root/rhcsa_journal/lab08/task5/reboot.txt
clean_diff=$(grep -c '^CLEAN$' /root/rhcsa_journal/lab08/task5/evidence.txt)
tree_match=$(grep -c '^TREE_MATCH$' /root/rhcsa_journal/lab08/task5/evidence.txt)
status=lab08-complete
EOF
cat /root/rhcsa_journal/lab08/task5/done.txt
```

### 🧹 Cleanup (No Regression)

```bash
# Remove the sandbox entirely
sudo rm -rf /srv/cp-lab
ls -d /srv/cp-lab 2>&1 | grep -q "No such" && echo "sandbox cleaned"

# Journal stays
ls /root/rhcsa_journal/lab08/
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `diff` reports a difference | Re-run Task 4 — `src:` likely pointed at a different file |
| `TREE_DIFFER` | One file in the tree diverged — run `diff -r` without `>/dev/null` to find which |

> **STOP — record `status=lab08-complete` in done.txt. Lab 08 is finished.**

---

## ✅ Lab 08 Complete When

```bash
ls /root/rhcsa_journal/lab08/task{1,2,3,4,5}/done.txt
grep -l 'lab08-complete' /root/rhcsa_journal/lab08/task5/done.txt
test -f /root/rhcsa_journal/lab08/task4/playbooks/copy.yml
grep -c 'CLEAN' /root/rhcsa_journal/lab08/task5/evidence.txt
```

All four checks must succeed. You can `cp` files by hand, choose the right preservation profile, replicate with `ansible.builtin.copy`, and audit with `diff`/`stat`/`ls -lZ`.
