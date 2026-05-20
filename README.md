# Configure SGID and Sticky Bit (RHEL 9)
### RHCSA EX200 Lab | Part of [linux-ops-mastery](https://github.com/kelvintechnical/linux-ops-mastery)
> Set the SGID and sticky bit on `/data/engineers` so new files inherit the `engineers` group and users can only delete their own files.

![RHCSA](https://img.shields.io/badge/RHCSA-EX200-EE0000?style=flat&logo=redhat&logoColor=white)
![Topic](https://img.shields.io/badge/Topic-Permissions-blue)

---

## 📋 Scenario

On **Node1**, configure `/data/engineers` so that:

1. The **SGID bit** is set — new files and subdirectories inherit the `engineers` group automatically.
2. The **sticky bit** is set — users can only delete or rename their own files.

---

## 🎯 Requirements

1. Set the SGID bit on `/data/engineers`.
2. Set the sticky bit on `/data/engineers`.
3. Verify both bits are active.

---

## ✅ Tasks

- Use `chmod g+s` to set SGID
- Use `chmod +t` to set sticky bit
- Verify with `ls -ld`

---

## 📚 Command Decision Map

| Lab Phrase | Question Being Asked | Tool |
|------------|---------------------|------|
| "Inherit the group" | What bit forces group inheritance? | SGID — `chmod g+s` |
| "Only delete their own files" | What bit restricts deletion? | Sticky bit — `chmod +t` |
| "Verify both bits" | How do I see special bits? | `ls -ld` — look for `s` and `t` |

---

## 🧠 Big Concept — Special Permission Bits

Beyond rwx, Linux has three special bits:

| Bit | Octal | Symbol | Effect on directory |
|-----|-------|--------|---------------------|
| SUID | 4 | `s` on owner x | Rarely used on dirs |
| SGID | 2 | `s` on group x | New files inherit the directory's group |
| Sticky | 1 | `t` on others x | Only file owner can delete their own files |

**SGID use case:** Shared team directories. Without SGID, a file created by `tom` inside `/data/engineers` gets `tom`'s primary group. With SGID, it gets `engineers` — so all team members can access it.

**Sticky bit use case:** Shared writable directories like `/tmp`. Without sticky, anyone with write access to the directory can delete anyone else's files. With sticky, you can only delete files you own.

---

## Step 1 — Set SGID bit

```bash
sudo chmod g+s /data/engineers
```

> `g+s` = add (`+`) the `s` bit to the group (`g`) permission slot.

---

## Step 2 — Set sticky bit

```bash
sudo chmod +t /data/engineers
```

> `+t` = add the sticky bit. No user prefix needed — sticky applies to the directory globally.

---

## Step 3 — Verify both bits

```bash
ls -ld /data/engineers
```

**Expected output:**
```
d---rws--T. 2 tom engineers 6 May 20 11:14 /data/engineers
```

**Reading the special characters:**

| Symbol | Position | Meaning |
|--------|----------|---------|
| `s` | group x slot | SGID is set — group inherits on new files |
| `T` | others x slot | Sticky bit set — uppercase T means others have no execute |

> **`t` vs `T`:** Lowercase `t` = sticky bit set AND others have execute. Uppercase `T` = sticky bit set but others have no execute. Since our others permission is `---`, you'll see `T`.

---   

## 🧠 Key Concepts

| Concept | What it means |
|---------|--------------|
| SGID on directory | New files created inside inherit the directory's group |
| Sticky bit | Only the file owner (or root) can delete or rename files |
| `chmod g+s` | Symbolic notation — adds SGID to group slot |
| `chmod +t` | Adds sticky bit |
| `chmod 3070` | Octal equivalent — `3` = SGID(2) + sticky(1), `070` = group rwx |
| `s` in ls output | Special bit set AND execute is also set |
| `S`/`T` in ls output | Special bit set but execute is NOT set |

---

## ⚠️ Pitfalls

- **Forgetting SGID doesn't affect existing files** → only new files created after SGID is set inherit the group
- **Confusing `S` vs `s`** → uppercase means execute is missing; if you see `S` where you expected `s`, the execute bit was removed
- **Sticky bit on files vs directories** → on files it has a different meaning (SUID-like); for RHCSA always apply it to directories

---

## ✅ Lab Checklist

- `chmod g+s /data/engineers` sets SGID ✓
- `chmod +t /data/engineers` sets sticky bit ✓
- `ls -ld` shows `s` in group slot and `T` in others slot ✓

---

## 🔗 Related Labs

- [User & Group Management / Permissions](https://github.com/kelvintechnical/linux-ops-mastery/blob/master/01-system-management/user-group-permissions.md)
- [Full RHCSA/RHCE Study Guide →](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 👤 Author

**Kelvin R. Tobias** — [kelvinintech.com](https://kelvinintech.com) | [GitHub](https://github.com/kelvintechnical) | [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)    
