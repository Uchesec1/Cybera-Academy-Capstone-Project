# Cybera-Academy-Capstone-Project
# # SSH Brute Force Investigation

This is my capstone project for my cybersecurity course. I was given three files from a compromised server (a packet capture, a Linux auth log and a Windows event export) and had to work out what the attacker did, when they did it, and what the company should do next.

## What happened

The attacker at 10.0.2.66 scanned the server at 10.0.2.20, found SSH open, and guessed the password for the user `grace` after 40 failed tries. Once in, they read `/etc/shadow` and created a new root account called `backupsvc` so they could get back in later. From the first failed login to the backdoor was about three minutes.

## Files I worked with

- `capstone-evidence.pcap` for the network traffic
- `capstone-auth.log` for the SSH logs from server01
- `capstone-windows-events.csv` for the Windows security events

I used Wireshark for the capture and just read through the logs and CSV directly.

## What I found

**The scan.** Filtering on `tcp.flags.syn==1` shows 10.0.2.66 hitting one machine across a long list of ports in about a second. Filtering on `tcp.flags.syn==1 && tcp.flags.ack==1` shows only three replied: 22 (SSH), 80 (HTTP) and 3389 (RDP).

**The brute force.** The auth log has 40 failed password lines for `grace`, all from 10.0.2.66, one every 2 to 5 seconds from 10:14:00 to 10:16:16. That rhythm looks like a tool, not a person. Four seconds after the last failure there is an `Accepted password for grace from 10.0.2.66` at 10:16:20.

**Same attacker everywhere.** The same IP shows up in the pcap, the auth log and the Windows events, so one attacker did the scan and the login attempts.

**After the login.** At 10:16:51 `grace` used `sudo` to read `/etc/shadow`, which gives the attacker password hashes to crack offline. At 10:17:11 a new account `backupsvc` was created with UID 0, which is root. The Windows log backs this up with failed logons (4625), the successful logon (4624), the account creation (4720) and the Administrators group change (4732). The timestamps between the Linux and Windows logs are a few seconds apart.

## Timeline

| Time (Oct 6) | What happened |
| --- | --- |
| Start of capture | SYN scan finds ports 22, 80 and 3389 open |
| 10:14:00 | First failed SSH login for grace |
| 10:16:16 | Last of 40 failed attempts |
| 10:16:20 | Successful login |
| 10:16:51 | /etc/shadow read with sudo |
| 10:17:11 | backupsvc created with UID 0 |

## Kill chain

I mapped it to reconnaissance (the scan), exploitation (the brute force and login), actions on objectives (stealing the shadow file) and installation (the backupsvc account).

## What I recommended

1. Turn on multi factor authentication for SSH, so a guessed password alone is not enough.
2. Enforce strong passwords. A password that falls in 40 guesses is weak or common.
3. Block an IP after a handful of failed logins. Five failures would have stopped this attacker long before they got in.

The most urgent job is disabling `backupsvc`. It does not depend on grace's password, so resetting her password does nothing to remove the attacker. After that, reset any credentials that could have been exposed and check the server for other changes.

## Repo layout

```
README.md
report/        the full write up
evidence/      the three evidence files
screenshots/   Wireshark and log screenshots
```

All the data is from a lab exercise, so none of these IPs or accounts are real.
