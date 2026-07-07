# Podplane Local Git Cache and Server

## Goal

Local Podplane clusters should be able to run Flux against cached Git repositories served by the existing Podplane local HTTPS server. This lets local VMs reconcile `platform-components` without depending on GitHub at runtime and gives Podplane developers a clean way to test local `components` changes before pushing/tagging a release.

## Non-goals

- Do not run a separate Git server process.
- Do not require the native `git` binary at Podplane CLI runtime.
- Do not implement general-purpose Git hosting, writes, auth, pull requests, or repository management.
- Do not create one clone per version; one repo cache should hold many refs.

## Cache layout

Cache Git repositories under the existing dependency cache:

```text
~/.podplane/cache/deps/git/<repo-path>.git
```

Examples:

```text
~/.podplane/cache/deps/git/github.com/podplane/components.git
~/.podplane/cache/deps/git/github.com/acme/components.git
~/.podplane/cache/deps/git/components.git
```

Each cached repo is a bare/mirror-style Git repository. Tags and branches are refs within the same repo. Do not copy each tag or branch into separate directories when cloning `https://github.com/podplane/components` into the `~/.podplane/cache/deps/git/` tree.

No additional metadata storage is required in the cache - the source identity is encoded by the directory path, and versions are encoded by Git refs.

### Cache namespace rule

Cache entries must use exactly one of these shapes:

```text
~/.podplane/cache/deps/git/<fqdn>/<owner-or-path>/<repo>.git
~/.podplane/cache/deps/git/<repo>.git
```

FQDN-rooted paths represent remote repos. Flat `<repo>.git` paths represent local-only repos.

Valid:

```text
~/.podplane/cache/deps/git/github.com/podplane/components.git
~/.podplane/cache/deps/git/git.example.com/platform/components.git
~/.podplane/cache/deps/git/components.git
```

Invalid:

```text
~/.podplane/cache/deps/git/local/components.git
~/.podplane/cache/deps/git/dev/components.git
```

Use the flat path for the components repo development snapshot:

```text
~/.podplane/cache/deps/git/components.git
```

Pros:

- Simple and short for the common local-only workflow.
- Strong, easy-to-explain namespace rule: first path segment is either a FQDN or the repo file itself.
- Avoids pseudo-namespaces such as `local/` or `dev/` that look like hosts but are not.

Cons:

- Local-only repo names must be globally unique within the local Git cache.

## Local server URL layout

The Podplane local HTTPS server serves cached Git repos under `/git/`:

```text
https://<local-server>/git/github.com/podplane/components.git
https://<local-server>/git/github.com/acme/components.git
https://<local-server>/git/components.git
```

The URL path after `/git/` maps directly to `~/.podplane/cache/deps/git/`.

Serving should use a smart Git HTTP implementation embedded in the CLI process. Isolated testing showed that static HTTP serving works with native Git clients, but fails with go-git clients as used by Flux. `go-git/v6/backend` worked for full and shallow clone tests and should be used for implementing a smart HTTP server.

For local VMs, do not rely on `.localhost`, VM `/etc/hosts`, CoreDNS overrides, Kubernetes `Service`/`Endpoint` objects, or Flux chart customization to expose this Git server to Flux. Flux runs in a pod and resolves names through CoreDNS; node `/etc/hosts` entries are not visible to normal pods. Instead, local Flux sources should use the local VM node IP and a stable VM-local port, with the VM forwarding that stable port to the current Podplane local HTTPS server port.

The default QEMU user-mode network address for the single local VM is expected to be stable as `10.0.2.15`, with the host machine reachable from the VM as `10.0.2.2`. The CLI should still derive or confirm the node IP where practical rather than scattering the literal across unrelated code.

Example local VM URL shape:

```text
https://10.0.2.15:<stable-git-port>/git/components.git
https://10.0.2.15:19443/git/components.git
```

The stable VM-local port is served by `socat` inside the VM:

```text
Flux source-controller pod
  -> https://10.0.2.15:19443/git/components.git
  -> VM-local socat listener
  -> 10.0.2.2:<current-local-https-port>
  -> Podplane local HTTPS server /git/components.git
```

`socat` must pass TCP through without terminating TLS. The Podplane local HTTPS certificate must include the local VM node IP, for example `10.0.2.15`, as an IP SAN so Flux/go-git can validate `https://10.0.2.15:19443/...`.

## Components manifest

The components dependency manifest should declare the Git source Flux should use for that manifest version. Use the same source shape as cluster config:

```json
{
  "components": {
    "version": "1.2.3",
    "source": {
      "url": "https://github.com/podplane/components.git",
      "ref": {
        "semver": "^1.2.3"
      }
    }
  }
}
```

Support exactly one ref selector:

