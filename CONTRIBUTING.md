# Contributing

This repository is the official template registry for
[OpsKit](https://github.com/opskit-sh/opskit). A contribution here is a
manifest: data that turns a container image into a deployable service on
any OpsKit node.

## What a template is

A template is one YAML manifest describing one deployable: the service
image with its pinned version, how the service lives (its lifecycle,
network posture, and resource reservation), where its data persists,
the configuration inputs it accepts, the datastore links it declares,
and the operational tools and actions it ships. Templates are data
only — nothing ships inside the CLI binaries. A deployment is created
by id:

    opskit deploy postgres <name>

## A minimal manifest

A small datastore is the smallest honest template. Every field the
schema accepts is documented in the CLI repository (see below); this
example shows the shape and the meaning of each part:

```yaml
# The manifest dialect. The CLI's pre-flight validation accepts
# exactly "1.1".
schema_version: "1.1"

# The deployable id: lowercase letters, digits, and hyphens. One
# manifest per deployable id — the id is what the operator deploys.
id: mongodb
# Human-readable display name.
name: MongoDB
# The curated version pin this manifest ships. A newer version is a
# new registry commit, not a runtime choice.
version: "7.0"
# One catalog category: cms, ecommerce, automation, analytics,
# communication, storage, datastore, or developer.
category: datastore
# One-line summary for the catalog listing.
description: "Document database with a JSON-like data model."

# A single-configuration template declares a top-level `services`
# mapping. A template with runtime variants declares `stacks` instead
# (one named stack per variant, each with its own `services`) — never
# both, and never one file per variant.
services:
  db:                            # the service key names the container
    image: "mongo:7.0"           # pinned, multi-arch tag
    lifecycle: stateful          # service, stateful, daemon, job, or cron
    state: persistent            # stateless or persistent
    storage: persistent          # ephemeral or persistent
    network: internal            # ingress, internal, or none
    resources:                   # what deploy admission reserves
      memory: "256MiB"
    data_path: "/data/db"        # the deployment's data/ directory
                                 # bind-mounts here; at most one
                                 # service per deployment declares it

# Typed configuration inputs. Each entry becomes one line in the
# deployment's `.env`, in declaration order. Every entry needs either
# a `default` or a `generate` rule so deploy can always resolve it.
env:
  MONGO_INITDB_ROOT_USERNAME:
    description: "Root account created on first run."
    type: string
    default: admin
    required: false
  MONGO_INITDB_ROOT_PASSWORD:
    description: "Root account password created on first run."
    type: secret                 # secrets are generated at deploy time —
    generate: random_base64_64   # never given a plaintext default, never
    required: true               # passed via argv
```

Templates that need a datastore declare `links` (relational,
key_value, object_storage, or document) instead of embedding a database
container: datastores always run as their own deployments. Templates
may also declare `hostpaths` (editable config files mounted into
containers), `tools` (CLIs shipped by the image, run inside the
container as a non-root user), and `actions` (curated tool invocations
exposed to the operator).

## Schema reference and worked examples

- The schema reference is
  [docs/template-schema.md](https://github.com/opskit-sh/opskit/blob/main/docs/template-schema.md)
  in the CLI repository.
- The worked examples are the round-trip fixtures in the CLI
  repository under
  [tests/fixtures/roundtrip/](https://github.com/opskit-sh/opskit/tree/main/tests/fixtures/roundtrip):
  `postgres/template.yaml` is the stackless shape (a single service
  and one secret), and `wordpress/template.yaml` is the stacks shape
  (two runtime variants of one template) together with links,
  hostpaths, tools, actions, and a typed env entry.

The validator shipped in the CLI is the gate: a manifest that the
current `opskit` build rejects at deploy pre-flight will not be
merged. Validate against the CLI you use, not against memory.

## Catalog gates

A manifest is accepted into `official/` when it meets all of these:

1. **Valid schema.** It passes the CLI's 1.1 schema validation with
   zero violations.
2. **Non-root execution.** Every workload runs rootless on the node;
   every declared tool runs as a non-root user.
3. **Multi-arch images.** Every image tag resolves for the node
   architectures the catalog supports (amd64 and arm64).
4. **One manifest per deployable id.** The catalog never ships two
   manifests for the same deployable; `index.json` carries exactly one
   entry per manifest.
5. **No variant forks.** Runtime variety lives inside one manifest as
   named `stacks`, never as a second manifest forked from the first.
6. **No generic framework skeletons.** A template deploys a running
   application, not an empty framework or web-server runtime awaiting
   code.

Secret hygiene is part of gate 1: an entry resolves at deploy time from
its declared `default`, from a `--set` override of that default, or — for
secrets — from a `generate` rule (`random_hex_16`, `random_hex_32`,
`random_base64_64`, `uuid`) whose value is generated at deploy time. A
plaintext default on a secret is a rejection.

## Trying a template on your own node

You do not need to merge anything to deploy your own manifest. Place it
in your user cache:

    ~/opskit/.templates/user/<id>.yaml

and deploy it by id:

    opskit deploy <id> <name>

Deploy resolves the official cache first
(`~/opskit/.templates/official/<id>.yaml`), then the user cache, so a
personal manifest with a fresh id never collides with the catalog.

## Submitting a pull request

1. Fork the repository and create a branch.
2. Add your manifest as `official/<id>.yaml` and one entry to
   `index.json`. Bumping a version is a normal commit: update the
   manifest's `version` and image pin together.
3. Check it locally: place the manifest in your user cache as above
   and run a deploy against it.
4. Open the pull request.

CI parses every YAML file in the repository and must pass. Review
checks the catalog gates above — the maintainers will validate your
manifest against the shipped CLI, verify the image tags are real
multi-arch pins, and confirm the id is not a fork or a skeleton.

## License

MIT for everything in this repository; see [LICENSE](LICENSE).
