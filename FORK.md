# HedgeDoc Fork Runbook: Self‑hosting

This document explains how to self‑host the `bexelbie/hedgedoc` fork using container images from `bexelbie/hedgedoc-container`.
It is written for people who want a clear, copy‑pastable path to running the fork on their own infrastructure.

If you want stock HedgeDoc, you should follow the upstream docs instead: https://docs.hedgedoc.org/setup/.

---

## 1. What this fork gives you

There are two repositories involved:

- **Application fork:** `https://github.com/bexelbie/hedgedoc`
- **Container/build fork (this repo):** `https://github.com/bexelbie/hedgedoc-container`

The application fork adds UX and collaboration features on top of HedgeDoc 1.x (see the FORK.md in the app repo for details).
This container repo builds images from that fork and publishes them to GitHub Container Registry so you can run them without rebuilding everything manually.

---

## 2. Choose an image

The default published image you can use directly is:

- `ghcr.io/bexelbie/hedgedoc-bex:latest-alpine`

You can check available tags on the `Packages` tab of the `bexelbie/hedgedoc-container` repository on GitHub.

If you prefer to build your own image (for example, to use a different branch or make local changes), from this repo run:

```bash
# From the hedgedoc-container checkout
docker build \
  -f alpine/Dockerfile \
  --build-arg HEDGEDOC_REPOSITORY=https://github.com/bexelbie/hedgedoc.git \
  --build-arg VERSION=bex-master \
  -t ghcr.io/<your-user>/hedgedoc-bex:local .
```

Then substitute your tag (e.g. `ghcr.io/<your-user>/hedgedoc-bex:local`) wherever this runbook uses `ghcr.io/bexelbie/hedgedoc-bex:latest-alpine`.

---

## 3. Minimal configuration

You need two things:

1. A HedgeDoc configuration file (JSON)
2. A few environment variables for the container

### 3.1 Example `config.json`

Create a file `config/config.json` on your host with at least:

```json
{
  "production": {
    "domain": "docs.example.com",
    "host": "0.0.0.0",
    "protocolUseSSL": true,
    "loglevel": "info",
    "db": {
      "dialect": "sqlite",
      "storage": "/data/db/hedgedoc.sqlite"
    },
    "email": true,
    "allowEmailRegister": false,
    "allowAnonymous": false,
    "allowAnonymousEdits": true,
    "requireFreeURLAuthentication": true,
    "disableNoteCreation": false,
    "allowFreeURL": false,
    "enableStatsApi": false,
    "defaultPermission": "limited",
    "imageUploadType": "filesystem"
  }
}
```

Adjust `domain` and other policy knobs to match your environment and threat model.
For the full list of options, see the upstream configuration docs: https://docs.hedgedoc.org/configuration/.

### 3.2 Environment variables

Set at least:

- `CMD_SESSION_SECRET` – a long random secret (used to sign sessions)
- `CMD_CONFIG_FILE=/hedgedoc/config.json` – where HedgeDoc should look for the config
- `NODE_ENV=production`

How you provide these (env file, Quadlet `Environment`/`EnvironmentFile`, Kubernetes Secret, etc.) depends on your runtime; the examples below use Podman Quadlet style.

---

## 4. Quickstart with Podman Quadlet (rootless)

This example shows a rootless Podman setup using Quadlet, with everything for HedgeDoc under `/srv/hedgedoc` on the host.
Adjust paths and user IDs to match your system.

1. Create directories on the host:

   ```bash
   sudo mkdir -p /srv/hedgedoc/data/db /srv/hedgedoc/data/uploads
   sudo mkdir -p /srv/hedgedoc
   sudo chown -R $USER:$USER /srv/hedgedoc
   ```

2. Place your `config.json` (from section 3.1) at `/srv/hedgedoc/config.json`.

3. Create a Quadlet unit file for a rootless user, for example at:

   - `~/.config/containers/systemd/hedgedoc.container`

   with contents similar to:

   ```ini
   [Unit]
   Description=HedgeDoc (SQLite) rootless container
   Wants=network-online.target
   After=network-online.target

   [Container]
   Image=ghcr.io/bexelbie/hedgedoc-bex:latest-alpine
   ContainerName=hedgedoc
   Label=io.containers.autoupdate=registry
   Network=host

   Volume=/srv/hedgedoc/data/db:/data/db:Z
   Volume=/srv/hedgedoc/data/uploads:/data/uploads:Z
   Volume=/srv/hedgedoc/config.json:/hedgedoc/config.json:Z,ro

   Environment=NODE_ENV=production
   Environment=CMD_SESSION_SECRET=change-me-to-a-long-random-string
   Environment=CMD_CONFIG_FILE=/hedgedoc/config.json

   [Service]
   Restart=always

   [Install]
   WantedBy=default.target
   ```