```json
{ "source": { "url": "https://github.com/podplane/components.git", "ref": { "semver": "^1.2.3" } } }
{ "source": { "url": "https://github.com/podplane/components.git", "ref": { "branch": "main" } } }
{ "source": { "url": "https://github.com/podplane/components.git", "ref": { "tag": "v1.2.3" } } }
{ "source": { "url": "https://github.com/podplane/components.git", "ref": { "commit": "abc123" } } }
```

Rationale:

- A local manifest file passed to `podplane deps download --components <path>` can tell the CLI which Git repo/ref to cache.
- Forked or downstream components manifests can point at their own Git source without CLI code changes.
- Repository moves or hosting changes can be represented in the manifest.

Do not add source override flags initially. Development can edit/use a local manifest. Add only a skip flag:

```sh
podplane deps download --skip-components-git
```

## `podplane deps download` behavior

When components Git caching is not skipped:

1. Load the components manifest from the normal source or `--components <path>`.
2. Read `components.source.url` and exactly one of `source.ref.semver`, `source.ref.tag`, `source.ref.branch`, or `source.ref.commit`.
3. Clone/fetch the repo into `~/.podplane/cache/deps/git/<repo-path>.git` using go-git.
4. Ensure the requested ref/commit is present in the cache.

For official releases, cache `github.com/podplane/components.git` and the manifest semver selector. For development manifests, cache the repo/ref declared by that manifest.

## Netsy seed requirements

Netsy seed snapshots already contain the Flux `GitRepository` for `podplane-components` and the `platform-components` `HelmRelease`. 

`netsyseed` should remain the place that interpolates the components Git source into that seed data.

No new seed object shape is required. The desired output is still the normal Flux source fields:

```yaml
spec:
  url: https://<components-git-url>
  secretRef:
    name: components-git-auth
  ref:
    semver: ^1.2.3
```

`secretRef` is optional and maps directly to Flux. It contains only `name` and
references a Kubernetes Secret in the `platform-components` namespace. Podplane
does not route production Flux Git auth through Podplane Secrets or Secrets Store
CSI because Flux reads Kubernetes Secrets directly during bootstrap.

or, for local development:

```yaml
spec:
  url: https://10.0.2.15:19443/git/components.git
  secretRef:
    name: podplane-components-git
  ref:
    branch: local-dev
```

Local development remains HTTPS. Bootstrap/operator-owned paths may create the
local CA Secret as local/dev bootstrap state. Reusable seed templates must not
carry dangling `secretRef`s: `seedgen` strips them, and `netsyseed` only updates
the Flux `GitRepository.spec.secretRef` from cluster config/local source data. It
must never create or inject Kubernetes Secret records.

Production cluster creation should continue using the existing cluster config and `terraform-provider-podplane` source override path. No production seed or OpenTofu/Terraform changes are expected unless that existing override path proves insufficient.

## Local VM behavior

For local VMs with seeded components, `podplane local start` should point the generated/interpolated components source at the VM-local `socat` listener (which TCP-forwards to the local HTTPS server) when the cached Git repo is available:

```json
{
  "components": {
    "source": {
      "url": "https://10.0.2.15:19443/git/github.com/podplane/components.git",
      "ref": {
        "semver": "^1.2.3"
      },
      "secretRef": {
        "name": "podplane-components-git"
      }
    }
  }
}
```

The `socat` listener should be configured only for local-provider VMs from the local VM user-data path. Note that the `socat` binary dependency is included in the Debian base image already. It should listen on a stable port on the VM/node address reachable from pods and forward to the current host-side Podplane local HTTPS server port at `10.0.2.2:<https-port>`. On `podplane local start`, after the local server is ensured and its current HTTPS port is known, the CLI should update/restart the VM-local `socat` service alongside the existing local runtime port updates, to ensure it's always forwarding to the latest correct local HTTPS server port.

The implementation must leave the existing local HTTPS server port wiring alone. The host-side HTTPS listener may remain dynamically allocated; the stable interface for Flux is the VM-local `socat` port.

`podplane local status --json` should expose the effective local components source so development Makefile targets can consume runtime facts without hardcoding the local VM node IP. The JSON shape should mirror the components source shape:

```json
{
  "cluster_id": "dev",
  "running": true,
  "vm": {
    "provider": "qemu",
    "node_ip": "10.0.2.15",
    "forward_https_port": 19443
  },
  "local_server": {
    "running": true,
    "http_port": 12345,
    "https_port": 23456
  },
  "components": {
    "source": {
      "url": "https://10.0.2.15:19443/git/components.git",
      "ref": {
        "branch": "local-dev"
      },
      "secretRef": {
        "name": "podplane-components-git"
      }
    }
  }
}
```

For `--components=none`, no components source is needed until bootstrap is run manually.

## Cluster create behavior

`podplane cluster create` should keep using the components source declared in cluster config and passed through the existing Terraform provider variables. The local Git cache/server is primarily for local VM and local development workflows. Production clusters should not be pointed at the local server unless a user explicitly configures that source.

