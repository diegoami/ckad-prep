# 011 — Build with Docker and Podman, push to a registry, run detached, capture logs

**Domain:** Application Design and Build · **Difficulty:** Medium

## Task

> The `rowan` team has a small Go service, `ledger-reporter`, that prints a status line to stdout
> every couple of seconds. Its source and Dockerfile are in `~/ckad/011/image/`.
>
> 1. The Dockerfile sets `REPORTER_ID` to a placeholder. Change it so the image bakes in
>    `REPORTER_ID=rowan-7f3a9c`.
> 2. Build the image with Docker, tag it `localhost:5000/ledger-reporter:1.0-docker`, and push it
>    to the registry at `localhost:5000`.
> 3. Build the same image with Podman, tag it `localhost:5000/ledger-reporter:1.0-podman`, and push
>    that too.
> 4. From the Podman-built image, start a detached container named `ledger-reporter`.
> 5. Save that container's log output to `~/ckad/011/ledger-reporter.log`.

## Documentation

What to look up: **Images**, for the Kubernetes-side concerns (tags, pull policy) — the actual
`docker build`/`podman build` mechanics live in Docker's/Podman's own docs, not Kubernetes'.
- <https://kubernetes.io/docs/concepts/containers/images/> — image names/tags and how a Pod spec
  resolves them, relevant once you `kubectl run` off what you pushed.

## Setup

In the exam the registry is provided for you. Locally, run a throwaway registry container on port
5000 to push to. Docker treats `localhost` registries as insecure by default, so no TLS setup is
needed. Podman needs `--tls-verify=false` (see the Solution).

If your user isn't in the `docker` group, prefix the `docker` commands with `sudo`. The exam
environment often expects `sudo docker` / `sudo podman`, so read the task wording there.

```bash
docker run -d -p 5000:5000 --name ckad-registry registry:2

mkdir -p ~/ckad/011/image && cd ~/ckad/011/image
cat > main.go <<'EOF'
package main

import (
	"fmt"
	"os"
	"time"
)

func main() {
	for {
		fmt.Println("ledger-reporter id:", os.Getenv("REPORTER_ID"), "at", time.Now().Format(time.RFC3339))
		time.Sleep(2 * time.Second)
	}
}
EOF
cat > Dockerfile <<'EOF'
FROM docker.io/library/golang:1.22-alpine AS build
WORKDIR /src
COPY main.go .
RUN CGO_ENABLED=0 go build -o /out/ledger-reporter main.go

FROM docker.io/library/alpine:3.20
COPY --from=build /out/ledger-reporter /usr/local/bin/ledger-reporter
ENV REPORTER_ID=placeholder
CMD ["ledger-reporter"]
EOF
```

## Solution

```bash
cd ~/ckad/011/image

# 1. set the ENV value (or just edit the line in vim)
sed -i 's/^ENV REPORTER_ID=.*/ENV REPORTER_ID=rowan-7f3a9c/' Dockerfile
grep ENV Dockerfile
# ENV REPORTER_ID=rowan-7f3a9c

# 2. build + push with docker
docker build -t localhost:5000/ledger-reporter:1.0-docker .
docker push localhost:5000/ledger-reporter:1.0-docker

# 3. build + push with podman (same Dockerfile; podman reads it as-is)
podman build -t localhost:5000/ledger-reporter:1.0-podman .
podman push --tls-verify=false localhost:5000/ledger-reporter:1.0-podman

# confirm both tags landed in the registry
curl -s http://localhost:5000/v2/ledger-reporter/tags/list
# {"name":"ledger-reporter","tags":["1.0-docker","1.0-podman"]}

# 4. run detached from the podman-built image
podman run -d --name ledger-reporter localhost:5000/ledger-reporter:1.0-podman
podman ps

# 5. capture the logs
sleep 5
podman logs ledger-reporter > ~/ckad/011/ledger-reporter.log
cat ~/ckad/011/ledger-reporter.log
# ledger-reporter id: rowan-7f3a9c at 2026-...
```

Things that trip people up:

- Docker and Podman keep separate image stores. The container in step 4 must come from the
  Podman-built tag, and it must be started with `podman run`. `docker run` won't find a
  Podman-only image locally.
- If the task says `sudo podman`, use `sudo` for every Podman step. Root and rootless Podman have
  separate image stores too, so an image built with `sudo podman build` doesn't show up in plain
  `podman images`.
- `podman logs ... > file` writes the logs to the file. `podman logs -f` never exits, so don't use
  it when redirecting to a file.

## Cleanup

```bash
podman rm -f ledger-reporter
podman rmi localhost:5000/ledger-reporter:1.0-podman
docker rmi localhost:5000/ledger-reporter:1.0-docker
docker rm -f ckad-registry
cd ~ && rm -rf ~/ckad/011
```

---

*Verified on 2026-09-23: Setup, the Docker steps and the Docker cleanup ran as written on the host. Podman
isn't installed on the test machine, so the Podman steps were run inside a `quay.io/podman/stable`
container (with `--storage-driver=vfs`, needed only for Podman-inside-Docker), including the
Podman cleanup, pushing to the same local registry. No Kubernetes cluster is involved in this scenario.*