4. Reload systemd and start the service as your user:

   ```bash
   systemctl --user daemon-reload
   systemctl --user enable --now hedgedoc.service
   ```

5. Visit `http://localhost:3000/` (or your reverse-proxy URL) and verify that you can sign in, create a note, and collaborate as expected.

For a system-wide (rootful) deployment, use `/etc/containers/systemd/hedgedoc.container` and `systemctl` instead of `systemctl --user`.

---

## 5. Operating and upgrading (Quadlet)

### 5.1 Upgrading to a new image

If you are using the published image and relying on `io.containers.autoupdate=registry`, you can use Podman’s auto-update:

```bash
podman auto-update --dry-run   # see what would change
podman auto-update             # apply updates
```

After an update, restart the systemd unit if needed:

```bash
systemctl --user restart hedgedoc.service
```

If you build your own image, rebuild with `podman build` (or `docker build`) and update the `Image=` line in the Quadlet file to point at your new tag, then reload and restart:

```bash
systemctl --user daemon-reload
systemctl --user restart hedgedoc.service
```

### 5.2 Backups

At minimum, back up:

- `/srv/hedgedoc/data/db` – contains `hedgedoc.sqlite` (all notes and metadata)
- `/srv/hedgedoc/data/uploads` – user‑uploaded files
- `/srv/hedgedoc/config.json` – your configuration

To restore, recreate those directories/files in the same locations and start the service again.

### 5.3 Rollback

To roll back to an older image:

1. Change the `Image=` line in your Quadlet file to a known‑good tag.
2. Run `systemctl --user daemon-reload`.
3. Run `systemctl --user restart hedgedoc.service`.
4. Verify basic functionality (login, edit, share link, etc.).

---

## 6. Troubleshooting

Some quick checks if something goes wrong:

- **Container starts but HTTP fails:** check container logs for config‑file errors and confirm `CMD_CONFIG_FILE` matches the mount path.
- **Notes or uploads disappear after restart:** ensure `./data/db` and `./data/uploads` are mounted as volumes and not inside the container filesystem only.
- **Unexpected access behavior (too open/too strict):** review `allowAnonymous`, `allowAnonymousEdits`, `defaultPermission`, and related fields in `config.json`.
- **New code changes not appearing:** confirm you pulled the latest image tag or rebuilt from the expected branch.

If you run into issues that look like generic HedgeDoc problems rather than fork‑specific ones, the upstream docs and community are still the best reference: https://docs.hedgedoc.org/.

---

## 7. Fork-specific commits over upstream/master (brief)

The canonical detail is the commit history (`git log upstream/master..master`).
Brief summary of current fork-only commits:

- `f790d8e` Add bex-master build workflow and disable quay.io builds
- `cb11561` Fix build errors and disable tests for fork
- `79e597d` Fix tag duplication and re-enable tests for alpine only
- `911ca23` Fix repository URL from hedge-doc to hedgedoc
- `f74cca0` Disable Docker cache to ensure fresh git clones on each build

---

## 8. How the images are built (for maintainers)

If you are maintaining this fork rather than just using it, the pipeline in this repo does the following:

- clones `bexelbie/hedgedoc` at the configured branch (`bex-master` in `bex-master-build.yml`)
- builds the app with Yarn
- assembles a minimal production image from `alpine/Dockerfile`
- publishes to `ghcr.io/bexelbie/hedgedoc-bex` with architecture and variant tags

### 7.1 Manual build trigger

Builds are **not** automatically triggered by pushes to `bexelbie/hedgedoc`.
To cut a new image after changing the app fork, you must explicitly run the workflow in this repo:

- go to `bexelbie/hedgedoc-container` → **Actions** → **Build from bex-master**
- click **Run workflow** and confirm

Alternatively, pushing a change to `master` in this repo will also trigger the workflow, but the recommended path is to treat image builds as an intentional, manual operation.

You can inspect `.github/workflows/bex-master-build.yml` and `alpine/Dockerfile` for the exact details.

This separation (app fork vs. container fork) is intentional so that you can rebase against upstream HedgeDoc while keeping deployment wiring in this repository relatively stable.