For production forks or private components repos, users should configure the components source in cluster config so the generated infrastructure passes it through to the seed interpolation path, for example:

```jsonc
{
  "cluster": {
    "components": {
      "source": {
        "url": "https://github.com/acme/components.git",
        "ref": {
          "semver": "^1.2.3"
        },
        "secretRef": {
          "name": "components-git-auth"
        }
      }
    }
  }
}
```

For private repositories, the user is responsible for creating
`Secret/components-git-auth` in `platform-components`; Podplane only wires the
Flux reference.

## Components repo development workflow

Add a `git-sync` target to `github.com/podplane/components`.

Purpose: snapshot the current working tree into the Podplane Git cache at:

```text
~/.podplane/cache/deps/git/components.git
```

and publish it on a predictable branch:

```text
refs/heads/local-dev
```

Then local bootstrap can use:

```sh
make recommended
```

When the local Git cache contains `~/.podplane/cache/deps/git/components.git`, the components repo `make recommended` target should default `PLATFORM_GIT_REPOSITORY_URL` to the VM-local HTTPS forwarding endpoint and `PLATFORM_GIT_REPOSITORY_BRANCH` to `local-dev`. Users can still override either environment variable explicitly for GitHub branches, forks, or other sources.

For local VM development, `<local-server>` concretely means the VM-local HTTPS forwarding endpoint, not the host-side dynamic HTTPS listener. The expected default URL shape is:

```text
https://10.0.2.15:19443/git/components.git
```

The stable port must match the `socat` service configured by the Podplane CLI/user-data path. `make recommended` may use `podplane local status --json` and read `.components.source.url` / `.components.source.ref.branch` when available instead of hardcoding the local VM node IP.

The `git-sync` target should include uncommitted working-tree changes, similar in spirit to the existing Kind `dev-sync` target.

## Implementation plan

### Repository: `github.com/podplane/podplane`

1. Add components manifest `components.source` fields and validation.
2. Extend deps cache helpers with `deps/git/<repo-path>.git` path derivation.
3. Implement go-git clone/fetch for manifest-declared components Git source.
4. Add `--skip-components-git` to `podplane deps download`.
5. Embed a `/git/` smart HTTP handler in the existing local HTTPS server using `go-git/v6/backend`.
6. Extend the local HTTPS certificate generation to include the local VM node IP as an IP SAN for local Git URLs.
7. In local-provider user-data, install/configure a systemd-managed `socat` TCP forwarder on stable VM-local HTTPS proxy port `19443`. The forwarder should pass through TCP to `10.0.2.2:<current-local-https-port>` and must not terminate TLS. Critically, any changes/additions to the user-data.sh file for this use case should ONLY sit inside a `{{if eq .Provider.Kind "local"}}` condition.
8. Update `podplane local start` to refresh/restart the VM-local `socat` service with the current local HTTPS server port whenever it performs the existing local VM runtime port updates for Netsy env vars (which is essentially just the `local start` command).
9. Add `podplane local status --json` output including `.vm.node_ip`, `.vm.forward_https_port`, and `.components.source.url` / `.components.source.ref.branch`.
10. Update local cluster source interpolation to prefer VM-local HTTPS forwarding URLs when cached Git source is available.
11. Add focused tests for manifest parsing, cache path derivation, skip behavior, local source URL rendering, local status JSON rendering, certificate SANs, and rendered local-provider user-data/service configuration.

### Repository: `github.com/podplane/components`

1. Update the components dependency manifest to include `components.source.url` and `components.source.ref.semver` for the released components repo version.
2. Add `make git-sync` to snapshot (similar to how `dev-sync` works) the working tree into `~/.podplane/cache/deps/git/components.git` on branch `local-dev`.
3. Change `make recommended` defaults so, when the local Git cache exists, it points `PLATFORM_GIT_REPOSITORY_URL` at `https://10.0.2.15:19443/git/components.git` and `PLATFORM_GIT_REPOSITORY_BRANCH` at `local-dev`, while preserving explicit env var overrides. Only read those defaults from `podplane local status --json` and never hard code them.
4. Keep existing Kind `dev-sync`/`dev-bootstrap` behavior unchanged.

### Validation

1. Unit-test path and manifest behavior.
2. Start the local HTTPS server and verify `gogit clone --depth 1 https://<host-local-server>/git/components.git` succeeds.
3. In a local VM, verify the `socat` listener forwards `https://10.0.2.15:19443/git/components.git` to the current host-side HTTPS port and that TLS validates against the local VM node IP SAN.
4. Verify Flux `GitRepository` becomes Ready against the VM-local forwarded URL.
5. Verify `podplane local start` can reconcile recommended components without reaching GitHub when deps Git cache is populated.
