---
layout: post
title: "BreachLab: Ghost"
date: 2026-06-15
author: Vincent Altvater
categories: [ctf, writeup, linux]
---
![BreachLab:Ghost header](/assets/images/breachlab/Ghost.PNG)

This is a full walkthrough of the BreachLab Ghost Track wargame series.
Each level builds on core Linux skills used daily in real security work — enumeration, decoding, process inspection, privilege escalation, and more.


*NOTE:* Be mindful of your terminal customizations, BreachLab has Black and Gray text within the instructions for each level and if your terminal is too dark, you may find yourself in the dark.
![terminal showing text color contrast](/assets/images/breachlab/colored_text.PNG)
---

## Level 0 → 1 — First Contact
[!terminal image showing: First Contact](/assets/images/breachlab/First_Contact.png)

**Tools:** `ls`, `cat`

The entry point. A basic directory listing and file read to get things started.

```bash
ls
cat flag
```

**Flag:** `W3lc0m3T0Gh0st`

**Takeaway:** Every engagement starts with enumeration. `ls` and `cat` are the bread and butter on any unfamiliar system.

---

## Level 1 → 2 — Name Game

**Tools:** `cat`, quoting

Files were named to confuse the shell — `-`, `--help`, and a backtick. Passing them directly to `cat` causes the shell to interpret them as flags. The trick is prefixing with `./` and quoting:

```bash
cat './file name'
cat './'-'
cat './'--help''
```

**Flag:** `D4shIsN0tAFl4g`

**Takeaway:** Always quote paths and use `./` when filenames could be interpreted as arguments.

---

## Level 2 → 3 — In the Shadows

**Tools:** `ls -la`

The flag was in a hidden file (dot-prefixed), invisible to plain `ls`. The `-a` flag reveals them:

```bash
ls -la
```

**Flag:** `H1dd3nInSH4dow`

**Takeaway:** Hidden files are a common place to stash credentials, keys, and other sensitive information.

---

## Level 3 → 4 — Access Denied

**Tools:** `ls -la`, `cd`

Navigated to the highest-privilege accessible path and read the flag. Required understanding `rwx` permission bits and which directories can entered.


**Flag:** `P3rm1ss10ns_M4tt3r`

**Takeaway:** File permissions are a fundamental access control mechanism. Misconfigured permissions are among the most common real-world vulnerabilities.

---

## Level 4 → 5 — Signal in the Noise

**Tools:** `ls`, `grep`, `sort`

There are 500 record files in the vault directory. I found that two approaches worked:

```bash
# Find the outlier by file size — record_0334 was smaller than all others
ls -la record_* | sort -k5 -n

# Or search for the anomaly directly -- Note: grep requires you to either
# guess or already know the word or phrase.
# In this instance I had already seen that the flag was attached to this
# CREDENTIALS keyword and with that hindsight it makes it very easy to search for. 
grep -i CREDENTIALS record_*
```

**Flag:** `Gr3p_F1nds_Truth`

**Takeaway:** When facing large datasets, don't read manually — use `grep` and `sort` to find anomalies.

---

## Level 5 → 6 — The Listener

**Tools:** `nmap`, `nc`

`ss` and `netstat` were disabled. Port scanned with `nmap`, then probed each open port manually:

```bash
nmap -p- localhost
nc -v 127.0.0.1 30100   # returned an Authorization Token
nc -v 127.0.0.1 30101   # accepted the token, returned the credential
```

Port 30100 handed out an Authorization Token; submitting it to port 30101 returned the flag.

**Flag:** `P0rts_N3v3r_L13`

**Takeaway:** When standard tools are locked down, we must fall back to primitives. This level really reinforced that knowing how to manually enumerate services is essential
when hardened systems strip convenience tools.

---

## Level 6 → 7 — Ghost in the Machine

**Tools:** `env`, `base64`

Dumped environment variables and spotted base64-encoded values:

```bash
env
echo "M252X0wzNGtzXzN2M3J5dGgxbmc=" | base64 -d
# API_DIGEST decodes to the flag
```

