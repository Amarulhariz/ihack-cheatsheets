# iHack Attack-Defense CTF — Cheat Sheets

Prep notes and cheat sheets for the RE:HACK / iHack UiTM 24-hour Attack-Defense final round.
First A/D CTF — browser pwnbox, docker services, web + binary challenges.

## Contents

| File | What's in it |
|------|--------------|
| [01-master-cheatsheet.md](01-master-cheatsheet.md) | **Start here.** Everything combined: the game, rules, what to learn, all 7 vulns, commands, exploit template. |
| [02-ad-command-cheatsheet.md](02-ad-command-cheatsheet.md) | Attack-defense commands: recon -> attack -> automate (DestructiveFarm) -> monitor. |
| [03-linux-and-defense.md](03-linux-and-defense.md) | Linux basics (running things, files, processes, docker) + the defense/patching section. |
| [04-vulnerability-playbook.md](04-vulnerability-playbook.md) | Each likely vuln as Spot -> Exploit -> Patch, tuned to iHack's web + binary + outdated-app mix. |
| `REHACK_UBUNTU_PWNBOX_FINAL_CHEATSHEET.pdf` | Teammate's full 54-page pwnbox reference (environment, commands, rules, Ghidra, Wireshark). |

## The rules (don't get disqualified)

- No scanning enemies (no nmap / gobuster) — targets are provided
- >= 2 second delay in any attack loop (no flooding)
- Defend by patching code only — no firewall / IP blocking
- Never crash or DoS an enemy service
- All work inside the pwnbox; search only in the pwnbox browser
- No AI during the game (pre-made scripts OK); phone banned during play

## Learn order (2-day crash course)

1. SQL injection (PwnFunction video -> PortSwigger labs)
2. Command injection (PortSwigger)
3. Path traversal (PortSwigger)
4. searchsploit workflow (outdated-app exploits)
5. Buffer overflow / pwn (Ghidra + pwntools) — if time, or split to teammate

## Writeups worth reading

- iHack 2024 semi-final A/D writeup — vicevirus: https://vicevirus.github.io/posts/ihack-2024-semi-final-writeup
- i-HACK 2022 A/D blog — pikaroot: https://pikaroot.github.io/blogs/2022-12-21-iHack2022_blog
- iHack 2022 challenge repo — fareedfauzi: https://github.com/fareedfauzi/UiTM-iHack2022-Qualification
- iHack A/D training guide: https://fareedfauzi.gitbook.io/ctf-training
