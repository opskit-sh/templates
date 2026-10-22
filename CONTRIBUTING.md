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

## Authoring a datastore template's provisioning recipe

A datastore template — a template other deployments bind through
their `links` — declares its own `provisioning:` block: the recipe
the deploy platform executes inside the datastore's running container
to create one consumer's logical entities. The platform runs the
declared recipe generically; nothing about your datastore engine
ships inside the CLI binaries, so the recipe is where your engine's
own administrative knowledge lives.

```yaml
provisioning:
  # What the recipe provisions. `relational` recipes create a
  # consumer's logical database and login user; `object_storage`
  # recipes create a per-consumer bucket.
  kind: relational
  # The command the platform runs inside the datastore's running
  # container, through `docker compose exec -T`. Entries may
  # reference the container's environment — its own shell expands
  # them there — so the datastore's superuser credential never
  # appears in any process argv: pass the variable NAME and expand
  # it in the command, or read it in the script's tool.
  exec: ["sh", "-c", "admin-cli --stdio --password-file \"$ADMIN_PASSWORD_FILE\""]
  # The text piped to that command's standard input. The platform
  # substitutes the closed placeholder set — nothing else:
  #   {database}  the consumer's logical database name
  #   {user}      the consumer's login user name
  #   {password}  a freshly generated login credential for the user
  #   {bucket}    the consumer's derived bucket name
  # The script must contain no `$` and no other brace expression:
  # the generated credential rides this text on standard input, so
  # it never appears in any argv, log, or record.
  script: |
    create-login "{user}" with-password '{password}';
    create-logical "{database}";
```

Authoring rules of thumb:

- **Make the recipe idempotent.** Deploy re-runs it on every bind;
  create-if-absent forms and re-issuable password statements keep
  re-binding safe. A relational recipe should end with the login
  password set to the freshly generated `{password}`, so a re-bind
  rotates the credential the consumer's environment splices.
- **Scope what the recipe grants.** A shared instance serves many
  consumers: create the login user so it reaches the one logical
  database named `{database}` and nothing else.
- **Keep secrets out of argv.** The `exec` entries may carry `$`
  because the container's own shell expands them there; the host-side
  argv carries variable names only. The generated `{password}` rides
  the `script` text on standard input — never put it in `exec`.
- **Object-storage recipes** also name the two environment entries
  the instance's generated identity pair lands in
  (`access_key_env` / `secret_key_env`); the same values flow to the
  consumer through its link's `access_key`/`secret_key` mappings.
  The bucket name substitutes as `{bucket}`, hyphenated the way S3
  naming rules require. The identity names must be uppercase
  environment-variable shaped and must not name any `env` entry the
  template itself declares — the datastore's own secrets are not the
  consumer identity.
- **Declare the recipe on a single-service, datastore-class
  template.** The recipe runs inside the one service the manifest
  declares; a `datastore`- or `storage`-category template with
  exactly one service is the shape that carries one.

An object-storage recipe:

```yaml
provisioning:
  kind: object_storage
  access_key_env: STORE_ACCESS_KEY
  secret_key_env: STORE_SECRET_KEY
  exec: ["sh", "-c", "store-shell"]
  script: 'make-bucket {bucket}'
```

A consumer's relational or object_storage link whose bound template
declares no recipe of the matching kind is refused at deploy
pre-flight — the refusal names the link and the template, so the
catalog gate is: ship the recipe with the datastore it provisions.
The block is deploy-time contract: it is never rendered into
`compose.yml` or the deployment record.

## Authoring a datastore template's dump recipe

A datastore template that can be backed up also declares its own
`dump:` block: the recipe the platform executes inside the datastore's
running container to take one consumer's logical database out — and to
import one back. As with the provisioning recipe, the platform runs the
declared recipe generically; your engine's own dump tool, its own
flags, and its own restore counterpart live in the template, never in
the CLI binaries.

```yaml
dump:
  # The command the platform runs inside the datastore's running
  # container to write the dump. It must write the dump to the file
  # /tmp/opskit-dump.dump — the platform's staging path — and the
  # platform copies that file out of the container with the engine's
  # own copy-out command, so the bytes arrive exactly as your tool
  # wrote them. A binary archive must never travel through a text
  # channel.
  #
  # The platform substitutes the closed placeholder set — nothing else:
  #   {database}  the consumer's logical database name
  #   {user}      the consumer's login user name
  #   {password}  the consumer's generated login credential
  # The entries may reference the container's environment — its own
  # shell expands them there — so the datastore's superuser credential
  # never appears in any process argv.
  exec: ["sh", "-c", "engine-dump --database {database} --file /tmp/opskit-dump.dump"]
  # The import counterpart: the command that loads a dump back into the
  # logical database the platform names. The platform copies the
  # archived dump into the container at /tmp/opskit-dump.dump before it
  # runs this command.
  restore_exec: ["sh", "-c", "engine-restore --database {database} --file /tmp/opskit-dump.dump"]
  # The text piped to the restore command's standard input — optional.
  # It carries the same closed placeholder set and no `$`.
  restore_script: |
    grant-all-to "{user}";
```

