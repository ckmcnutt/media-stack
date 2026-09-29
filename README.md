# media-stack

A self-hosted media automation stack run with Docker Compose. Requests come in
through Seerr, Sonarr and Radarr decide what to grab, Prowlarr feeds them
indexers, SABnzbd does the downloading, and Jellyfin serves the result.

| Service  | Role                                   | Web UI                  |
| -------- | -------------------------------------- | ----------------------- |
| Jellyfin | Media server / player                  | http://localhost:8096   |
| Sonarr   | TV series management                   | http://localhost:8989   |
| Radarr   | Movie management                       | http://localhost:7878   |
| Prowlarr | Indexer manager (feeds Sonarr/Radarr)  | http://localhost:9696   |
| SABnzbd  | Usenet downloader                      | http://localhost:8080   |
| Seerr    | Request portal for users               | http://localhost:5055   |

All images are [LinuxServer.io](https://docs.linuxserver.io/) builds except
Seerr, which comes from `ghcr.io/seerr-team/seerr`.

## Contents

- [Requirements](#requirements)
- [The `/data` layout](#the-data-layout)
- [Configuration you will need to change](#configuration-you-will-need-to-change)
  - [Optional: parameterize the paths](#optional-parameterize-the-paths)
- [Setup](#setup)
  - [Linux](#linux)
  - [macOS](#macos)
  - [Windows](#windows)
- [First-run configuration](#first-run-configuration)
  - [1. SABnzbd](#1-sabnzbd-httplocalhost8080)
  - [2. Prowlarr](#2-prowlarr-httplocalhost9696)
  - [3. Sonarr and Radarr](#3-sonarr-httplocalhost8989-and-radarr-httplocalhost7878)
  - [4. Jellyfin](#4-jellyfin-httplocalhost8096)
  - [5. Seerr](#5-seerr-httplocalhost5055)
- [Day-to-day operation](#day-to-day-operation)
  - [Updating](#updating)
  - [Backups](#backups)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)

---

---

## Requirements

- **Docker Engine 20.10+** with the Compose V2 plugin (`docker compose`, not the
  old `docker-compose` script). Verify with `docker compose version`.
- **~2 GB RAM** for the stack itself, plus whatever your library and transcoding
  need. 4 GB is a comfortable floor.
- **Disk on a single filesystem** for downloads and media — see
  [The `/data` layout](#the-data-layout). This is the single most important
  decision in the whole setup.
- A **Usenet provider** account and at least one **indexer** (NZBgeek, DrunkenSlug,
  etc.). This stack has no torrent client; it is Usenet-only as written.

---

## The `/data` layout

Sonarr and Radarr both mount `/data` as a single volume:

```yaml
volumes:
  - /data/docker/sonarr:/config
  - /data:/data
```

That is deliberate. Because SABnzbd's completed downloads and the final media
library live under the *same* mount inside the container, Sonarr and Radarr can
**hardlink** files instead of copying them. An import is instant and uses no
extra disk, and the original stays in place so you can continue seeding or keep
the download history intact.

If you split these into separate mounts (e.g. `/downloads` and `/media`), Docker
presents them as different filesystems, hardlinking silently fails, and every
import becomes a full copy — double disk usage and slow imports. Keep one root.

Create this tree before the first start:

```
/data
├── docker/            # container config + databases (back this up)
│   ├── jellyfin/
│   ├── sonarr/
│   ├── radarr/
│   ├── prowlarr/
│   ├── sabnzbd/
│   └── jellyseerr/    # note: Seerr's config dir kept its old name
├── usenet/            # SABnzbd working directories
│   ├── incomplete/
│   └── complete/
└── media/             # your library — what Jellyfin scans
    ├── tv/
    ├── movies/
    └── music/
```

One-liner:

```bash
mkdir -p /data/docker/{jellyfin,sonarr,radarr,prowlarr,sabnzbd,jellyseerr} \
         /data/usenet/{incomplete,complete} \
         /data/media/{tv,movies,music}
```

> **Why `jellyseerr` and not `seerr`?** The service was renamed from Jellyseerr
> to Seerr upstream, but the config path was left alone so existing installs keep
> their database. If you are starting fresh you can rename both the directory and
> the volume line in `docker-compose.yml` to `seerr`.

---

## Configuration you will need to change

The compose file hardcodes three things. Edit `docker-compose.yml` before your
first `up`:

| Setting          | Current value      | Change it to                                         |
| ---------------- | ------------------ | ---------------------------------------------------- |
| `PUID` / `PGID`  | `1000` / `1000`    | Your own user's IDs (`id -u` / `id -g`) on Linux/Mac  |
| `TZ`             | `America/New_York` | Your [TZ identifier](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) |
| Volume paths     | `/data/...`        | Wherever your data actually lives (Mac/Windows)      |

`PUID`/`PGID` tell the LinuxServer images which user to run as, so the files they
write are owned by you rather than root. Getting them wrong is the usual cause of
permission errors on import.

Seerr also ships with `LOG_LEVEL=debug`, which is noisy. Change it to `info` once
you have the stack working.

### Optional: parameterize the paths

Hand-editing a dozen volume lines per machine gets old. If you plan to run this
on more than one OS, replace the literals with variables:

```yaml
environment:
  - PUID=${PUID:-1000}
  - PGID=${PGID:-1000}
  - TZ=${TZ:-America/New_York}
volumes:
  - ${DATA_ROOT:-/data}/docker/sonarr:/config
  - ${DATA_ROOT:-/data}:/data
```

Then keep a per-machine `.env` next to the compose file (and add it to
`.gitignore`):

```ini
DATA_ROOT=/Users/chris/media-stack/data
PUID=501
PGID=20
TZ=America/New_York
```

Compose reads `.env` automatically. The defaults above mean the file stays
optional on your Linux box.

---

## Setup

Clone the repo first, on any platform:

```bash
git clone <this-repo> media-stack
cd media-stack
```

Then follow the section for your OS.

### Linux

The native case — bind mounts, `PUID`/`PGID` and hardware transcoding all work
as intended.

1. **Install Docker Engine and the Compose plugin.** Use Docker's official
   repository rather than your distro's `docker.io` package, which is often
   several versions behind:

   ```bash
   curl -fsSL https://get.docker.com | sh
   ```

   On Fedora/RHEL, follow the
   [dnf instructions](https://docs.docker.com/engine/install/fedora/) instead.

2. **Let your user run Docker without sudo:**

   ```bash
   sudo usermod -aG docker "$USER"
   newgrp docker   # or log out and back in
   ```

3. **Create the data tree** (see [The `/data` layout](#the-data-layout)) and take
   ownership:

   ```bash
   sudo mkdir -p /data
   sudo chown "$(id -u):$(id -g)" /data
   mkdir -p /data/docker/{jellyfin,sonarr,radarr,prowlarr,sabnzbd,jellyseerr} \
            /data/usenet/{incomplete,complete} \
            /data/media/{tv,movies,music}
   ```

4. **Set `PUID`/`PGID`** in `docker-compose.yml` to the output of `id -u` and
   `id -g`. On most single-user desktop installs these are already `1000`.

5. **Start the stack:**

   ```bash
   docker compose up -d
   docker compose ps
   ```

**SELinux (Fedora, RHEL, Rocky):** bind mounts are blocked by default. Either
append `:z` to the volume options (`- /data:/data:z`) or, if you would rather not
relabel a large media tree, run `sudo setsebool -P container_manage_cgroup on`
and add `:Z` only to the `/config` mounts.

**Mounting a NAS?** Put the *whole* `/data` tree on one share. An NFS or SMB
mount cannot hardlink into a local directory, so straddling the boundary
reintroduces the copy problem. For SMB, mount with
`uid=1000,gid=1000,file_mode=0664,dir_mode=0775` so `PUID`/`PGID` line up.

### macOS

Two things differ from Linux: you cannot create `/data` at the filesystem root,
and file ownership works differently.

1. **Install Docker Desktop** (Apple silicon or Intel build as appropriate):

   ```bash
   brew install --cask docker
   ```

   Launch it once and let it finish initializing. Alternatives like Colima or
   OrbStack work too — the compose file is unchanged, but you will manage file
   sharing through their own config.

2. **Pick a data root inside your home directory.** macOS has blocked writes to
   `/` since Catalina, so `sudo mkdir /data` will fail. Use something like:

   ```bash
   export DATA_ROOT="$HOME/media-stack/data"
   mkdir -p "$DATA_ROOT"/docker/{jellyfin,sonarr,radarr,prowlarr,sabnzbd,jellyseerr} \
            "$DATA_ROOT"/usenet/{incomplete,complete} \
            "$DATA_ROOT"/media/{tv,movies,music}
   ```

   An external drive (`/Volumes/Media/data`) is fine as long as it is formatted
   APFS or HFS+. **Do not use exFAT or NTFS** — neither supports the hardlinks
   Sonarr and Radarr rely on, and NTFS-3G write support on macOS is unreliable.

3. **Rewrite the volume paths.** Either use the
   [`.env` approach](#optional-parameterize-the-paths) or substitute in place:

   ```bash
   sed -i '' "s|- /data|- $DATA_ROOT|g" docker-compose.yml
   ```

   Check the result with `git diff` — the container-side paths (after the `:`)
   must stay as `/data`, `/config`, etc. Only the host side changes.

4. **Add the path to Docker Desktop's file sharing** if it is outside your home
   directory: *Settings → Resources → File sharing → +*. Paths under `$HOME` are
   shared by default.

5. **Set `PUID`/`PGID`.** On macOS your user is typically `501:20`:

   ```bash
   echo "$(id -u):$(id -g)"
   ```

   In practice Docker Desktop's virtiofs layer maps ownership for you and these
   values matter less than on Linux, but setting them correctly costs nothing and
   avoids surprises if you later move the stack to a Linux host.

6. **Start the stack:**

   ```bash
   docker compose up -d
   ```

**Expect slower I/O than Linux.** Bind mounts cross a virtualization boundary, so
large library scans and imports take noticeably longer. Keeping `/config`
directories (the SQLite databases) on the internal SSD rather than an external
drive helps a lot.

**No hardware transcoding.** Docker on macOS cannot pass through the Apple
VideoToolbox encoder, so Jellyfin transcodes on CPU only. Direct play is
unaffected. If transcoding performance matters, run Jellyfin natively on the Mac
and keep only the automation services in Docker.

### Windows

Run the stack inside WSL2 and keep the data there too. This is the difference
between a setup that works and one that is mysteriously slow and broken.

1. **Install WSL2 and a distro** in an admin PowerShell:

   ```powershell
   wsl --install -d Ubuntu
   ```

   Reboot when prompted, then finish the Ubuntu username/password setup.

2. **Install Docker Desktop** and enable the WSL2 backend:
   *Settings → General → Use the WSL 2 based engine*, then
   *Settings → Resources → WSL integration → enable your distro*.

3. **Do the rest of the work from inside WSL.** Open the Ubuntu terminal — not
   PowerShell — and clone the repo into the Linux filesystem:

   ```bash
   cd ~
   git clone <this-repo> media-stack
   cd media-stack
   ```

   > **Do not put the repo or the data under `/mnt/c/`.** Windows drives are
   > exposed to WSL over a 9p network filesystem. It is an order of magnitude
   > slower, and — critically — it does not support hardlinks or Unix ownership,
   > which breaks both the import path and `PUID`/`PGID`. The whole stack lives
   > on the ext4 side.

4. **Create the data tree** exactly as on Linux:

   ```bash
   sudo mkdir -p /data
   sudo chown "$(id -u):$(id -g)" /data
   mkdir -p /data/docker/{jellyfin,sonarr,radarr,prowlarr,sabnzbd,jellyseerr} \
            /data/usenet/{incomplete,complete} \
            /data/media/{tv,movies,music}
   ```

   `/data` here is inside the WSL distro, so no compose edits are needed.
   Your default WSL user is `1000:1000`, matching the compose file as written.

5. **Start the stack** from the WSL shell:

   ```bash
   docker compose up -d
   ```

   The web UIs are reachable from Windows at `http://localhost:8096` and friends —
   WSL2 forwards localhost automatically.

**Where do files actually live?** At `\\wsl$\Ubuntu\data\media` in Explorer. You
can map that as a network drive for convenience. Writing there from Windows is
fine; just don't move the canonical location.

**Storing the library on a big NTFS drive.** If your media is on a D: drive you
cannot avoid, mount it into WSL with metadata support so ownership works:

```bash
sudo mkdir -p /data/media
sudo mount -t drvfs D:\\Media /data/media -o metadata,uid=1000,gid=1000
```

Add it to `/etc/fstab` to persist. Accept that hardlinks between
`/data/usenet` (ext4) and `/data/media` (NTFS) will not work — Sonarr and Radarr
will fall back to copying. If you go this route, put `/data/usenet` on the same
NTFS drive so at least the two share a filesystem.

**Cap WSL2's memory** if Docker starts eating the machine. Create
`C:\Users\<you>\.wslconfig`:

```ini
[wsl2]
memory=6GB
processors=4
```

Then `wsl --shutdown` and restart Docker Desktop.

**Keep it running.** Docker Desktop must be running for the stack to be up. For
an always-on server, enable *Start Docker Desktop when you log in* and set the
machine to auto-login, or — better — run this on a Linux box.

---

## First-run configuration

Order matters here. Each step depends on the one before it.

### 1. SABnzbd (http://localhost:8080)

Run the setup wizard. Enter your Usenet provider's hostname, port (563 for SSL),
username and password, then test the connection.

Under *Settings → Folders*, set the paths to the **container-side** values:

- Temporary Download Folder: `/data/usenet/incomplete`
- Completed Download Folder: `/data/usenet/complete`

Copy your API key from *Settings → General* — you need it twice in a moment.

### 2. Prowlarr (http://localhost:9696)

Create your admin login (*Settings → General → Security*; use *Forms* auth).

Add your indexers under *Indexers → Add Indexer*, entering the API keys from each
provider.

Then wire Prowlarr to Sonarr and Radarr under *Settings → Apps → +*. Prowlarr
pushes indexer definitions to them, so you never configure indexers twice:

- **Prowlarr Server:** `http://prowlarr:9696`
- **Sonarr Server:** `http://sonarr:8989`
- **Radarr Server:** `http://radarr:7878`
- **API Key:** from that app's *Settings → General*

Those short hostnames work because Compose puts all these containers on a shared
project network with DNS. Use `http://localhost:...` here and it will fail —
inside a container, `localhost` is the container itself.

### 3. Sonarr (http://localhost:8989) and Radarr (http://localhost:7878)

In each:

- **Settings → Media Management → Root Folders:** add `/data/media/tv` (Sonarr)
  or `/data/media/movies` (Radarr).
- **Settings → Media Management:** enable *Use Hardlinks instead of Copy*.
- **Settings → Download Clients → + → SABnzbd:**
  - Host: `sabnzbd`
  - Port: `8080`
  - API Key: the one you copied
  - Click *Test* — a green check means the network path is good.

Grab each app's API key from *Settings → General*; Seerr needs them next.

### 4. Jellyfin (http://localhost:8096)

Run the wizard, create your admin account, then add libraries:

| Library type | Container path      |
| ------------ | ------------------- |
| Shows        | `/data/tv`          |
| Movies       | `/data/movies`      |
| Music        | `/data/music`       |

Note these differ from Sonarr's paths. Jellyfin mounts the media
subdirectories individually, so its view is `/data/tv`, not `/data/media/tv`.
Both point at the same host directory.

### 5. Seerr (http://localhost:5055)

Sign in with Jellyfin, then connect the services. **Here you cannot use container
hostnames for Jellyfin** — see the note below. Use your machine's LAN IP:

- **Jellyfin:** `http://192.168.1.x:8096` (find yours with `ip addr` /
  `ipconfig getifaddr en0`)
- **Sonarr:** `http://sonarr:8989` + API key
- **Radarr:** `http://radarr:7878` + API key

Set default quality profiles and root folders for each, then run a library scan.

> **Why the IP for Jellyfin?** Jellyfin is configured with
> `network_mode: bridge`, which attaches it to Docker's legacy default bridge
> instead of this project's network. The default bridge has no DNS, so
> `http://jellyfin:8096` does not resolve from other containers, and Jellyfin
> cannot resolve them either. Reaching it by host IP works because port 8096 is
> published.
>
> If you would rather have DNS everywhere, delete the `network_mode: bridge` line
> from the `jellyfin` service. Jellyfin joins the project network and
> `http://jellyfin:8096` starts working. The one thing you lose is nothing
> meaningful for this setup — the line was likely a leftover. Keep it only if you
> later switch Jellyfin to `network_mode: host` for DLNA discovery, which does
> need the host's broadcast domain.

---

## Day-to-day operation

```bash
docker compose up -d              # start everything
docker compose stop               # stop, keep containers
docker compose down               # stop and remove containers (config is safe)
docker compose ps                 # what is running
docker compose logs -f sonarr     # follow one service's logs
docker compose restart radarr     # bounce one service
docker compose exec sabnzbd bash  # shell into a container
```

Config lives in bind-mounted host directories, so `down` never loses data.

### Updating

```bash
docker compose pull
docker compose up -d
docker image prune -f    # reclaim space from old layers
```

All images are pinned to `:latest`, so `pull` moves you to whatever is current.
That is convenient but means an update can surprise you — check the
[LinuxServer release notes](https://docs.linuxserver.io/) before pulling if the
stack is load-bearing, and back up `/data/docker` first. Pinning to specific
tags (`lscr.io/linuxserver/sonarr:4.0.10`) is the safer choice for a stack you
depend on.

### Backups

Everything worth saving is in `/data/docker` — a few hundred MB of SQLite
databases and settings. Stop the stack first so the databases are consistent:

```bash
docker compose stop
tar czf "media-stack-$(date +%F).tar.gz" -C /data docker
docker compose up -d
```

Restoring is the reverse: stop, extract over `/data/docker`, start.

---

## Troubleshooting

**A container restarts in a loop.** Read the logs first:
`docker compose logs --tail=50 <service>`. For LinuxServer images the cause is
usually a `/config` directory the `PUID`/`PGID` user cannot write to.

**Permission denied on import.** The classic `PUID`/`PGID` mismatch. Compare what
the container thinks it is against what owns the files:

```bash
docker compose exec sonarr id
ls -ln /data/media/tv
```

The UIDs should match. Fix with
`sudo chown -R "$(id -u):$(id -g)" /data` and correct the compose values.

**Port already in use.** `8080` is the frequent offender — it collides with
Jenkins, Tomcat, and a long list of dev servers. Remap the host side only:

```yaml
ports:
  - "8081:8080"   # host:container — change the left number
```

Find the conflict with `sudo lsof -i :8080` (Linux/Mac) or
`netstat -ano | findstr :8080` (Windows).

**Sonarr can't reach SABnzbd.** Use `sabnzbd` as the host, not `localhost` or an
IP. Confirm the containers can see each other:

```bash
docker compose exec sonarr curl -s -o /dev/null -w '%{http_code}\n' http://sabnzbd:8080
```

**Seerr can't reach Jellyfin.** Expected — use the host's LAN IP, or remove
`network_mode: bridge` from the `jellyfin` service. See
[step 5](#5-seerr-httplocalhost5055).

**Imports are slow and disk usage doubles.** Hardlinking is failing. Either
*Use Hardlinks instead of Copy* is off in Sonarr/Radarr, or downloads and media
are on different filesystems. Verify inside the container:

```bash
docker compose exec sonarr df /data/usenet /data/media
```

Same device on both lines means hardlinks are possible.

**`database is locked` in Sonarr or Radarr.** This is what the
`SQLite_Timeout=5000` setting already in the compose file addresses — it waits
5 s for a lock instead of erroring immediately. If you still see it, the `/config`
directory is on storage that handles SQLite locking badly. Move it to local disk;
never put `/config` on an NFS or SMB share.

**Jellyfin transcoding pegs the CPU.** You are software transcoding. On Linux,
pass through your GPU:

```yaml
jellyfin:
  devices:
    - /dev/dri:/dev/dri    # Intel QSV or AMD VAAPI
  group_add:
    - "989"                # your host's 'render' group: getent group render
```

Then enable VAAPI in *Dashboard → Playback → Transcoding*. Not available on
macOS or Windows containers — see those sections.

**Changes to `docker-compose.yml` don't take effect.** Volume and environment
changes need a recreate, not a restart:

```bash
docker compose up -d --force-recreate <service>
```

---

## Security notes

This compose file publishes six web UIs on all interfaces with no authentication
in front of them. That is fine on a trusted LAN and **not fine exposed to the
internet**.

- Set a password in every service immediately. Sonarr, Radarr and Prowlarr start
  with authentication disabled.
- Don't port-forward these to the internet. If you need remote access, use a VPN
  (WireGuard, Tailscale) or put a reverse proxy with TLS and auth in front
  (Caddy, Traefik, or `swag`).
- To restrict a service to the local machine only, bind it explicitly:
  `- "127.0.0.1:8080:8080"`.
- Your Usenet and indexer credentials sit in plaintext in `/data/docker`. Keep
  backups of that directory somewhere you would keep a password file.
