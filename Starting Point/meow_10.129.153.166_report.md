# HTB — Meow (10.129.153.166) — Recon & Exploitation Report

**Date:** 2026-09-23
**Target:** 10.129.153.166 (HTB `Meow`, Linux)
**Result:** Full root compromise — no exploitation required (misconfigured telnet + blank root password)
**Root flag:** `b40abdfe23665f766f9c61ecba8a4c19`

---

## 1. Executive Summary

The target is an Ubuntu 20.04.2 LTS host exposing a single network service:
**Telnet (TCP/23)** managed by `xinetd`. The `root` account is permitted to log in over
Telnet **with an empty password**, yielding an immediate unauthenticated-remote root shell.
No exploit development, brute-forcing, or privilege escalation was necessary — the
vulnerability is a configuration flaw.

**Overall risk: Critical.** A remote, unauthenticated attacker obtains complete control of
the host. Telnet also transmits all data (including credentials) in cleartext.

---

## 2. Attack Path

```
Recon (nmap -p-)  →  TCP/23 telnet open
        │
        ▼
Interact with telnet  →  "Meow login:" prompt
        │
        ▼
Login: root / <blank password>
        │
        ▼
Shell as uid=0(root)   →  read /root/flag.txt
```

---

## 3. Reconnaissance

### 3.1 Host Discovery
- ICMP echo reply received — host up, ~188 ms RTT.

### 3.2 Port Scan (`nmap -sV -sC -p- --min-rate 2000 -T4`)

| Port | State | Service | Version |
|------|-------|---------|---------|
| 23/tcp | open | telnet | Linux telnetd |

All 65,534 other TCP ports were closed (RST). Full scan output saved to
`meow_nmap.txt`.

### 3.3 Service Enumeration (Telnet)

- The service did **not** emit a banner until data was sent. Sending a newline
  triggered telnet IAC option negotiation:
  `FF FD 18` (IAC DO TERMINAL-TYPE), `FF FD 20` (IAC DO NAWS),
  `FF FD 23` (IAC DO X-DISPLAY-LOCATION), `FF FD 27` (IAC DO NEW-ENVIRON).
- Refusing these options (WONT/DONT) and then supplying a username revealed the
  login prompt and MOTD.
- MOTD / banner disclosed:
  - Hostname: **Meow**
  - OS: **Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)**
  - IPv4 `10.129.153.166`, IPv6 `dead:beef::a0de:adff:fe5d:aa98`
  - Last successful login from `10.10.14.18` (attacker-style source)

---

## 4. Exploitation

The login banner presented `Meow login:`. Supplying the username `root` with an
**empty password** was accepted.

```text
Meow login: root
Welcome to Ubuntu 20.04.2 LTS ...
root@Meow:~# id
uid=0(root) gid=0(root) groups=0(root)
```

**Finding:** Telnet permits root login with no password — both a
"null/blank password" weakness and a "direct root login over cleartext protocol"
policy failure.

---

## 5. Post-Exploitation Enumeration

| Item | Value |
|------|-------|
| Identity | `uid=0(root) gid=0(root) groups=0(root)` |
| Hostname | `Meow` |
| Kernel | `Linux 5.4.0-77-generic #86-Ubuntu SMP ... x86_64` |
| OS | Ubuntu 20.04.2 LTS (Focal Fossa) |
| Listening sockets | TCP/23 only (`xinetd`, pid 939); UDP/68 (`dhclient`) |
| IPv4 | `10.129.153.166/16` (eth0) |
| IPv6 | `dead:beef::a0de:adff:fe5d:aa98/64`, `fe80::a0de:adff:fe5d:aa98/64` |
| sudoers | `root ALL=(ALL:ALL) ALL`, `%admin`/`%sudo` full rights |
| Notable account | `test-user-0:x:1000:1000::/home/test-user-0:/bin/sh` |

**Flags**
- Root flag (`/root/flag.txt`): `b40abdfe23665f766f9c61ecba8a4c19`
- User flag: none present (`/home` is empty; only `test-user-0` home, no `flag.txt`).

**SUID binaries:** only the standard Ubuntu snap/core set (`sudo`, `mount`, `su`,
`passwd`, `snap-confine`, etc.). No unusual SUID binary present — irrelevant here
since root was already obtained.

---

## 6. Root Cause & Findings

| # | Finding | Severity |
|---|---------|----------|
| 1 | Root login over Telnet with **blank password** | Critical |
| 2 | Telnet exposed to the network — **cleartext** protocol (credentials & session sniffable) | High |
| 3 | **Direct root login** permitted (no `PermitRootLogin`-style restriction) | High |
| 4 | Host missing security updates (75 pending, 31 security) | Medium |

---

## 7. Remediation

1. **Immediately** set a strong root password and disable blank-password accounts
   (`passwd root`; enforce PAM `nullok` removal — the `nullok` option allows empty passwords).
2. **Disable Telnet entirely**; replace with **SSH** (key-based auth, no root login,
   no password auth where possible). Remove the `xinetd` telnet service.
3. Restrict management access to a trusted network / VPN and add host firewall rules.
4. Apply pending OS security updates and enable unattended-upgrades.
5. Audit for other blank/weak accounts (`test-user-0`) and remove unused ones.

---

## 8. Evidence / Artifacts

| File | Contents |
|------|----------|
| `meow_nmap.txt` | Full nmap service/version scan |
| `meow_recon_output.txt` | Raw post-exploitation command output |
| `meow_10.129.153.166_report.md` | This report |

**Flags captured:** `b40abdfe23665f766f9c61ecba8a4c19` (root)
