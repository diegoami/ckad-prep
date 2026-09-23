# 059 — Build an image, tag it correctly, save it in OCI format

**Domain:** Application Design and Build · **Difficulty:** Easy

Unlike `011-build-and-push-image.md`, which builds with both Docker and Podman and pushes to a
registry, nothing here touches a registry: the deliverable is a tar file on disk, and the point is
knowing that a plain `docker save` is enough.

## Task

> The Thames team keeps the build files for its `invoice-api` image in `~/ckad/059/`. Before
> building, change the `ENV` instruction in the Dockerfile so `BUILD_ID` is set to
> `rc-2026-07-19-01`. Build the image as `registry.ckad.local/team-thames/invoice-api:v1`, then
> export it in OCI format to `~/ckad/059/invoice-api-v1-oci.tar`. Don't push it anywhere.

## Documentation

What to look up: **Images**, on the Kubernetes side — `docker save`/OCI format itself is Docker's
own documentation, not Kubernetes'.
- <https://kubernetes.io/docs/concepts/containers/images/> — image references and tags, relevant
  once the saved/loaded image needs to be referenced from a Pod spec.

## Setup

```bash
mkdir -p ~/ckad/059 && cd ~/ckad/059
cat > Dockerfile <<'EOF'
FROM busybox:1.36
ENV BUILD_ID=placeholder
CMD ["sh", "-c", "echo build-id: $BUILD_ID"]
EOF
```

## Solution

Set the `ENV` value, build, and tag in one step with `-t` (tagging separately with `docker tag`
afterward works too, `-t` at build time is just fewer commands):
```bash
cd ~/ckad/059
sed -i 's/ENV BUILD_ID=placeholder/ENV BUILD_ID=rc-2026-07-19-01/' Dockerfile

docker build -t registry.ckad.local/team-thames/invoice-api:v1 .
```

Save it — `docker save` is genuinely all this needs, no special flag:
```bash
docker save --output invoice-api-v1-oci.tar registry.ckad.local/team-thames/invoice-api:v1
```

**Gotcha verified live:** on a Docker Engine using the containerd image store (`docker version`
showing a `containerd` section under `Server`, which is the default on current Docker Desktop and
recent Engine installs), the tar `docker save` produces genuinely *is* an OCI Image Layout — not
Docker's older proprietary tar format. Confirmed by extracting it and checking for the two files
the OCI Image Layout spec requires:
```bash
mkdir -p verify && tar -xf invoice-api-v1-oci.tar -C verify
ls verify
# blobs  index.json  manifest.json  oci-layout

cat verify/oci-layout
# {"imageLayoutVersion":"1.0.0"}
```
`oci-layout` and an `index.json` with `mediaType: application/vnd.oci.image.index.v1+json` are the
two things that make it OCI rather than Docker's legacy format. If your Engine is still on the
classic image store (no containerd integration), `docker save` instead produces Docker's original
tar layout (`repositories` + per-layer directories, no `oci-layout` file) — check
`docker info --format '{{.DriverStatus}}'` or look for a `containerd` block in `docker version` if
the saved tar's contents look unfamiliar; on the real exam, assume whatever the provided Engine
does is correct and don't fight it.

Round-trip proof that the tar is a complete, correct image — not just a plausible-looking file:
```bash
docker rmi registry.ckad.local/team-thames/invoice-api:v1 -f
docker load --input invoice-api-v1-oci.tar
docker run --rm registry.ckad.local/team-thames/invoice-api:v1
# build-id: rc-2026-07-19-01
```

## Cleanup

```bash
cd ~
docker rmi registry.ckad.local/team-thames/invoice-api:v1 -f 2>/dev/null
rm -rf ~/ckad/059
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-23.*
