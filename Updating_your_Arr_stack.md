# Updating your ARR stack

Once your stack is running, updating the containers is a repetitive loop:
back up your config, pull the newest images, recreate the containers whose
image changed, and clean up the old layers. This guide gives you one script
that wraps that whole loop into a single command.

Because the images in the compose file are pinned to `:latest`, "updating"
just means pulling the newest image and recreating the container. Your
configuration and media are never touched, because they live in bind mounts
on the host (your appdata folder and your `/data` folder), not inside the
containers.

---

## What the script does

Each time you run it, in order:

1. **Backs up your appdata folder** to a timestamped `.tar.gz`. This folder
   holds your entire configuration - indexers, API keys, quality profiles -
   so a backup means you can always roll back a bad update.
2. **Pulls the newest images** for your services.
3. **Recreates only the containers whose image actually changed** (the rest
   keep running untouched).
4. **Prunes the old, now-unused image layers** so they don't pile up on disk.
5. Keeps only the most recent few backups so the backup folder doesn't grow
   forever.

It also has a `--safe` flag that stops the stack during the backup, so the
databases aren't being written to while they're archived - a clean snapshot.
Use it before a risky or major update; skip it for routine ones.

---

## Before you paste: find YOUR two paths

The script has two placeholder paths at the top that you must change to match
your own machine. Here's how to find each one.

### 1. Your compose folder (`COMPOSE_DIR`)

This is the folder that contains the `docker-compose.yml` your stack actually
runs from. To find it, ask Docker which compose project is running:

```
sudo docker compose ls
```

You'll get something like:

```
NAME     STATUS       CONFIG FILES
data     running(7)   /data/docker-compose.yml
```

The path under `CONFIG FILES` is your compose file. **`COMPOSE_DIR` is the
folder that file sits in** - so if it says `/data/docker-compose.yml`, your
`COMPOSE_DIR` is `/data`. If it says `/home/yourname/docker-compose.yml`,
it's `/home/yourname`.

### 2. Your appdata folder (`APPDATA_DIR`)

This is where each container stores its config on the host. Look in your
`docker-compose.yml` at the volume lines - you'll see entries like:

```
volumes:
  - /docker/appdata/radarr:/config
```

The host path before `:/config` is your appdata root. In the example above
the containers live under `/docker/appdata`, so `APPDATA_DIR` is
`/docker/appdata`. If yours differs, use whatever comes before `/<service>/:/config`.

---

## Install the script

Once you know your two paths, create the file. Paste this whole block into
your terminal - but first change the two paths in the CONFIG section to the
ones you found above:

```bash
cat > ~/arr-update.sh << 'SCRIPT_EOF'
#!/usr/bin/env bash
#
# arr-update.sh — update the ARR docker stack.
# Backs up appdata, pulls newest images, recreates changed containers,
# then prunes dangling images.
#
# Usage:
#   ./arr-update.sh                  # hot backup, update all services
#   ./arr-update.sh --safe           # stop stack for a clean DB snapshot, then update all
#   ./arr-update.sh radarr           # update one (or more) named services
#   ./arr-update.sh --safe radarr    # clean snapshot + update just Radarr
#

set -euo pipefail

# ══════════════════════════════════════════════════════════════════════
#   EDIT THESE TWO LINES — nothing else needs changing (see guide above)
# ══════════════════════════════════════════════════════════════════════
COMPOSE_DIR="/path/to/your/compose/folder"   # <<< CHANGE ME (folder with docker-compose.yml)
APPDATA_DIR="/path/to/your/appdata"          # <<< CHANGE ME (your container config folder)
# ══════════════════════════════════════════════════════════════════════
BACKUP_DIR="$HOME/arr-backups"               # where backups are written (fine as-is)
KEEP_BACKUPS=3                               # how many backups to keep

# Parse --safe flag; everything else is passed through as service names.
SAFE=0
SERVICES=()
for arg in "$@"; do
  if [ "$arg" = "--safe" ]; then
    SAFE=1
  else
    SERVICES+=("$arg")
  fi
done

# Use sudo only if the current user can't reach the docker socket.
if docker ps >/dev/null 2>&1; then
  DOCKER="docker"
else
  DOCKER="sudo docker"
fi

log() { printf '\n\033[1;34m==>\033[0m %s\n' "$1"; }

cd "$COMPOSE_DIR"

# 1. Back up appdata ---------------------------------------------------
if [ "$SAFE" -eq 1 ]; then
  log "Stopping stack for a clean snapshot"
  $DOCKER compose stop
fi

log "Backing up $APPDATA_DIR"
mkdir -p "$BACKUP_DIR"
BACKUP_FILE="$BACKUP_DIR/appdata-$(date +%F-%H%M%S).tar.gz"
sudo tar czf "$BACKUP_FILE" "$APPDATA_DIR"
echo "    saved: $BACKUP_FILE"

if [ "$SAFE" -eq 1 ]; then
  log "Restarting stack"
  $DOCKER compose start
fi

# Trim old backups, keep the newest $KEEP_BACKUPS
ls -1t "$BACKUP_DIR"/appdata-*.tar.gz 2>/dev/null \
  | tail -n +$((KEEP_BACKUPS + 1)) \
  | xargs -r rm -f

# 2. Pull newest images ------------------------------------------------
log "Pulling images"
$DOCKER compose pull "${SERVICES[@]}"

# 3. Recreate changed containers --------------------------------------
log "Recreating containers"
$DOCKER compose up -d "${SERVICES[@]}"

# 4. Clean up old image layers ----------------------------------------
log "Pruning dangling images"
$DOCKER image prune -f

log "Done. Running containers:"
$DOCKER compose ps
SCRIPT_EOF
```

Then make it executable:

```
chmod +x ~/arr-update.sh
```

> If you'd rather edit the paths after pasting instead of before, open the
> file with `nano ~/arr-update.sh`, change the two lines in the CONFIG
> section, then save with `Ctrl+O`, `Enter`, `Ctrl+X`.

---

## Using it

```
~/arr-update.sh          # routine update, minimal downtime
~/arr-update.sh --safe   # clean-snapshot update, before a major/risky bump
```

You can also update a single service by name:

```
~/arr-update.sh radarr
~/arr-update.sh --safe radarr
```

On the first run it will ask for your `sudo` password (for the backup, and
for Docker if your user isn't in the `docker` group). That's expected. After
that, keeping your stack current is just running that one command.
