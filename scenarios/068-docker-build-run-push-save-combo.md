# 068 — Docker: build once, smoke-test on a published port, push two tags, bundle two names in one archive

**Domain:** Application Design and Build · **Difficulty:** Medium

`059` only builds, tags and saves, and `011` builds, pushes and runs. This one chains the whole
workflow, and every step needs a *different reference* to the same image: a local tag, two
registry-qualified tags, and a set of local names inside an archive. What's being tested is keeping
track of which reference each command needs, and checking the result rather than trusting exit
codes.

## Task

> The `cobalt` team keeps the Dockerfile for its `ledger-api` service in
> `~/ckad/068/ledger-api/`. Using Docker:
>
> 1. Build an image `ledger-api:4.2.0` from it.
> 2. Smoke-test it: run a detached container `ledger-smoke` from that image, reachable on host port
>    `18080` (the service listens on `8080` inside the container), and save the response to
>    `http://localhost:18080/` in `~/ckad/068/smoke.txt`.
> 3. Publish the image to the team registry at `localhost:5055`, repository `cobalt/ledger-api`,
>    under both the tag `4.2.0` and the tag `stable`.
> 4. Write a single archive `~/ckad/068/ledger-api-bundle.tar` that, when loaded, gives back the
>    image under both local names `ledger-api:4.2.0` and `ledger-api:stable`.

## Documentation

What to look up: **Images**, on the Kubernetes side. Build, tag, push and save are covered by
Docker's own documentation, not Kubernetes'.
- <https://kubernetes.io/docs/concepts/containers/images/> — image reference syntax (registry host,
  repository path, tag), relevant once any of these references ends up in a Pod spec.

## Setup

On the exam the registry would already exist. Locally, a throwaway `registry:2` container stands in
for it:

```bash
docker run -d -p 5055:5000 --name cobalt-registry registry:2

mkdir -p ~/ckad/068/ledger-api
cat > ~/ckad/068/ledger-api/index.html <<'EOF'
ledger-api 4.2.0 ok
EOF
cat > ~/ckad/068/ledger-api/Dockerfile <<'EOF'
FROM busybox:1.36
COPY index.html /www/index.html
EXPOSE 8080
CMD ["httpd", "-f", "-p", "8080", "-h", "/www"]
EOF
```

## Solution

Build, then run the smoke-test container. The order in `-p` is `HOST:CONTAINER`, so it's
`18080:8080`, not the other way round. `EXPOSE` in the Dockerfile is documentation only and
publishes nothing by itself:
```bash
docker build -t ledger-api:4.2.0 ~/ckad/068/ledger-api
docker run -d --name ledger-smoke -p 18080:8080 ledger-api:4.2.0

sleep 1
curl -s http://localhost:18080/ > ~/ckad/068/smoke.txt
cat ~/ckad/068/smoke.txt
# ledger-api 4.2.0 ok
```

Push. `docker push` takes no destination argument: where an image goes is encoded in its *name*
(`localhost:5055/cobalt/ledger-api`), so each registry tag has to exist locally first. Two tags means
two `docker tag` + `docker push` pairs:
```bash
docker tag ledger-api:4.2.0 localhost:5055/cobalt/ledger-api:4.2.0
docker tag ledger-api:4.2.0 localhost:5055/cobalt/ledger-api:stable
docker push localhost:5055/cobalt/ledger-api:4.2.0
docker push localhost:5055/cobalt/ledger-api:stable
```
The second push prints `Layer already exists` for every layer: both tags point at the same image,
so the registry stores it once. `docker push --all-tags localhost:5055/cobalt/ledger-api` pushes
every local tag of that repository in one command, if you prefer.

Save. The archive records the *names you pass* to `docker save` (its `RepoTags`), not every tag the
image happens to have. The task wants the plain local names, so `ledger-api:stable` has to exist
before you save, and both names go on the same command line:
```bash
docker tag ledger-api:4.2.0 ledger-api:stable
docker save -o ~/ckad/068/ledger-api-bundle.tar ledger-api:4.2.0 ledger-api:stable
```

Check each result on its own. A successful `docker push` says nothing about the archive, and the
other way round:
```bash
docker ps --filter name=ledger-smoke --format '{{.Names}} {{.Image}} {{.Ports}}'
# ledger-smoke ledger-api:4.2.0 0.0.0.0:18080->8080/tcp, [::]:18080->8080/tcp

curl -s http://localhost:5055/v2/cobalt/ledger-api/tags/list
# {"name":"cobalt/ledger-api","tags":["4.2.0","stable"]}

tar -xOf ~/ckad/068/ledger-api-bundle.tar manifest.json
# [{"Config":...,"RepoTags":["ledger-api:4.2.0","ledger-api:stable"],"Layers":[...]}]
```

**The trap this scenario is built around:** every step also "works" with the wrong reference.
Saving `localhost:5055/cobalt/ledger-api:stable` instead of `ledger-api:stable` succeeds, but the
image loads back under the registry name. Saving only `ledger-api:4.2.0` succeeds, but the archive is
missing the second name. Pushing only one tag succeeds, and the other tag is simply absent from the
registry. None of these produce an error, so compare the registry's tag list and the archive's
`RepoTags` against what the task asked for.

## Cleanup

```bash
docker rm -f ledger-smoke cobalt-registry
docker rmi ledger-api:4.2.0 ledger-api:stable \
  localhost:5055/cobalt/ledger-api:4.2.0 localhost:5055/cobalt/ledger-api:stable
rm -rf ~/ckad/068
```

*Verified end-to-end with Docker 29 and a local registry:2 container on 2026-09-24.*
