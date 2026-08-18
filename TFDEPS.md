# Terraform Dependencies Cache

> **STATUS**: Draft

## Goal

Allow Podplane cluster OpenTofu/Terraform providers and modules to be downloaded once, reused across cluster deployments, and used without public Terraform/OpenTofu registry access.

## Command

`github.com/podplane/podplane` adds:

```shell
podplane deps tf [--platform OS_ARCH]...
```

The command uses the existing Terraform executable selection: `PODPLANE_TF_CMD` when set, otherwise `tofu`, then `terraform`. `--platform` is optional, repeatable, and defaults to the engine's current platform; cross-platform CI jobs should pass values such as `linux_amd64` explicitly.

The command:

1. generates a temporary minimal root module containing every provider and Nstance module required by generated cluster infrastructure;
2. runs `init -backend=false` to resolve modules and transitive provider requirements;
3. runs `providers mirror` for each requested platform;
4. normalizes the downloaded Nstance source into Podplane's dependency cache; and
5. retains the resolved provider lock information needed by later deployments.

The cache contains provider filesystem-mirror data and normalized module source under `deps/tf/`. Podplane does not retain the temporary `.terraform` tree.

## Terraform Generation

Provider and module source/version strings move to constants in `internal/tfgen/cluster.go`. A small shared helper renders provider requirements for both normal cluster generation and the temporary dependency root. Synthetic module blocks may remain separate from config-dependent cluster module blocks.

Normal `tfgen` output remains unchanged: registry module sources and existing golden `.expected.tf` files continue to support ordinary and manual testing. An explicit cached-dependencies mode, such as `cluster create --tf-deps`, instead materializes cached Nstance modules into a managed directory beside the generated files and emits relative local module sources without `version` attributes.

Cached-dependencies mode changes only module source paths. Generated provider addresses remain unchanged, while OpenTofu/Terraform CLI configuration directs provider installation to the cached filesystem mirror and disables direct registry downloads.

## CI Usage

A CI job can:

1. select exact Podplane and OpenTofu/Terraform versions;
2. run `podplane deps tf --platform <target-platform>` while online;
3. persist or package the provider mirror, normalized modules, and lock information for later jobs; and
4. verify `cluster create --no-apply --tf-deps`, `init`, and `plan` with public registry access blocked.

Runtime cluster directories use copied local module source and the provider filesystem mirror. They never reuse a preinitialized `.terraform` directory.

## Non-goals

- Managing how cached dependencies or execution environments are packaged and distributed.
- Caching or creating OIDC infrastructure.
- Reusing `.terraform`, whose internal module state is engine- and configuration-dependent.
- Removing cloud API or Podplane VM dependency-mirror access required during deployment and bootstrap.
