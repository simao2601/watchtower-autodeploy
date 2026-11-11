# Automatic Deployment with Watchtower (Pull Model)

Topic 13: a running app updates itself after every `git push`, without the CI/CD pipeline ever touching the server.

**Author:** [YOUR NAME] · [COURSE / CLASS] · [DATE]

---

## 1. What this project does

1. I push a code change to GitHub.
2. GitHub Actions builds a Docker image and publishes it to a registry (GHCR).
3. On the server, a container called **Watchtower** checks the registry every 30 seconds.
4. When it finds a new image, it replaces the running app container with the new one.

The pipeline only builds and publishes. It has no access to the server. This is the **pull model**: the server fetches updates, nobody pushes them in.

The app is a tiny static web page. Its only job is to show a version number ("Version 5", "Version 6"...) so the update is easy to see.

## 2. Architecture

```
 You          GitHub Actions            GHCR (registry)         Server (VM)
┌─────┐ push ┌───────────────┐ publish ┌───────────────┐ poll  ┌─────────────┐
│ Git │ ───► │ build image   │ ──────► │ app:latest    │ ◄──── │ Watchtower  │
└─────┘      └───────────────┘         └───────────────┘ 30 s  │     │       │
                                                               │     ▼       │
                                                               │   my app    │
                                                               └─────────────┘
```

| Part | Role |
|---|---|
| GitHub Actions | Builds the image and publishes it. Never connects to the server |
| GHCR (`ghcr.io`) | Stores the image (public package) |
| `minhaapp` | Nginx container that serves the web page |
| Watchtower | Checks the registry and recreates `minhaapp` when the image changes |

## 3. Push vs. pull

There are two ways to deploy a new version of an app to a server.

**Push model:** the pipeline starts the update. After building the image, it connects to the server (usually with SSH) and tells it to update. This means the pipeline needs access to the server, so SSH credentials are stored in the CI system and the server must accept connections from outside.

**Pull model:** the server starts the update. The pipeline only builds the image and publishes it to a registry. A program on the server (here, Watchtower) checks the registry regularly and updates the app when it finds a new image. The pipeline never talks to the server.

| | Push | Pull (this project) |
|---|---|---|
| Who starts the update | The pipeline | The server |
| Pipeline needs server access | Yes (SSH, credentials) | No |
| Server needs ports open to the outside for the pipeline | Yes | No, it only makes outgoing connections |
| Update timing | Immediate, decided by the pipeline | Delayed by the polling interval (30 s here) |

**Why I chose pull:** it is safer, because no server credentials are stored in the pipeline and no ports have to be opened to the outside. It is also simple: the pipeline has one job (build and publish) and the server looks after itself. The cost is a small delay, because the update only happens at the next check.

## 4. Repository structure

```
├── .github/workflows/build.yml   # pipeline: build and publish image
├── app/
│   ├── index.html                # web page with the version number
│   └── Dockerfile                # nginx:alpine + index.html
├── server/
│   └── docker-compose.yml        # app + Watchtower
├── docs/                         # timing results and screenshots
└── README.md
```

## 5. How the code works

**Dockerfile** (`app/Dockerfile`): starts from the small `nginx:alpine` image and copies `index.html` into the folder Nginx serves. `EXPOSE 80` documents the port. Alpine is used because the image is small, so each update downloads quickly.

**Pipeline** (`.github/workflows/build.yml`):
- Runs on every push to `main` (or manually).
- Runs on a temporary Ubuntu runner provided by GitHub.
- Logs in to GHCR with the automatic `GITHUB_TOKEN` (no passwords in the repo).
- Builds the image from `./app` and publishes `ghcr.io/simao2601/watchtower-autodeploy:latest`.

**Compose file** (`server/docker-compose.yml`): runs two containers.
- `app`: the image from GHCR, on port 80.
- `watchtower`: checks every 30 s (`WATCHTOWER_POLL_INTERVAL=30`), removes old images (`WATCHTOWER_CLEANUP=true`) and only watches `minhaapp`.

**How Watchtower detects a new version:** every 30 seconds it asks the registry for the digest of the image that `minhaapp` uses. The digest is a unique hash of the image content. It compares it with the digest of the image already running on the server. If they are the same, nothing happens. If they differ, Watchtower downloads the new image, stops the old container and starts a new one with the same settings. This works even though the tag is always `latest`, because the digest changes whenever the content changes.

