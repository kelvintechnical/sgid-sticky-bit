# Lab: Configure SGID and Sticky Bit — Shared Directories Done Safely

- **Series:** linux-ops-mastery — RHCSA Permissions, Special Bits & ACLs
- **Subjects covered:** Set-GID on directories (inherit group ownership of new entries), sticky bit on world-writable directories (`+t`), `chmod g+s`, `chmod +t`, `chmod 2770` / `1777`, interpreting `ls -ld` (`s` in group triplet, `t` at end), collaboration patterns without world-readable data
- **Career arcs covered:** RHCSA (special permission bits on directories), RHCE (playbook `mode: '2775'`), SRE (`/tmp` incident analysis), DevOps (multi-tenant build drop directories), AI/MLOps (shared scratch volumes for batch jobs)
- **Prerequisite:** Labs 40–41 — standard modes and `chown`/`chgrp` are comfortable
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 baseline shared dir · 2 enable SGID and observe group inheritance · 3 break inheritance without SGID (contrast) · 4 sticky bit on public write dir · 5 numeric `2770` / `1777` fluency · 6 capstone drop-box + cleanup

---

## Objective

Master the two directory bits that make **team folders** and **public temporary directories** behave on multi-user Linux: **SGID** forces new files to inherit the directory's group (so `umask` collaboration works), and the **sticky bit** lets everyone create files in a shared directory while only allowing each user to delete **their own** files (`/tmp` semantics).

The capstone is an exam-style scenario: *"Create `/tmp/sgid-lab/incoming` mode `2770` owned by `root:writers`, SGID set, containing a submission from `devuser` whose group is automatically `writers`. Create `/tmp/sgid-lab/scratch` mode `1777` with sticky so users cannot erase each other's drop files."*

> **Lab safety note:** All work stays under `/tmp/sgid-lab`. You will add a throwaway user `devuser` if policy permits; otherwise substitute any two unprivileged accounts.

---

## Concept: SGID and Sticky Change *Directory* Semantics

```
   SGID on directory (chmod g+s, shown as 's' in group execute slot)
   ───────────────────────────────────────────────────────────────
   New inode created inside dir  →  st_gid := directory's group
                                    (instead of creator's primary gid)

   Sticky on directory (chmod +t, final char 't' or 'T')
   ─────────────────────────────────────────────────────
   World-writable dir  +  sticky  →  unlink/rename only if:
        you own the file, OR you own the dir, OR you are root
```

Without SGID, Bob's `umask 027` files in a `finance` group folder might still land as `bob:bob`, breaking shared access. With SGID, they land `bob:finance` automatically.

> **Why this matters:** RHCSA loves `/tmp`-style prompts and "collaboration directory with fixed group." SGID is the standard Unix answer before ACL complexity. Sticky is why `/tmp` is not chaos.

---

## 📜 Why SGID and Sticky Exist — The Story

Early Unix machines hosted many researchers on one disk. Shared project directories needed every new file to **belong to the project group** automatically — otherwise each `chmod`/`chgrp` became a manual ritual. The SGID-on-directory behavior encoded that policy in the filesystem itself.

The sticky bit's directory form became famous with **`/tmp`**: every user must write transient files, but users must not delete competitors' work or metadata races. Today `/tmp` and `var/tmp` still ship `1777` (`rwxrwxrwt`) on RHEL.

Modern systems add SELinux, ACL default entries, and richer IAM — yet these two bits remain the **fastest** way to express "shared group workspace" and "world drop box" in one `chmod` call.

> **The point of the story:** When you see `drwxrws---` you should think **collaborative group workspace**. When you see `...rwt` think **`/tmp` policy**.

---

## 👪 The Special Bits on Directories — Who Lives There

| Bit | Octal digit | `chmod` | Directory effect |
|---|---|---|---|
| SUID | 4 | `u+s` | ignored for dirs on Linux |
| SGID | 2 | `g+s` | new files inherit directory group |
| Sticky | 1 | `+t` | extra unlink rules in world-writable dirs |

### Companion commands

| Command | Role |
|---|---|
| `ls -ld DIR` | View permissions on directory inode itself |
| `chmod 2770 DIR` | `rwxrws---` pattern |
| `chmod 1777 DIR` | sticky + world read/write/exec (`/tmp`) |
| `chown root:grp DIR` | Pair SGID with purposeful group |

