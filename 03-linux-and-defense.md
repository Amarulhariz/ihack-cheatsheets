# Linux & Defense Cheat Sheet (Ubuntu / Kali)

Everyday commands to operate the pwnbox and defend your services. Pair this with the A/D Command Cheat Sheet.

## Running things
```
./script.sh            # run a script (needs execute permission)
bash script.sh         # run with bash (no +x needed)
python3 file.py        # run a python file
chmod +x script.sh     # make a file executable
./exploit.py 10.0.0.5  # run your exploit with an argument
```

## Editing files (nano / vim)
```
nano file     # easy: Ctrl+O save, Ctrl+X exit
vim file      # i = insert, Esc then :wq = save+quit, :q! = quit no save
```

## Navigating & files
```
pwd                    # where am I
ls -la                 # list all, with details
cd /path               # change dir   (cd .. = up,  cd ~ = home)
cat file               # print a file
less file              # scroll a file (q to quit)
head -n 20 file        # first 20 lines
tail -n 20 file        # last 20 lines
tail -f file           # follow a file live (logs!)
cp a b                 # copy
mv a b                 # move / rename
rm file                # delete   (rm -r dir = delete folder)
mkdir dir              # make a folder
```

## Finding things (vulns, flags, code)
```
grep -r "password" .            # search text in all files under here
grep -rn "os.system" .          # find a dangerous call + line numbers
grep -ri "flag" .               # case-insensitive search
find / -name "*.py" 2>/dev/null # find files by name
find . -type f -newermt "-1 hour"  # files changed in last hour (spot tampering)
```

## Installing tools
```
sudo apt update && sudo apt install <pkg>        # Ubuntu/Kali packages
pip3 install requests --break-system-packages    # python libraries
which python3 curl git                           # check if a tool exists
```

## Processes
```
ps aux                 # all running processes
ps aux | grep python   # find a specific one
top    /   htop        # live process monitor (q to quit)
kill <pid>             # stop a process
kill -9 <pid>          # force stop
pkill -f exploit.py    # kill by name
<cmd> &                # run in background
nohup <cmd> &          # keep running after logout
jobs / fg / bg         # manage background jobs
```

## tmux (run things that must stay up)
```
tmux new -s work       # new named session
Ctrl+b d               # detach (keeps running)
tmux attach -t work    # reattach
Ctrl+b "   /  Ctrl+b % # split panes
```

## Network quick checks (your own box)
```
ip a                   # your IP addresses
ss -tlnp               # listening ports + process
ping <ip>              # reachability
curl http://ip:port/   # hit a web service
```

## Docker (your services live here)
```
docker ps              # running containers + ports
docker logs <name>     # view a service's logs
docker logs -f <name>  # follow logs live
docker exec -it <name> bash    # shell inside
docker restart <name>  # restart after patching
docker cp <name>:/path ./here  # copy files out
docker compose up -d --build   # rebuild from compose
```

---

# DEFENDING

## The golden rule

Keep the service **running** while you fix it. A service that's down scores **zero** (checker fails = lost SLA). Patch the bug, don't break the feature.

## Step 1 — back up before you touch anything
```
cp vuln_file.py vuln_file.py.bak    # so you can revert if a patch breaks it
```

## Step 2 — find the vuln in your own code

Grep for the usual dangerous patterns:
```
grep -rnE "os.system|subprocess|eval|exec" .   # command injection
grep -rnE "SELECT|%s|format\(" .               # SQL built from strings
grep -rnE "open\(|\.\./|send_file" .           # path traversal
grep -rnE "pickle.loads|yaml.load" .           # unsafe deserialization
```

## Step 3 — patch by vuln type

**SQL injection** — use parameters, never string-building:
```
# BAD:  query = "SELECT * FROM u WHERE name='" + name + "'"
# GOOD: cursor.execute("SELECT * FROM u WHERE name=%s", (name,))
```

**Command injection** — no shell, pass arguments as a list:
```
# BAD:  os.system("ping " + host)
# GOOD: subprocess.run(["ping", "-c", "1", host])   # no shell=True
```

**Path traversal** — block ../ and pin to a safe folder:
```
# GOOD: safe = os.path.normpath(os.path.join(BASE, name))
#       if not safe.startswith(BASE): abort(403)
```

**Unsafe deserialization** — don't unpickle untrusted data; use json instead.

## Step 4 — reload and verify
```
docker restart <name>          # load the fix
curl http://localhost:PORT/    # test it still works like normal
```

If the checker would do X, you do X — make sure it still passes.

## Allowed hardening (code / app level only)

- Change default or weak passwords in the app and database
- Remove debug or admin endpoints you don't need
- Keep `.bak` copies so you can revert a bad patch fast

## NOT allowed (will get you disqualified)

- Firewall rules / IP blocking / dropping enemy connections
- Shutting the service off to "protect" it (= zero SLA)
- Scanning or attacking the competition infrastructure

## Watch for incoming attacks
```
docker logs -f <name>                # watch requests hit your service
sudo tcpdump -i any -A port PORT     # see raw attack traffic
```

See how they broke in -> patch that hole -> optionally reuse their exploit against others.