The other encoded variables (`TRACE_SALT`, `RUNTIME_TOKEN`, `CACHE_SEED`) were decoys — `CACHE_SEED` actually decoded to `not_a_real_credential`.

**Flag:** `3nv_L34ks_3v3ryth1ng`

**Takeaway:** Environment variables can leak secrets in real systems. CI pipelines, crash logs, and `ps aux` output routinely expose credentials passed as env vars.

---

## Level 7 → 8 — Lost in Translation

**Tools:** `xxd`, `base64`

`transmission.dat` was a hexdump of base64 data. Used `xxd -r` to reverse the hexdump back to raw bytes, then decoded:

```bash
xxd -r transmission.dat | base64 -d
```

**Flag:** `D3c0d3_0r_D13`

**Takeaway:** Data is often layered. Hex-encoded base64 is a common obfuscation pattern so learning to recognize encoding formats and chain decode tools together is important.

---

## Level 8 → 9 — Something's Running

**Tools:** `ps`, `/proc`

`ss` and `netstat` were disabled. `ps aux` revealed two `ghost8`-owned processes (PIDs 40 and 41) running `level8-daemon.py`. Even though cmdline arguments had been wiped, `/proc` retained the environment:

```bash
ps aux
cat /proc/40/environ | tr '\0' '\n'
cat /proc/41/environ | tr '\0' '\n'
```

**Flag:** `Pr0c_T3lls_4ll`

**Takeaway:** Even when a process wipes its cmdline arguments, `/proc/<pid>/environ` retains environment variables. This is how real-world tools like `pspy` find leaked secrets in running processes.

---

## Level 9 → 10 — Noise Floor

**Tools:** `strings`

Used `strings` to extract printable text from a binary file, filtering out the noise to find the embedded flag:

```bash
strings file.bin
```

**Flag:** `N01s3_Fl00r`

**Takeaway:** Binaries routinely contain embedded plaintext — passwords, URLs, API keys. `strings` is one of the first tools used in malware analysis and binary reverse engineering.

---

## Level 10 → 11 — Binary Strings

**Tools:** `sort`, `uniq`

Used `sort` and `uniq` to deduplicate a data file and surface the unique entry among thousands of duplicates:

```bash
sort data.txt | uniq
```

**Flag:** `Str1ngs_R3v34l`

**Takeaway:** Sorting and reducing multiple instances of something are core data analysis skills.

---

## Level 11 → 12 — Wrapped Three Deep

**Tools:** `file`, `tar`, `gunzip`, `bunzip2`

`data.wrapped` was a nested archive — tar wrapping gzip wrapping bzip2. Used `file` at each step to identify the format, then unwrapped layer by layer:

```bash
file data.wrapped       # POSIX tar archive
tar -xvf data.wrapped   # extract
file core.txt.gz.bz     # bzip2 compressed
bunzip2 core.txt.gz.bz  # extract
gunzip core.txt.gz      # extract
cat core.txt            # flag
```

**Flag:** `Unwr4pp3d_Thr33`

**Takeaway:** Data exfiltration and malware often use layered compression to obscure content. Always `file` before assuming a file type — never trust the extension.

---

## Level 12 → 13 — Key Not Password

**Tools:** `ssh`, `chmod`

Used the provided SSH private key to authenticate as ghost13. SSH refuses keys with overly permissive permissions, so `chmod 600` was required first:

```bash
chmod 600 sshkey.private
ssh -i sshkey.private ghost13@204.168.229.209 -p 2222
cat flag
```

**Flag:** `K3y_N0t_P4ss`

**Takeaway:** Key-based SSH authentication is the standard for every production server on the planet so understanding key permissions and the `-i` flag is non-negotiable.

---

## Level 13 → 14 — Port 30000

**Tools:** `nc`, `printf`

Port 30000 was a plaintext password-trading service. Hand-crafted the conversation using `printf` to send a properly newline-terminated password:

```bash
printf "K3y_N0t_P4ss\n" | nc localhost 30000
# Correct! Next password: N3tc4t_D3l1v3r
```

**Flag:** `N3tc4t_D3l1v3r`

**Takeaway:** Many real services use simple challenge-response protocols over raw TCP. Knowing how to hand-craft conversations with `nc` is a fundamental penetration testing skill.