> **The point of the family tree:** Combine **correct owning group** + **SGID** + **sane `chmod`** — any one missing breaks the collaboration story.

---

## 🔬 The Anatomy of `ls -ld` for SGID + Sticky — In One Diagram

```
drwxrws---. 2 root writers 4096 May 26 12:00 incoming
│      ││
│      │└ lowercase 's' → SGID ON and group execute ON
│      └── group triplet rwx
└ d = directory

drwxrwxrwt. 14 root root 4096 May 26 12:00 scratch
        ││
        │└ lowercase 't' → sticky ON and other execute ON
        └── world rwx (typical /tmp style)
```

> **Reading rule:** Capital `S` or `T` means the special bit is present **without** the underlying execute bit — uncommon on directories but possible if mis-set.

---

## 📚 SGID / Sticky Reference Table

| Task | Command | Notes |
|---|---|---|
| Enable SGID | `chmod g+s DIR` | Same as `chmod 2xxx` prefix on top of base |
| Enable sticky | `chmod +t DIR` | Same as leading `1` in octal |
| Combined example | `chmod 2770 DIR` | SGID + `rwxrwx---` |
| Public drop box | `chmod 1777 DIR` | Sticky + `rwxrwxrwx` |
| Remove SGID | `chmod g-s DIR` | Subtract special bit |
| Verify | `ls -ld DIR` | Visual `s` / `t` confirmation |
| Inherit test | `touch DIR/file; ls -l DIR/file` | Group column should match dir group when SGID set |

> **Rule one of SGID labs:** Set the directory's group (`chgrp writers DIR`) **before** relying on inheritance — SGID copies the directory group, not a guessed name.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Special bits appear explicitly; you must set **and verify** with `ls -ld`. |
| **RHCE candidate** | `mode: '02770'` strings encode SGID in automation. |
| **SRE / Platform** | Mis-set `/tmp` permissions are a recurring compliance finding — know `1777`. |
| **DevOps** | CI artifact directories often need SGID so build users share a group. |
| **AI / MLOps** | Shared NFS scratch: SGID + correct `umask` prevents accidental `others` leakage. |

---

## 🔧 The 6 Tasks

> Six phases: **baseline → SGID on → contrast off → sticky behavior → octal speed → capstone**.

---

### Task 1 — Sandbox, groups, and a naive shared directory

**Purpose:** Create `/tmp/sgid-lab`, a group `writers`, and a directory owned by `root:writers` **without** SGID yet.

```bash
sudo -i
mkdir -p /tmp/sgid-lab
cd /tmp/sgid-lab

groupadd -f writers 2>/dev/null || true
useradd -G writers -m devuser 2>/dev/null || true

install -d -m 0770 -o root -g writers naive
ls -ld naive
```

**Human-Readable Breakdown:** Prepare group membership, create a `0770` directory so only root and `writers` members traverse it.

**Reading it left to right:** `install -d` creates directory with explicit mode/owner/group atomically.

**The story:** This directory *looks* ready for collaboration — but new files may still get the creator's primary group until SGID lands in Task 2.

**Expected output:**

```text
drwxrwx---. 2 root writers 4096 May 26 12:05 naive
```

**Switches**

| Token | Meaning |
|---|---|
| `groupadd -f` | Idempotent group |
| `useradd -G writers` | Supplementary group membership |
| `install -d -m MODE -o -g` | mkdir + chmod + chown combined |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `devuser` lacks group | `usermod -aG writers devuser` then re-login |
| Cannot `cd naive` as devuser | Directory mode or group membership wrong |

---

### Task 2 — Core operation A: enable SGID and watch group inheritance

**Purpose:** `chmod g+s` on the collaborative directory and prove new files pick up `writers`.

```bash
cd /tmp/sgid-lab

chown root:writers naive
chmod 2770 naive
ls -ld naive

sudo -u devuser bash -lc 'cd /tmp/sgid-lab/naive && umask 007 && echo team > from-dev.txt && ls -l from-dev.txt'
```

**Human-Readable Breakdown:** Set SGID via `2770` (`2` prefix), then as `devuser` create a file and inspect group column.

**Reading it left to right:** Leading octal `2` turns on setgid; combined with `770` base yields `rwxrws---` display.

**The story:** This is the canonical **engineering share** — everyone in `writers` sees `rw` on new files when paired with `umask 007`.