## 6. How to run it

You need a Linux machine with Docker and Docker Compose.

```bash
git clone https://github.com/simao2601/watchtower-autodeploy.git
cd watchtower-autodeploy/server
docker compose up -d
```

Open `http://<machine-ip>` in a browser. To watch Watchtower work: `docker compose logs -f watchtower`.

## 7. Results

### 7.1 Control experiment

I stopped Watchtower and published Version 4. The pipeline succeeded and the new image was in the registry, but the page stayed on Version 3. When I started Watchtower again, the page changed to Version 4 on its next check, with no new push.

**Conclusion:** the pipeline only updates the registry. Watchtower is what updates the server.

### 7.2 Timing

I ran 5 valid cycles. One earlier cycle (V6) was a warm-up and one (V8) was discarded because its record was incomplete.

| Cycle | Push → Watchtower update (s) | Push → new page (s) |
|---|---|---|
| V7 | 44 | 49 |
| V9 | 31 | 35 |
| V10 | 25 | 27 |
| V11 | 35 | 37 |
| V12 | 30 | 31 |
| **Average** | **33.0** | **35.8** |

On average the app was updated about **36 seconds** after the push. Replacing the container itself took only a few seconds. The warm-up cycle (V6) took 83 s; I did not confirm why, but cached image layers are a possible reason.

Limitations:
- Times were recorded by hand with buttons in a small local tool, so there is an error of about 1 to 2 seconds.
- The "pipeline finished" time was unreliable in two cycles, because the GitHub Actions page refreshes only every 30 s. That is why the two main columns above do not use it.
- With a 30 s polling interval, the wait for Watchtower varies between 0 and 30 s.
- Five cycles are only indicative, not statistically strong.

## 8. Problems I ran into

| Problem | Cause | Fix |
|---|---|---|
| Original Watchtower exited at start with `client version 1.25 is too old` | It uses an old Docker API that current Docker rejects | Switched to the maintained fork `nickfedor/watchtower` |
| VM clock was 5 days behind after the VM was stopped | The clock did not resync | Check `date -u` on the VM before each test session |
| Negative polling values in two cycles | I clicked "pipeline finished" late | Used measurements that do not depend on that click |

## 9. Limitations and critical analysis

- **Docker socket:** Watchtower mounts `/var/run/docker.sock`. This gives it almost full control of the machine, because anyone who controls the Docker socket can start any container. If Watchtower were compromised, so would the server. To reduce the risk, it only watches one container and uses an image from a source I checked.
- **Maintenance risk:** the original Watchtower stopped working with recent Docker versions. I switched to a fork, which also needs to be trusted and kept up to date. A tool with this much access should be actively maintained.
- **No rollback:** if a broken image is published, Watchtower still deploys it. There are no tests in the pipeline and no health check before the old container is replaced.
- **`latest` tag:** it moves with every build, so I cannot tell which commit is running. Tagging images with the commit hash would fix this.
- **Polling:** 30 seconds is good for a demo, but too frequent for production, because it sends many requests to the registry. A longer interval would cost less but delay updates.
- **Public image:** it keeps the demo simple because the server needs no credentials, but anyone can download the image. A private image would need a read-only token stored on the server.
- **Basic pipeline:** it only builds and publishes. It does not run tests or scan the image.

## 10. Comparison with the push model

Pull is safer for the server, because the pipeline holds no server credentials and no ports need to be opened. It is also simpler to set up, because the pipeline has a single job. The downside is less control: the update happens automatically at the next check, and I cannot decide exactly when, or stop a bad version from being deployed. Push gives immediate, controlled deployments, but needs server access from the CI system, which is a bigger security risk.

## 11. Possible improvements

- Tag images with the commit hash as well as `latest`.
- Use Watchtower labels to choose which containers are updated.
- Add notifications (Discord or email).
- Use a private image with a read-only token.
- Run the pipeline only when `app/` changes.

## 12. Tools

Git and GitHub · GitHub Actions · GitHub Container Registry · Docker and Docker Compose · Nginx (Alpine) · Watchtower (`nickfedor/watchtower`) · Multipass (Ubuntu VM)