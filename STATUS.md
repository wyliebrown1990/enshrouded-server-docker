# Session status notes

Handoff notes for the next agent working on this fork. Last updated 2026-10-19.

## Repo

- Fork: `git@github.com:wyliebrown1990/enshrouded-server-docker.git` (`origin`, SSH works)
- Upstream: `https://github.com/indifferentbroccoli/enshrouded-server-docker.git` (`upstream`)
- Local clone: `C:\Users\wylie\enshrouded-server-docker`
- `gh` CLI is not installed (no PR/issue automation; plain git only)

## Deployment (Windows 11 host, Docker Desktop + WSL2)

- Running via `docker compose up -d` from the repo root. Container: `enshrouded`, image
  `indifferentbroccoli/enshrouded-server-docker:latest` (engine: Proton GE).
- Server name: **Wylie Enshrouded Server**, UDP **15637**, max 12 players.
- Password protected: `SERVER_PASSWORD` is set in `.env` (gitignored; never commit it).
  It ends up on the `Default` user group in `enshrouded_server.json`.
- Game files and saves live in the Docker named volume `enshrouded-server-docker_enshrouded-data`,
  set by `docker-compose.override.yml` (auto-loaded by compose).
- The old `server-files\` folder (~7.6 GB) is an abandoned bind-mount copy and is unused.
  The user hasn't decided yet whether to delete it.

### Why the override exists

With the stock bind mount (`./server-files`), the files are served over 9p/drvfs from `C:\`.
Proton crashes about 4s after launch (`NtRaiseException ... c0000005`, exit code 5, no server log),
and the container restart-loops. Moving the files to a named volume fixed it immediately.

### Known upstream issue

The healthcheck (`pgrep -f "enshrouded_server"`) always reports healthy, because it matches its
own `sh -c` command line. Use `docker logs enshrouded` instead. A real check looks for
`[Session] finished transition ... 'Host_Online'`.

## Networking: OPEN ITEM

- LAN IP of host: `192.168.1.158`. Router: Verizon Fios, admin at `http://192.168.1.1`.
  Credentials are on the router sticker, or use the My Fios app.
- Steam master list has the server registered at the public IP on 15637.
- A2S query works on `127.0.0.1` and `192.168.1.158`, but **times out on the public IP**, so the
  server does not show in the in-game browser.
- **TODO:** add a router port forward: UDP 15637 → 192.168.1.158:15637, and reserve that IP in DHCP.
  Then verify with the A2S check below. If it still fails from inside the LAN, the router may not
  support NAT loopback; test from outside.
- The in-game browser searches by server name, not IP.

```bash
# Is the server registered with Steam? (replace with current public IP: curl -s https://api.ipify.org)
curl -s "https://api.steampowered.com/ISteamApps/GetServersAtAddress/v1/?addr=<PUBLIC_IP>"
```

```python
# A2S_INFO probe; run with: python -I a2s.py 127.0.0.1 192.168.1.158 <PUBLIC_IP>
import socket, sys
q = b'\xff\xff\xff\xffTSource Engine Query\x00'
for host in sys.argv[1:]:
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM); s.settimeout(3)
    try:
        s.sendto(q, (host, 15637)); d, _ = s.recvfrom(4096)
        if d[4:5] == b'A': s.sendto(q + d[5:9], (host, 15637)); d, _ = s.recvfrom(4096)
        print(host, 'OK', d[6:60].split(b'\x00')[0])
    except Exception as e: print(host, 'FAIL', e)
```

The router config was going to be done through the Claude in Chrome extension, but it wasn't
connected in the previous session. Check that the `mcp__claude-in-chrome__*` tools are available
before trying.

## Other host notes

- Docker Desktop keeps popping up "WSL integration with distro 'Ubuntu' unexpectedly stopped".
  This is harmless to the server. Fix: Docker Desktop → Settings → Resources → WSL integration →
  turn off Ubuntu (if the user doesn't use `docker` inside Ubuntu).
- The Docker VM has about 16 GB RAM and 16 CPUs (host has 32 GB). That's enough.
- In Git Bash, `docker exec ... /home/...` paths get mangled. Prefix with `MSYS_NO_PATHCONV=1`.

## Useful commands

```bash
docker compose logs -f            # follow server logs
docker compose up -d              # apply .env changes / start
docker compose down               # stop (world is kept in the named volume)
git fetch upstream && git merge upstream/main   # sync fork
```