**Expected output:**

```text
drwxrws---. 2 root writers 4096 May 26 12:08 naive
-rw-rw----. 1 devuser writers ... from-dev.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `chmod 2770` | SGID + `rwxrwx---` |
| `umask 007` | Files default `660`, dirs `770` for group collab |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| File still `devuser devuser` | SGID not set — `ls -ld` missing `s` |
| Permission denied creating file | Missing group write on directory |

---

### Task 3 — Core operation B: remove SGID and contrast behavior

**Purpose:** Demonstrate regression — same user, same `umask`, different inherited group.

```bash
cd /tmp/sgid-lab

cp -a naive no-sgid
chmod 0770 no-sgid
chmod g-s no-sgid
ls -ld no-sgid

sudo -u devuser bash -lc 'cd /tmp/sgid-lab/no-sgid && umask 007 && echo solo > solo.txt && ls -l solo.txt'
```

**Human-Readable Breakdown:** Duplicate tree, strip SGID (`g-s` or `0770` without `2` prefix), create another file, observe group reverts toward creator primary group.

**Reading it left to right:** `chmod g-s` clears only SGID bit while leaving base mode — verify with `ls -ld`.

**The story:** Removing SGID is how a "working" share silently rots — new files stop being group-readable.

**Expected output:**

```text
drwxrwx---. 2 root writers 4096 May 26 12:10 no-sgid
-rw-rw----. 1 devuser devuser ... solo.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `chmod g-s` | Clear SGID |
| `cp -a` | Archive copy preserving metadata |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Still inherits writers | Directory still has SGID — re-run `ls -ld` |

---

### Task 4 — Verification: sticky bit prevents cross-user deletes

**Purpose:** Build `scratch` with `1777`, create files as two identities, attempt delete.

```bash
cd /tmp/sgid-lab

install -d -m 1777 scratch
ls -ld scratch

sudo -u devuser bash -lc 'echo a > /tmp/sgid-lab/scratch/dev.file'
sudo -u nobody bash -lc 'echo b > /tmp/sgid-lab/scratch/other.file' 2>/dev/null || sudo -u nfsnobody bash -lc 'echo b > /tmp/sgid-lab/scratch/other.file'

sudo -u devuser rm -f /tmp/sgid-lab/scratch/other.file 2>&1 | head -n 1
ls -l /tmp/sgid-lab/scratch
```

**Human-Readable Breakdown:** World-writable sticky dir; each user writes a marker; `devuser` tries to delete `other.file` and should be denied while still able to delete own file.

**Reading it left to right:** Sticky enforces per-file ownership for unlink in world-writable dirs.

**The story:** This is `/tmp` civility — without sticky, any user could unlink any other user's transient payload.

**Expected output:**

```text
drwxrwxrwt. 2 root root 4096 May 26 12:12 scratch
rm: cannot remove '/tmp/sgid-lab/scratch/other.file': Operation not permitted
```

**Switches**

| Token | Meaning |
|---|---|
| `chmod 1777` / `install -m 1777` | sticky + world `rwx` |
| `rm -f` | Attempt unlink |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Delete succeeded | Sticky not set — `ls -ld` should show `t` |
| `nobody` missing | Use any second unprivileged account |

---

### Task 5 — Edge case: numeric fluency — decode modes quickly

**Purpose:** Translate between octal and `ls` symbols for mixed bits.

```bash
cd /tmp/sgid-lab

stat -c '%a %A' naive scratch
python3 - <<'PY'
print(oct(0o2000 | 0o770))  # SGID + rwxrwx---
print(oct(0o1000 | 0o777))  # sticky + rwxrwxrwx
PY
```

**Human-Readable Breakdown:** Use `stat` for octal and human view; optional Python prints expected composites.

**Reading it left to right:** `2770` = `0o2000 | 0o770`; `1777` = `0o1000 | 0o777`.

**The story:** Exam pressure rewards instant decoding — practice until `2770` is muscle memory for "SGID group share."

**Expected output:**

```text
2770 drwxrws---
1777 drwxrwxrwt
0o2770
0o1777
```

**Switches**

| Token | Meaning |
|---|---|
| `stat -c '%a'` | Octal mode without type char |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Python missing | `dnf install python3` or mental math only |

---

### Task 6 — Capstone + cleanup