---

## Level 14 → 15 — TLS, Not Plaintext

**Tools:** `openssl s_client`

Same mechanic as Level 13 but the equivalent service (port 30001) spoke TLS. First identified which ports used TLS by testing each with `openssl s_client` — a successful handshake indicates TLS, `wrong version number` indicates plaintext:

```bash
# Identify TLS ports
for port in 30000 30001 30100 30101 31339 31790 41337; do
  echo "== $port =="
  echo | timeout 2 openssl s_client -connect localhost:$port 2>&1 | head -5
done

# Submit to the TLS port
printf "N3tc4t_D3l1v3r\n" | openssl s_client -connect localhost:30001 -quiet
```

**Flag:** `TLS_0r_N0th1ng`

**Takeaway:** Plaintext protocols are being replaced with TLS everywhere. The `-quiet` flag suppresses the certificate noise and makes `openssl s_client` behave like `nc` for interactive use.

---

## Level 15 → 16 — Port Range

**Tools:** `openssl s_client`

Submitted the current password to the TLS service on port 31790:

```bash
printf "TLS_0r_N0th1ng\n" | openssl s_client -connect localhost:31790 -quiet
```

**Flag:** `P0rt_Sc4nn3d`

**Takeaway:** Methodical port classification (TLS vs plaintext) before interacting saves time and prevents confusion. Know what you're talking to before you talk to it.

---

## Level 16 → 17 — Diff

**Tools:** `diff`

Two password files were present. Used `diff` to find the changed line between them:

```bash
diff passwords.new passwords.old
```

**Flag:** `D1ff_Sp0ts_1t`

**Takeaway:** `diff` is invaluable for spotting changes in config files, password lists, and logs. In incident response, diffing a suspicious file against a known-good baseline is perfect for spotting changes.
---

## Level 17 → 18 — No Shell For You

**Tools:** `ssh`

ghost17's shell was set to `/usr/local/bin/relay17` (a relay, not bash) — interactive sessions were disabled. SSH still allows non-interactive command execution by appending a command directly to the connection:

```bash
ssh ghost17@204.168.229.209 -p 2222 "cat flag"
```

**Flag:** `Sh3ll_D3n13d`

**Takeaway:** Restricted shells prevent interactive use but not non-interactive SSH command execution. This technique is used legitimately in automation and maliciously in post-exploitation.

---

## Level 18 → 19 — Wrong User

**Tools:** `find`, SUID binaries

Searched for setuid binaries across the filesystem:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Found `/usr/local/bin/readback` — owned by `ghost19`, setuid bit set (`-rwsr-x---`), executable by group `ghost18`. Running it executed with ghost19's effective UID, granting access to ghost19-owned files:

```bash
/usr/local/bin/readback
# SU1D_Fl1p
```

**Flag:** `SU1D_Fl1p`

**Takeaway:** Misconfigured SUID binaries are one of the most common local privilege escalation vectors. Attackers run this exact `find` command on every compromised system. Any non-standard binary with the setuid bit is a red flag.

---

## Level 19 → 20 — Your First Script

**Tools:** `nc`, bash loop

Port 30003 handed out one character at a time by index, using a password-and-index protocol. First queried the total count, then looped to reassemble:

```bash
# Check total pieces
printf "SU1D_Fl1p count\n" | nc localhost 30003
# 26

# Reassemble
PASS="SU1D_Fl1p"
result=""
for i in $(seq 0 25); do
  piece=$( (printf "$PASS $i\n"; sleep 0.3) | nc localhost 30003 | tr -d '\n\r ')
  result="$result$piece"
done
echo "$result"
```

**Flag:** `r34ss3mbl3d_p13c3_by_p13c3`

**Takeaway:** Never fetch data manually when a loop can do it. Scripting repetitive protocol interactions is a core skill in both security automation and exploit development.

---

## Level 20 → 21 — Cron Discovery

**Tools:** `cat`, cron inspection, bash loop

Found a root cron job in `/etc/cron.d/ghost-level20`:

```bash
# Check cron.d
cat /etc/cron.d/ghost-level20
# * * * * * root /opt/ghost-cron/job.sh

cat /opt/ghost-cron/job.sh
# cat /etc/ghost-cron-secret > /var/tmp/ghost-cron-output
# sleep 2
# rm -f /var/tmp/ghost-cron-output
```

The script wrote the secret to a world-readable path for 2 seconds before deleting it. So I used a polling loop to catch it in the window:

```bash
while true; do
  if cat /var/tmp/ghost-cron-output 2>/dev/null; then
    break
  fi
  sleep 0.05
done
```

**Flag:** `Cr0n_R34ds`

**Takeaway:** Temporary files in world-readable directories (`/tmp`, `/var/tmp`) are a classic TOCTOU (time-of-check/time-of-use) vulnerability.

---

## Level 21 → 22 — Git Archaeology

**Tools:** `git tag`, `git show`

A git repo was present with a clean `main` branch — `config.txt` showed only a placeholder `SECRET_KEY=${GHOST_SECRET}`. The README hinted to check tags:

```bash
git tag -l
# v0.9-internal

git show v0.9-internal
```

**Flag:** `G1t_H1st0ry`

The tag pointed to a commit with message "temp: hardcode prod secret for debug trace" — the diff showed the real secret hardcoded in `config.txt` before it was sanitised:

```diff
-SECRET_KEY=${GHOST_SECRET}
+SECRET_KEY=G1t_H1st0ry
```

**Takeaway:** Deleting a secret from a file and committing does not erase it from git history. Tags, branches, and the reflog preserve every state a file has ever been in. 

---

## Level 22 · GRADUATION — Three Shards

**Tools:** `strings`, `base64`, SUID binary, `nc`

The final level combined three techniques from across the track. Three files were present: `relic.bin`, `scroll.b64`, and `BRIEFING`.

**Shard 1 — buried in a binary:**

```bash
strings relic.bin | xxd -d 2>/dev/null
# SHARD1:ALPHA_Z3R0 visible in the output
```

**Shard 2 — encoded for transport:**

```bash
cat scroll.b64 | base64 -d
# SHARD2:BR4V0_0N3
```

**Shard 3 — guarded by a SUID helper:**

```bash
find / -perm -4000 -type f 2>/dev/null
# /usr/local/bin/ghost-archivist (owned by root, executable by ghost22 group)

/usr/local/bin/ghost-archivist
# SHARD3:CH4RL13_TW0
```

**Submitting to the gatekeeper:**

The first attempt I used `BR4VO_0N3` (letter O) instead of `BR4V0_0N3` (number 0) — a classic leet-speak trap. Once corrected:

```bash
echo "SHARD1:ALPHA_Z3R0|SHARD2:BR4V0_0N3|SHARD3:CH4RL13_TW0" | nc localhost 31339
# VERIFIED. All three shards accepted.
# GRADUATION FLAG: Gh0st_0p3r4t1v3
```

**Takeaway:** Investigations rarely require one skill in isolation but a multitude of tools and techniques in unison. The graduation level tested whether you could chain `strings`, base64 decoding, and SUID exploitation together.

---

## Bonus — Port 41337

After graduation, a port scan revealed `41337` still open. Connecting manually revealed a hidden message from KAEL (An operative we are tailing throughout this Ghost track):

```bash
nc localhost 41337
```

> *"The machines you trust every day are not what they appear to be. Docker. Kubernetes. GitHub Actions. The real breach starts in the pipeline."*

The bonus message pointed to the next track: **PHANTOM** — 30 levels of Linux privesc, live at [breachlab.org](https://breachlab.org).

**Takeaway:** Always enumerate. The official scope is never the whole picture.

---
**Final Takeaway:** As someone who has only delved into linux over the last 6 months this Ghost wargame was a lot of fun and super informative!
I think wargames like Breachlab: Ghost and Bandit OvertheWire are really helpful with getting new linux users accustomed to a "Curiosity First" mindset that is invaluable to 
Blue and Red teaming.
*Ghost Track completed.*
*You can find this track available at [breachlab.org](https://breachlab.org) — Ghost Track.*
