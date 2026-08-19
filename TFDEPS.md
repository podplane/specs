# Terraform Dependencies Cache

> **STATUS**: Implemented

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

The shared cache contains provider filesystem-mirror data and versioned normalized module source under the `tf/` subdirectory of the configured Podplane dependency cache (normally the XDG cache location or `~/.podplane/cache/deps/tf/`). Downloads are additive: existing provider packages and normalized module versions remain available when newer versions are downloaded. Podplane does not retain the temporary `.terraform` tree.

## Terraform Generation

Provider and module source/version strings move to constants in `internal/tfgen/cluster.go`. A small shared helper renders provider requirements for both cluster generation and the temporary dependency root. Synthetic registry module blocks remain separate from config-dependent local cluster module blocks.

Generated cluster output always emits relative local module sources pointing to the exact resolved Nstance version in the shared cache, without `version` attributes. Registry module sources are used only by the temporary dependency-resolution root. The version embedded in each generated cluster's module path is its module-version record; no separate dependency manifest is maintained.

Generated provider addresses remain unchanged. One OpenTofu/Terraform CLI configuration in the shared dependency cache directs provider installation to the cached filesystem mirror and disables direct registry downloads. Each cluster directory retains its own provider lock file, so clusters using different provider versions can share the additive mirror safely. When rerunning an existing cluster, Podplane reads the module version from the generated source path and provider versions from `.terraform.lock.hcl`; if any exact package is absent from the shared cache, it retrieves that package automatically while registry access is available without changing the cluster's versions.

`podplane cluster upgrade` is the explicit version-changing workflow for an existing cluster. It downloads the latest module and provider versions allowed by Podplane's constraints, adds them to the shared cache, updates the generated module paths and cluster provider lock, regenerates managed Terraform files, and asks before applying. `--no-apply` performs the download and file updates without applying. The ordinary `cluster create` path remains restorative and never silently upgrades an existing stack.

## CI Usage

A CI job can:

1. select exact Podplane and OpenTofu/Terraform versions;
2. run `podplane deps tf --platform <target-platform>` while online;
3. persist or package the provider mirror, normalized modules, and lock information for later jobs; and
4. verify `cluster create --no-apply`, `init`, and `plan` with public registry access blocked.

Runtime cluster directories reference exact versioned module source and the provider filesystem mirror in the shared dependency cache. They retain only their provider lock file, and never copy dependency packages, duplicate the shared CLI configuration, or reuse a preinitialized `.terraform` directory.

## Non-goals

- Managing how cached dependencies or execution environments are packaged and distributed.
- Caching or creating OIDC infrastructure.
- Reusing `.terraform`, whose internal module state is engine- and configuration-dependent.
- Removing cloud API or Podplane VM dependency-mirror access required during deployment and bootstrap.