**Task statement:** *"Ensure `/tmp/sgid-lab/incoming` exists with mode `2770`, owner `root:writers`, SGID set. As `devuser`, create `submit.txt`. Ensure `/tmp/sgid-lab/scratch` is `1777` with sticky. Verify `submit.txt` group is `writers` and cross-delete is denied in `scratch`."*

**Purpose:** Single pass combining SGID collaboration and sticky protections.

```bash
sudo -i
rm -rf /tmp/sgid-lab && mkdir -p /tmp/sgid-lab && cd /tmp/sgid-lab
groupadd -f writers 2>/dev/null || true
useradd -G writers -m devuser 2>/dev/null || true

install -d -m 2770 -o root -g writers incoming
install -d -m 1777 scratch
ls -ld incoming scratch

sudo -u devuser bash -lc 'umask 007; echo capstone > /tmp/sgid-lab/incoming/submit.txt; ls -l /tmp/sgid-lab/incoming/submit.txt'

sudo -u nobody bash -lc 'echo drop > /tmp/sgid-lab/scratch/nobody.txt' 2>/dev/null || true
sudo -u devuser rm -f /tmp/sgid-lab/scratch/nobody.txt 2>&1 | head -n 1
```

**Cleanup**

```bash
sudo -i
rm -rf /tmp/sgid-lab
userdel -r devuser 2>/dev/null || true
groupdel writers 2>/dev/null || true
exit
```

**Expected output:**

```text
drwxrws---. 2 root writers ... incoming
drwxrwxrwt. 2 root root ... scratch
-rw-rw----. 1 devuser writers ... submit.txt
rm: cannot remove '/tmp/sgid-lab/scratch/nobody.txt': Operation not permitted
```

**Switches**

| Token | Meaning |
|---|---|
| `install -d -m 2770` | mkdir + SGID share preset |
| `install -d -m 1777` | mkdir + sticky world tmp |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `incoming` not `2770` | Re-run `chmod 2770` after ownership set |
| Cross-delete works | Confirm sticky with `ls -ld scratch` |

---

## 🔍 SGID / Sticky Decision Guide

```
Need a shared workspace for a UNIX group?
  │
  ├── "Everyone in finance should see new files"
  │       └── chown root:finance DIR && chmod 2770 DIR
  │
  ├── "World can drop files but not erase rivals"
  │       └── chmod 1777 DIR (sticky + rwxrwxrwx)
  │
  ├── "SGID set but files still wrong group"
  │       └── Directory group wrong — chgrp first
  │
  └── "SGID not enough (per-user ACLs needed)"
          └── default ACLs (separate lab) / SELinux types
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Create baseline `0770` `naive/` without SGID
- [ ] 02 Enable SGID (`2770`) and verify inherited group on new file
- [ ] 03 Strip SGID and show regression
- [ ] 04 Demonstrate sticky protection in `1777 scratch/`
- [ ] 05 Practice octal decoding with `stat` / optional Python
- [ ] 06 Run combined capstone and cleanup users/groups

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| SGID without group write | Users cannot create files | Add `g+w` on directory |
| `chmod 770` without `2xxx` | No inheritance | Use `2770` or `g+s` |
| Sticky without world write | Rare layout | Match `/tmp` pattern deliberately |
| Expect SGID on files to inherit groups | Files behave differently | focus on directory semantics |
| ACL + SGID confusion | unexpected masks | inspect `getfacl` separately |
| Cleanup skipped | stray world-writable dirs | Task 6 `rm -rf` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize `/tmp` mode string: **`drwxrwxrwt`** → `1777`.

**RHCE candidate**
- Represent SGID in Ansible with quoted octal: `mode: '02770'` to avoid YAML decimal conversion.

**SRE / Platform interview**
- Explain why package managers warn about world-writable dirs missing sticky — cite unlink races.

**DevOps**
- Map container volume group IDs to host SGID directories for writable multi-pod caches.

**AI / MLOps**
- Shared experiment directories: SGID + `umask 007` is cheaper than per-user ACL churn.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 40 — Standard File Permissions | Base triplets before special bits |
| Lab 41 — Changing Ownership | `chgrp` pairs with SGID |
| Lab 42 — SUID Executables | Other high-order permission bits |
| Lab 44 — Immutable Attribute | Hardening beyond classic bits |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