Authoring rules of thumb:

- **Dump one logical database, never the instance.** The recipe
  targets the consumer's own database: a shared instance serves many
  consumers, and the backup of one must never read — or drop — the
  others.
- **Keep the archive owner-free.** An archive that carries ownership
  and access-control statements restores into one database under one
  owner; an archive without them restores into any database. Grant the
  consumer's user what it needs in the restore step instead.
- **Cite the upstream documentation.** The tool's name, its flags, and
  its behavior come from your engine's own documentation — link it in
  the manifest's comments, the way the shipped recipes do, so a
  reviewer can check the recipe against its source.
- **Declare the recipe on a single-service, datastore-class
  template**, exactly like the provisioning recipe: the commands run
  inside the one service the manifest declares.

The block is deploy-time contract: it is never rendered into
`compose.yml` or the deployment record.

## Authoring a datastore template's drop recipe

A datastore template that provisions entities also declares its own
`drop:` recipe, inside the `provisioning:` block: the command the
platform executes inside the datastore's running container to remove
one consumer's logical entities again — its logical database and its
login user. The platform runs the declared recipe generically, the way
it runs the provisioning and the dump recipes; your engine's own
drop commands and their own flags live in the template, never in the
CLI binaries.

```yaml
provisioning:
  kind: relational
  exec: ["sh", "-c", "admin-cli --stdio"]
  script: |
    create-login "{user}" with-password '{password}';
    create-logical "{database}";
  # The drop recipe: the same closed placeholder set as the create
  # recipe — {database}, {user}, {password}, {bucket} — and the same
  # no-dollar rule for the script.
  drop:
    # The command the platform runs inside the datastore's running
    # container, through `docker compose exec -T`.
    exec: ["sh", "-c", "admin-cli --drop --database {database} --user {user}"]
    # The text piped to that command's standard input — optional. It
    # carries the same closed placeholder set and no `$`.
    script: |
      drop-logical "{database}";
      drop-login "{user}";
```

Authoring rules of thumb:

- **Drop one consumer's entities, never the instance.** The recipe
  targets the consumer's own database and its own login user: a shared
  instance serves many consumers, and the decommissioning of one must
  never touch the others.
- **The drop is not the create, reversed.** The two differ by dialect
  and by dependency order — the login user usually owns the logical
  database, so the database goes first — and the platform never
  derives one from the other. Declare the drop you mean.
- **Make the drop idempotent.** A decommissioning may run twice — a
  failed one, then a retry — so the recipe must tolerate entities
  that are already gone: the shipped recipes use the engine's own
  "if exists" forms and their own conditional forms.
- **Cite the upstream documentation.** The command's name, its flags,
  and its behavior come from your engine's own documentation — link it
  in the manifest's comments, the way the shipped recipes do, so a
  reviewer can check the recipe against its source.
- **Declare the recipe on a single-service, datastore-class
  template**, exactly like the provisioning and the dump recipes: the
  commands run inside the one service the manifest declares.

A datastore template that provisions entities and declares no drop
recipe cannot be decommissioned cleanly: the platform refuses the
delete rather than leave the consumer's logical entities behind, so
the catalog gate is: ship the drop recipe with the datastore it
provisions.

The block is deploy-time contract: it is never rendered into
`compose.yml` or the deployment record.

## Schema reference and worked examples

- The schema reference is
  [docs/template-schema.md](https://github.com/opskit-sh/opskit/blob/main/docs/template-schema.md)
  in the CLI repository.
- The worked examples are the round-trip fixtures in the CLI
  repository under
  [tests/fixtures/roundtrip/](https://github.com/opskit-sh/opskit/tree/main/tests/fixtures/roundtrip):
  `postgres/template.yaml` is the stackless shape (a single service,
  one secret, and a declared provisioning recipe — the same bytes the
  registry publishes), and `wordpress/template.yaml` is the stacks
  shape (two runtime variants of one template) together with links,
  hostpaths, tools, actions, and a typed env entry.

The validator shipped in the CLI is the schema gate. This
repository's CI parses every YAML file — it does not run the
validator. The CLI repository's own CI validates its own round-trip
fixtures, not this repository's manifests: every manifest here is
validated with the shipped `opskit` build in maintainer review,
before merge. A registry CI job that invokes the published validator
on every manifest lands with the first release. Until then: validate
against the CLI you use, not against memory.

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
