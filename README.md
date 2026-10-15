# OpsKit Templates

The official template registry for [OpsKit](https://github.com/opskit-sh/opskit).
Deployed on GitHub as **[`opskit-sh/templates`](https://github.com/opskit-sh/templates)**.

A template is one YAML manifest describing a deployable: the service images with
their pinned versions, the configuration inputs it accepts with their defaults,
and its operational actions. Templates are fetched by id on demand — they never
ship inside the binaries:

    opskit deploy postgres <name>

The manifest schema is documented in the
[OpsKit repository](https://github.com/opskit-sh/opskit).

## Layout

```text
official/    one manifest per template id, curated and security-reviewed
index.json   the browse index — one entry per official manifest
```

## Versioning

This repository **is** the version source. Git tags are release pins; the CLI
fetches a manifest at a ref (tag or commit). Image tags are the manifest's
curated pins — a newer version is a new registry commit, not a runtime choice.
Manifests carry no internal update machinery.

## Contributing

The template-authoring quickstart lives in [CONTRIBUTING.md](CONTRIBUTING.md).
Pull requests that add or change an official manifest keep the catalog gates: a
valid 1.1 manifest schema, non-root execution for every service, multi-arch
images (amd64 and arm64), one manifest per deployable id, no variant forks
(runtime variety ships inside one manifest as stacks), and no generic framework
skeletons.

## License

MIT for everything in this repository. Manifests are data — lift and reuse them
freely; the notice in [LICENSE](LICENSE) is the only condition. The
[code repositories](https://github.com/opskit-sh/opskit) are Apache-2.0.
