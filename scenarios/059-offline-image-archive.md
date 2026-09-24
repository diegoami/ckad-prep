# 059 — Hand over an image as a single archive for an offline site

**Domain:** Application Design and Build · **Difficulty:** Easy

Unlike `011-build-and-push-image.md`, which builds with both Docker and Podman and pushes to a
registry, nothing here touches a registry: the deliverable is one tar file on disk that carries two
image names, and the point is knowing that a plain `docker save` is enough.

## Task

> The `annatto` team ships its `label-printer` service to a site with no network access. The build
> context is in `~/ckad/059/`, and its Dockerfile takes the release number as a build argument.
>
> 1. Build the image with the build argument `RELEASE=2.3.0` so that it carries **both** of these
>    names: `harbor.annatto.internal/label-printer:2.3.0` and
>    `harbor.annatto.internal/label-printer:stable`.
> 2. Write one archive, `~/ckad/059/label-printer-2.3.0.tar`, that contains the image under both
>    names. The site imports it with a tool that only accepts an OCI image layout.
>
> Don't push anything.

## Documentation

What to look up: **Images**, on the Kubernetes side. `docker build`/`docker save` are documented
by Docker, not Kubernetes.
- <https://kubernetes.io/docs/concepts/containers/images/> — image names, registries and tags, which
  is what the two names above are made of.

## Setup

```bash
mkdir -p ~/ckad/059 && cd ~/ckad/059
cat > Dockerfile <<'EOF'
FROM busybox:1.36
ARG RELEASE=dev
ENV RELEASE=${RELEASE}
CMD ["sh", "-c", "echo label-printer release $RELEASE"]
EOF
```

## Solution

`-t` can be given more than once, so one build produces both names. `--build-arg` sets the `ARG`,
and the `ENV` line copies it into the image so it's still there at run time:
```bash
cd ~/ckad/059
docker build --build-arg RELEASE=2.3.0 \
  -t harbor.annatto.internal/label-printer:2.3.0 \
  -t harbor.annatto.internal/label-printer:stable .

docker image ls harbor.annatto.internal/label-printer
# harbor.annatto.internal/label-printer:2.3.0    17a401127e1f  ...
# harbor.annatto.internal/label-printer:stable   17a401127e1f  ...   <- same image ID
```
Building once and running `docker tag` for the second name gives the same result.

`docker save` takes several image references and writes them into one archive:
```bash
docker save -o label-printer-2.3.0.tar \
  harbor.annatto.internal/label-printer:2.3.0 \
  harbor.annatto.internal/label-printer:stable
```
Name both references. Saving only `:2.3.0` gives an archive that loads without the `stable` name,
even though both names point at the same image ID. (`docker save harbor.annatto.internal/label-printer`
with no tag saves every tag of that repository, which works here but can drag in more than you
meant.)

**Is it OCI?** On a Docker Engine that uses the containerd image store (the default on current
Docker Desktop and recent Engine releases; `docker info` shows the driver type
`io.containerd.snapshotter.v1`), the archive `docker save` writes is an OCI image layout. Check for
the two things the layout spec requires, an `oci-layout` file and an `index.json`:
```bash
mkdir -p verify && tar -xf label-printer-2.3.0.tar -C verify
ls verify
# blobs  index.json  manifest.json  oci-layout

cat verify/oci-layout
# {"imageLayoutVersion":"1.0.0"}

grep -o '"io.containerd.image.name":"[^"]*"' verify/index.json
# "io.containerd.image.name":"harbor.annatto.internal/label-printer:2.3.0"
# "io.containerd.image.name":"harbor.annatto.internal/label-printer:stable"
```
The extra `manifest.json` is there so older `docker load` versions can still read the file. On an
Engine with the classic image store, `docker save` writes Docker's legacy layout instead
(`repositories` plus per-layer directories, no `oci-layout`). On the exam, go with whatever the
provided Engine produces.

Round trip, to prove the archive is complete: delete both names, load, run.
```bash
docker rmi harbor.annatto.internal/label-printer:2.3.0 harbor.annatto.internal/label-printer:stable
docker load -i label-printer-2.3.0.tar
# Loaded image: harbor.annatto.internal/label-printer:2.3.0
# Loaded image: harbor.annatto.internal/label-printer:stable
docker run --rm harbor.annatto.internal/label-printer:stable
# label-printer release 2.3.0
```

## Cleanup

```bash
cd ~
docker rmi harbor.annatto.internal/label-printer:2.3.0 harbor.annatto.internal/label-printer:stable 2>/dev/null
rm -rf ~/ckad/059
```

*Verified end-to-end on a local kind cluster (Kubernetes v1.30) on 2026-09-24.*
