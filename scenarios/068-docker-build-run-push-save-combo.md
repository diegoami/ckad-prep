# 068 — Docker: build, name a container, push under a username, save under a different tag

**Domain:** Application Design and Build · **Difficulty:** Medium

`059` only builds, tags and saves, and `011` builds, pushes and runs. This one chains all four
steps, and each step asks for a different tag of the same image. What's being tested is keeping
track of which reference each command needs.

## Task

> The `kilimanjaro` team keeps the Dockerfile for its `trail-api` service in
> `~/ckad/068/trail-api/`. Using Docker:
>
> 1. Build an image `trail-api:1.0.1` from it.
> 2. Start a detached container named `trail-api` from that image.
> 3. Push the image to the team registry at `localhost:5000`, under the team's path `kilimanjaro`,
>    tagged `latest`.
> 4. Save the image, tagged `v2`, to the archive `~/ckad/068/trail-api-v2.tar`.

## Documentation

What to look up: **Images**, on the Kubernetes side. Build, tag, push and save are covered by
Docker's own documentation, not Kubernetes'.
- <https://kubernetes.io/docs/concepts/containers/images/> — image reference syntax, relevant once
  any of these tags gets referenced from a Pod spec.

## Setup

On the exam the registry would already exist. Locally, a throwaway `registry:2` container stands in
for it:

```bash
docker run -d -p 5000:5000 --name team-registry registry:2

mkdir -p ~/ckad/068/trail-api
cat > ~/ckad/068/trail-api/Dockerfile <<'EOF'
FROM busybox:1.31.0
CMD ["sh", "-c", "echo trail-api running; sleep 3600"]
EOF
```

## Solution

Build and tag in one step, then run a **named** container from it. "Start a container named
`trail-api`" is a separate requirement from building the image, and easy to forget:
```bash
docker build -t trail-api:1.0.1 ~/ckad/068/trail-api
docker run -d --name trail-api trail-api:1.0.1
```

Push under the team's registry path. This needs a *second* tag, because
`localhost:5000/kilimanjaro/trail-api:latest` doesn't match `trail-api:1.0.1` in registry, path or
tag:
```bash
docker tag trail-api:1.0.1 localhost:5000/kilimanjaro/trail-api:latest
docker push localhost:5000/kilimanjaro/trail-api:latest
```

Save under **yet another** tag (`v2`) that was never pushed and never ran. This is the step most
likely to be skipped or fumbled, since it's easy to assume "the image" means whichever tag was used
last:
```bash
docker tag trail-api:1.0.1 trail-api:v2
docker save --output ~/ckad/068/trail-api-v2.tar trail-api:v2
```

Check each of the three references on its own. A passing `docker push` says nothing about whether
`docker save` used the right reference, and the other way round:
```bash
docker ps --filter name=trail-api --format '{{.Names}} {{.Image}} {{.Status}}'
# trail-api trail-api:1.0.1 Up ...

curl -s http://localhost:5000/v2/kilimanjaro/trail-api/tags/list
# {"name":"kilimanjaro/trail-api","tags":["latest"]}

tar -xOf ~/ckad/068/trail-api-v2.tar manifest.json
# [{"Config":...,"RepoTags":["trail-api:v2"],"Layers":[...]}]
```

**The trap this scenario is built around:** the task names *three different tags* for the *same
underlying image* (`1.0.1` built, `latest` pushed, `v2` saved) and never says to retag before push
or save. That's implied by "push it... tagged latest" and "save it, tagged v2" each being
requirements of their own. Building once and reusing `trail-api:1.0.1` for the push or the save
without retagging produces a push or save that runs without error but under the wrong reference.
It looks exactly like success unless you check the tag on the pushed or saved artifact against what
was asked for.

## Cleanup

```bash
docker rm -f trail-api team-registry
docker rmi trail-api:1.0.1 trail-api:v2 localhost:5000/kilimanjaro/trail-api:latest
rm -rf ~/ckad/068
```

*Verified end-to-end with Docker 29 and a local registry:2 container on 2026-09-23.*
