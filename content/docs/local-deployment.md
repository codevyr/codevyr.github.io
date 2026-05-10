---
title: "Local Deployment with Docker Compose"
description: "Run Codevyr locally, index Kubernetes, and upload the index to askld"
weight: 120
---

This tutorial walks you through running Codevyr locally (askld API, UI, and Postgres), generating a Kubernetes index with askl-golang-indexer, and uploading it into your local askld.

## Prerequisites

- Docker and Docker Compose
- make
- git
- Enough disk space for the Kubernetes repository and index output

## 1) Clone the required repos

Pick a working directory and clone the repositories you need.

```sh
CODEVYR_WORK="$HOME/codevyr-work"
mkdir -p "$CODEVYR_WORK"
cd "$CODEVYR_WORK"

git clone https://github.com/codevyr/docker-compose.git compose
git clone --depth 1 https://github.com/kubernetes/kubernetes.git
```

The `compose` repo contains the Docker Compose configuration for the local stack. The Kubernetes repo provides source code for indexing and analysis.

## 2) Start the stack

From the `compose` repo (use the local override to allow HTTP tokens and local CORS):

```sh
cd "$CODEVYR_WORK/compose"
make deploy-local
```

Confirm the services are running:

```sh
docker compose ps
```

You should see `askld`, `codevyr`, and `db` running.

If you are indexing a large project, increase the upload limit in `compose.local.yaml` via `ASKL_MAX_UPLOAD_BYTES` (the default in the local override is 512 MB).

## 3) Generate a Kubernetes index

Use the helper script from the `compose` repo to run the indexer and write the output into a local directory:

```sh
mkdir -p "$CODEVYR_WORK/index"
./bin/generate_index_docker.sh "$CODEVYR_WORK/kubernetes" "$CODEVYR_WORK/index/index-kubernetes.pb" \
  --include-git-files \
  --project kubernetes \
  --path 'cmd/*'
```

The helper script forwards indexer flags as-is, runs the container with the working directory set to the project root, and prefixes `--path` values with `/<project>` automatically. Quote globs like `'cmd/*'` so the host shell does not expand them. Use repeated `--path` flags to index multiple commands in a single index.

By default, the indexer includes only files compiled by the Go compiler. The `--include-git-files` flag adds all files tracked by git, which is useful to include additional files that are not part of the Go build.

The default logging level is `error`. If you want to see more details about the indexing process, add `--log-level debug` to the command. Currently, the indexer prints large amounts of warning messages, due to partial support for Go language features. These warnings can be safely ignored for now.

The `generate_index_docker.sh` script disables cgo, because it is not supported in the current indexer.

Copy the host index file into the `askld` container so it can be uploaded:

```sh
docker compose cp "$CODEVYR_WORK/index/index-kubernetes.pb" askld:/tmp/index-kubernetes.pb
```

## 4) Create an API token (inside askld)

The admin endpoint is only available inside the container, so use `docker compose exec`:

```sh
ASKL_TOKEN=$(
  docker compose exec -T askld /usr/local/bin/askld auth --port 8080 create-api-key \
    --email "you@example.com" \
    --name "local" \
    --json | jq -r .token
)
```

If you do not have `jq`, run the command without `| jq -r .token` and copy the `token` value manually.

## 5) Upload the index (inside askld)

The index file is now inside the `askld` container at `/tmp/index-kubernetes.pb`:

```sh
docker compose exec -T askld /usr/local/bin/askld index upload \
  --file /tmp/index-kubernetes.pb \
  --url 127.0.0.1:8080 \
  --token "$ASKL_TOKEN"
```

If you have `askld` available on the host, you can run the same command directly without `docker compose exec`.

Uploads can take a while for large indexes. The CLI default timeout is 180 seconds; pass `--timeout` if you need more time. If you see an upload size error, increase `ASKL_MAX_UPLOAD_BYTES` in `compose.local.yaml`.

Verify the project is listed:

```sh
docker compose exec -T askld /usr/local/bin/askld index list-projects \
  --url 127.0.0.1:8080 \
  --token "$ASKL_TOKEN"
```

## 6) Open the UI and verify

Open the UI in your browser:

- [http://localhost:3000](http://localhost:3000)

You should see the Kubernetes project available. Try a simple Askl query like:

[main](http://localhost:3000/#q=AIBwTgpghgtgRgGwgKGASwOYDsD2kAUIUAxgNZQYQC8ARHAK5oIAuaWNAlKprgUWRWo0AZjGadu2PBEIlylWsRxZmEAB7iu6KXzmDaOAM4TtvGf3lCEODCZ7TZAhTTD0VaGBDs7ze521UwLCgEbzNHS1o0HHpWUK17XSchDBwEKCxbBJ8I-RpSAA5DADpogHpSayyAbmRUcGh4JAACMoAqZoAFSFhECGaoEBAENAhDZuYAC37hNOsAdzYMZoA3CDA4cYxrOBCEAE9i5ray+rAcACsIYmZ8fPo4dawIVWMOWuQaGCg2GmagA)

```askl
"main"
```

## Cleanup

To stop the stack:

```sh
docker compose down
```

To remove the Postgres data volume as well:

```sh
docker compose down -v
```
