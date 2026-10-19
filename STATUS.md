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

## Networking

- LAN IP of host: `192.168.1.158` (router hostname `beans`).
  Router: Verizon Fios, admin at `http://192.168.1.1`.
  Credentials are on the router sticker, or use the My Fios app.
- Steam master list has the server registered at the public IP on 15637.
- **Done 2026-10-19:** router port forward UDP Any -> 15637 to 192.168.1.158 (Firewall → Port
  Forwarding, Custom Ports). A2S now answers on `127.0.0.1`, `192.168.1.158` and the public IP
 , so the router does NAT loopback.
- **Done 2026-10-19:** 192.168.1.158 is reserved as a Static DHCP lease for `beans`
  (Advanced → IPv4 Address Distribution → DHCP Leases). The router admin session expires after a
  while; the user has to log in again (password is on the router sticker).
- Optional TODO: dynamic DNS name (DuckDNS/No-IP) so friends keep working if the public IP
  changes. The user has to create the account; then set up an updater on this PC.
- The in-game Join screen connects by IPv4 address or domain only; you can't search by server
  name. LAN players use `192.168.1.158`, outside players use the public IP (port 15637). The
  public IP is dynamic (Fios WAN uses DHCP), so consider a dynamic DNS name.

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
- Staying up: sleep on AC is set to Never, and Docker Desktop's "Start when you sign in" is on
  (`AutoStart` in `%APPDATA%\Docker\settings-store.json`); both done 2026-10-19. Tested with
  `docker desktop restart`: the container came back by itself and A2S answered on all three
  addresses within about a minute. A2S can time out for the first ~30s after `Host_Online`, so
  wait before treating that as a failure. Docker only runs after a Windows sign-in, so a Windows
  Update reboot keeps the server down until someone signs in.
- **Monitor won't wake (OPEN, test pending):** Alienware AW3423DWF (QD-OLED) on DisplayPort,
  NVIDIA RTX 4070 SUPER. After the 30-min display timeout, the screen stays black on mouse or
  keyboard input. The PC does NOT sleep: no Kernel-Power sleep events, and the server stays up.
  The user wants the display timeout kept (OLED burn-in), so "never turn off display" is not a fix.
  - 2026-10-19: monitor firmware updated to M3B107 (Dell driver 892HJ, SHA-256 verified,
    Dell-signed). The user ran the updater and it succeeded. The release notes don't mention wake
    fixes.
  - Wake test after the update: **Win+Ctrl+Shift+B** (resets the graphics driver) woke the screen.
    So the monitor is losing the DisplayPort signal; the PC is fine. Display timeout is back to
    30 min (confirmed 0x708). This is the workaround for now.
  - Possible permanent fixes, not tried yet: confirm whether plain mouse/keyboard input ever
    wakes it now, update the NVIDIA driver, try HDMI or a different DP cable.
  - This session has no desktop-control tool; the user clicks through GUI installers themselves.
    In Git Bash, `powercfg /...` args get mangled (use MSYS_NO_PATHCONV=1), and the `!` prefix
    only works in the Claude Code prompt, not in the user's own PowerShell window.
- Two Chrome browsers are connected to Claude in Chrome: pick "Browser 2" (Windows, this PC).
  The MacBook can't reach the router admin.
- The Docker VM has about 16 GB RAM and 16 CPUs (host has 32 GB). That's enough.
- In Git Bash, `docker exec ... /home/...` paths get mangled. Prefix with `MSYS_NO_PATHCONV=1`.

## Useful commands

```bash
docker compose logs -f            # follow server logs
docker compose up -d              # apply .env changes / start
docker compose down               # stop (world is kept in the named volume)
git fetch upstream && git merge upstream/main   # sync fork
```
