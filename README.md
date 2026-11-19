# rc-cli

`rc-cli` is the Runtime Conditions CLI. A Runtime Conditions Profile is a portable, machine-readable list of what one workload needs at runtime: the APIs, caches, datastores, and environment configuration it depends on. 

`rc-cli` **generates** Runtime Conditions Profiles from your workload's binding declarations and **validates** profiles against the spec and the extensions it declares, whether the profile was written by hand, by an agent, or by a profiler. The profile states the demand; platforms and adapters decide how to fulfill it.

**Example scenario:** Your service depends on a Todos API reachable at `TODOS_API_URL` and a Redis cache at `REDIS_URL`. Today that knowledge lives in a wiki page or someone's head, and the platform team provisions from a guess. A Runtime Conditions Profile writes it down where the code lives: you declare the need once in your source, `rc profile generate` reads those declarations and emits the profile as YAML, and `rc profile validate` checks the profile is well-formed and matches the extensions it uses. Then Kratix, Crossplane, Radius, or Backstage can read the same file and fulfill the need. No re-explaining, no drift between docs and reality.

- Guide and concepts: <https://runtimeconditions.io>
- Profile specification: <https://github.com/runtimeconditions/spec>

## Install

The package is named `rc-cli`. The executable is named `rc`.

**Homebrew (macOS and Linux):**

```sh
brew install runtimeconditions/tools/rc-cli
```

**Linux packages (apt / dnf / yum):** configure the signed package source once, then install
with your package manager:

```sh
curl -fsSL https://runtimeconditions.io/packages/setup.sh | sudo sh
sudo apt install rc-cli    # Debian / Ubuntu
sudo dnf install rc-cli    # Fedora / RHEL
```

Language profilers are separate add-ons. A profiler reads your workload's code and
generates its profile:

- Go profiling is built into `rc`, so Go workloads need nothing else.
- TypeScript, Java, and Python profilers install through their own channels; see [Optional language profilers](#optional-language-profilers).

**From source:** see [Build from source](#build-from-source).

## Commands

Verify the install and see which profilers `rc` can run:

```sh
rc --version
```

```text
rc v0.6.0 (a1b2c3d)
```

`rc version --output yaml` adds the Go toolchain, platform, and linked profiler components.

```sh
rc plugins list
```

```text
LANGUAGE    STATUS    VERSION  PATH
go          built-in  v0.8.0
typescript  missing
java        missing
python      missing
```

`go` is the profiler linked into `rc` — nothing extra to install. `typescript`, `java`, and
`python` are add-ons: `missing` just means "not installed", which only matters if your
workload is in that language. `rc plugins list --output yaml` also prints each missing
add-on's install command, e.g. `install with: npm install --global @runtimeconditions/typescript-profiler`.

`rc --help` lists all commands; `rc completion <bash|zsh|fish|powershell>` prints a shell
completion script.

`go` is the profiler linked into `rc` — nothing extra to install. `typescript`, `java`, and
`python` are add-ons: `missing` just means "not installed", which only matters if your
workload is in that language. `rc plugins list --output yaml` also prints each missing
add-on's install command, e.g. `install with: npm install --global @runtimeconditions/typescript-profiler`.

`rc --help` lists all commands; `rc completion <bash|zsh|fish|powershell>` prints a shell
completion script.

## Quickstart: validate an existing profile

This is the one hands-on step that works on a fresh install. It checks a committed demo profile.
It does not create a profile yet:

```sh
git clone https://github.com/runtimeconditions/rc-demos.git
rc profile validate rc-demos/artifacts/request-logger-http.profile.yaml
```

```text
runtimeconditions: profile validated
```

`rc profile validate` checks any profile file against the core schema embedded in `rc` and
against the definitions of the extensions the profile declares. By default those definitions
are downloaded from the `https` ids the profile names. To work offline or against local
definitions instead, pass one or more catalog directories:

```sh
rc profile validate profile.yaml --catalog ./my-extensions
rc profile validate - < profile.yaml   # read the profile from stdin
```

## Generating a profile

`rc profile generate` reads the Go binding declarations in a workload and writes a validated
profile. It analyzes declarations statically and never runs your code. Three prerequisites:

1. **A Go toolchain on `PATH`.** Without one, generation fails with
   `exec: "go": executable file not found in $PATH`. `rc` does not download Go for you.
2. **`--dir` must point at a directory containing a `go.mod`.** Current releases fail fast:

   ```text
   rc: /path/to/dir: workload must have a go.mod: stat /path/to/dir/go.mod: no such file or directory
   ```

   Older releases (for example the Homebrew v0.6.0 build) fell back to GOPATH mode and scanned
   your entire home directory, spraying `operation not permitted` errors for TCC-protected
   paths. Always point `--dir` at a Go module.
3. **The workload must import published RC binding modules** (`runtimeconditions.io/x/rc/...`).
   Bindings wired with `go.mod` `replace` directives to a local clone are rejected **by
   design**:

   ```text
   rc: github.com/runtimeconditions/extensions/catalog/rc/common-integrations/go uses a go.mod replace; generated bindings must resolve to a published module version
   ```

   This is why a fresh clone of `rc-demos/apps/request-logger-http` cannot be generated: its
   `go.mod` replaces the binding modules with a sibling checkout of `extensions`. Generate
   from a module that requires the published versions instead.

### Example workload

`go.mod`:

```go
module example.com/my-service

go 1.25.0

require (
	runtimeconditions.io/x/rc/common v0.3.0
	runtimeconditions.io/x/rc/env v0.3.0
)
```

`main.go` — declare conditions with the published binding packages:

```go
package main

import (
	"runtimeconditions.io/x/rc/common"
	"runtimeconditions.io/x/rc/env"
)

var todosAPI = common.Api(
	common.Http{
		Spec: &common.Spec{Format: common.FormatOpenapi, Uri: "catalog://api/default/todos-api"},
		Operations: common.HttpOperations{
			{Method: common.MethodGet, Path: "/todos/{id}",
				ResponseSchema: common.ResponseSchemaMap{"id": common.SimpleSchemaStringInteger}},
		},
	},
	env.HttpConfiguration{
		Env: env.HttpConfigurationEnv{
			{Name: "TODOS_API_URL", Property: env.HttpConfigurationEnvPropertyBaseUrl},
		},
	},
)

var requestCache = common.Cache(
	common.KeyValue{Engine: &[]common.KeyValueEngine{common.KeyValueEngineRedis}[0]},
	env.KeyValueConfiguration{
		Env: env.KeyValueConfigurationEnv{
			{Name: "REDIS_URL", Property: env.KeyValueConfigurationEnvPropertyUrl},
		},
	},
)

func main() {}
```

Resolve dependencies, then generate:

```sh
go mod tidy
rc profile generate --dir ./my-service --out profile.yaml
```

Defaults: `--dir` is the current directory, `--out` is stdout, `--name` is the directory's
basename, `--workload-uri` is the enclosing Go module path, `--workload-version` is `dev`.
A failed generation leaves an existing `--out` file unchanged. You can set them explicitly:

```sh
rc profile generate \
  --dir ./my-service \
  --name my-service \
  --workload-uri https://example.com/my-service \
  --workload-version 1.0.0 \
  --out profile.yaml
```

**Verified status today:** a workload importing only `runtimeconditions.io/x/rc/common@v0.3.0`
generates a profile that passes `rc profile validate`, end to end. Adding
`runtimeconditions.io/x/rc/env@v0.3.0` currently fails dependency artifact verification with:

```text
rc: runtimeconditions.io/x/rc/env dependency runtimeconditions.io/x/rc/common: dependency Go module archive SHA-256 mismatch
```

The published-module pattern above is the supported direction; treat that error as a
known gap in the current published `env` release, not as a mistake in your workload.

### Checking a workload's bindings

```sh
rc profile validate-extensions --dir ./my-service
rc profile validate-extension --dir ./my-service --package runtimeconditions.io/x/rc/common
rc profile go version    # linked Go profiler version, e.g. "go-rc-profiler v0.8.0 (linked into rc)"
```

## Command reference

`rc profile` uses the built-in Go profiler by default. The same commands are available under
`rc profile go`, with identical flags and behavior:

| Command | Purpose |
| --- | --- |
| `rc profile generate` | Generate a validated profile from Go binding declarations. Flags: `--dir` (default `.`), `--name`, `--workload-uri`, `--workload-version` (default `dev`), `--out` (default stdout). |
| `rc profile validate <profile.yaml>` | Validate any profile against the embedded core schema and its extensions. Flags: `--catalog` (repeatable). |
| `rc profile validate-extensions` | Validate all imported generated binding packages. Flag: `--dir`. |
| `rc profile validate-extension` | Validate one imported binding package. Flags: `--dir`, `--package`. |
| `rc profile version` | Show the linked Go profiler version. |
| `rc plugins list [--output yaml]` | Show the Go profiler and each add-on's status, version, and path. |
| `rc version [--output text\|yaml]` | Show CLI build information. `rc --version` prints the same one-liner. |
| `rc completion <bash\|zsh\|fish\|powershell>` | Print a shell completion script. |

For full flag lists:

```sh
rc profile --help
rc profile go --help
rc profile go generate --help
rc bindings --help
```

### Binding commands

`rc bindings` resolves, generates, and verifies extension bindings. The command group
provides `resolve`, `normalize`, `generate`, `verify`, `package`, `plan-release`, `check`,
and `update`. Cobra owns help, completion, and flag parsing; execution uses the shared
binding library from `extensions`.

Binding commands also require the shared YAML schemas. Installed layouts place them under
`share/rc/extension-bindings/model/`, alongside `bin/rc`. CI stages these files from the
exact `extensions` commit in `distribution/components.lock.yaml`.

For local binding development, supply `--tooling-dir` pointing to matching schema resources,
such as the `tooling/extension-bindings` directory in a checkout of that pinned revision:

```sh
rc bindings generate \
  --packages /path/to/binding-project/packages.yaml \
  --toolchain-lock /path/to/binding-project/toolchain.lock.yaml \
  --tooling-dir /path/to/extensions/tooling/extension-bindings \
  --core-schema /path/to/runtimeconditions.profile.schema.yaml \
  --output /tmp/generated-bindings
```

`packages.yaml` selects extension roots, language targets, and native package versions.
`toolchain.lock.yaml` declares the applicable toolchain constraints. Use repeatable
`--target package-key:language` flags to limit the selected targets. `generate` requires a
new or empty output directory. `verify --input` accepts the generated tree; use the same
catalog and resource flags for these commands. `update` verifies candidates before changing
committed binding directories; `check` compares regenerated content without updating those
files. Other shared flags include `--cache`, `--work-dir`, `--catalog-root`,
`--dependency-lock`, `--extension-root`, and `--extension-override`; see
`rc bindings generate --help` for the full list.

The core schema must match the version declared in the package catalog. Without
`--core-schema`, normalization uses upstream core-schema resolution. Offline generation
also requires locally available extension definitions; native package verification and
profiling require resolved package dependencies and Go.

The normalizer and Go emitter run as linked libraries. Explicit executable overrides for
those tools (`--normalizer`, `--go-emitter`) conflict with this mode. For binding
provenance, `--go-profiler /absolute/path/to/go-rc-profiler` selects a standalone profiler
file; otherwise the linked profiler's actual module version and checksum are recorded. The
`rc profile` commands always use the linked Go profiler.

## Optional language profilers

Other languages run a separate add-on that `rc` finds on `PATH`. It can come from the
language's own tooling or from a system package, and `rc` does not care which:

- **TypeScript:** `npm install --global @runtimeconditions/typescript-profiler`, or
  `brew install runtimeconditions/tools/rc-profiler-typescript`.
- **Java:** download the launcher `rc-profiler-java` from the
  [releases](https://github.com/runtimeconditions/java-rc-profiler/releases) of
  `runtimeconditions/java-rc-profiler` and put it on `PATH`, or
  `brew install runtimeconditions/tools/rc-profiler-java`.
- **Python:** `pip install runtimeconditions-profiler` into the workload's own virtual
  environment — it finds binding packages through that environment's metadata — and put the
  environment's `bin` directory on `PATH`. Do not use `pipx`, which would give it an
  environment of its own.

| Command | Purpose |
| --- | --- |
| `rc profile typescript <command> [flags]` | Run the installed TypeScript profiler. |
| `rc profile java <command> [flags]` | Run the installed Java profiler. |
| `rc profile python <command> [flags]` | Run the installed Python profiler. |
| `rc plugins list [--output yaml]` | Show the Go profiler and each add-on's status, version, and path. |

Everything after `typescript`, `java`, or `python` goes to the add-on unchanged, so it keeps
its own flags (for example `--project` instead of `--dir`, and `--classpath` for Java). `rc`
looks for `rc-profiler-typescript`, then `runtimeconditions-typescript-profiler`, or
`rc-profiler-java`, or `runtimeconditions-python-profiler`, on `PATH`;
`RC_TYPESCRIPT_PROFILER`, `RC_JAVA_PROFILER`, or `RC_PYTHON_PROFILER` can name an absolute
path instead. It runs the add-on only if `--version --json` lists the core profile schema
`rc` targets (0.4.0). The add-on's output and exit status pass through.

The Java launcher chooses the JDK, not `rc`: `RC_JAVA_HOME`, then `JAVA_HOME`, then the
runtime bundled in its system package, then `java` on `PATH`. It needs JDK 17 or newer and
says so when none is found.

The Python profiler runs on the interpreter of the environment it is installed in (Python
3.11 or newer), and sees the binding packages installed there.

## Build from source

From this repository's root, build the executable with the schema-aware helper:

```sh
go run ./scripts/build.go --output ./rc
./rc --version
./rc profile generate --dir /path/to/go/workload --out profile.yaml
```

The helper downloads the pinned core profile schema, verifies its SHA-256, and embeds it
alongside the linked Go profiler. Go profiling needs no runtime schema download or separate
profiler installation. Analyzing a workload requires a working Go toolchain and its binding
dependencies. CI builds with Go 1.26.5 and tests workload analysis with Go 1.25.0.

A plain `go build` or `go install` omits the schema linker value. Such a binary can display
help and version information, but profile generation fails with
`rc: installed profiler has no verified core profile schema`. Rebuild it using the command
above and run the resulting `./rc` executable.

For a build using an already downloaded schema, supply the approved artifact:

```sh
go run ./scripts/build.go --output ./rc \
  --core-schema /path/to/runtimeconditions.profile.schema.yaml
```

The local schema must match the helper's pinned checksum. Release builds also pass
`--version <release-tag>` and `--commit <source-commit>` to embed provenance.

CI tests Linux and macOS on amd64 and arm64. It relocates the staged executable and YAML
resources, exercises all eight binding operations, installs a freshly generated Go module
through a local proxy, and profiles its declarations through both Go command routes.
Negative checks cover altered committed source, nonempty output directories, and invalid
declarations preserving existing profile output. The small extension fixture is checked in as
YAML under `testdata/ci/bindings/`; generated packages, archives, profiles, and module caches
remain temporary CI outputs. Homebrew and APT/RPM packaging live in
[`homebrew-tools`](https://github.com/runtimeconditions/homebrew-tools) and
[`packages`](https://github.com/runtimeconditions/packages).

## Ecosystem map

| Repository | What it is |
| --- | --- |
| [spec](https://github.com/runtimeconditions/spec) | The Runtime Conditions Profile specification and supporting docs. |
| [extensions](https://github.com/runtimeconditions/extensions) | First-party extension definitions and the shared binding tooling. |
| [rc-demos](https://github.com/runtimeconditions/rc-demos) | End-to-end demos, including the `request-logger-http` app and its committed profile. |
| [go-rc-profiler](https://github.com/runtimeconditions/go-rc-profiler) | Go AST profiler; linked into `rc`, also usable standalone. |
| [ts-rc-profiler](https://github.com/runtimeconditions/ts-rc-profiler) | TypeScript profiler add-on (`rc profile typescript`). |
| [java-rc-profiler](https://github.com/runtimeconditions/java-rc-profiler) | Java profiler add-on (`rc profile java`). |
| [python-rc-profiler](https://github.com/runtimeconditions/python-rc-profiler) | Python profiler add-on (`rc profile python`). |
| [ebpf-rc-profiler](https://github.com/runtimeconditions/ebpf-rc-profiler) | Infers external dependencies by watching syscall and network traffic instead of reading bindings. |
| [backstage-plugin](https://github.com/runtimeconditions/backstage-plugin) | Backstage plugin for Runtime Conditions Profiles. |
| [homebrew-tools](https://github.com/runtimeconditions/homebrew-tools) | Homebrew tap (`runtimeconditions/tools`). |
| [packages](https://github.com/runtimeconditions/packages) | Signed apt/dnf/yum package source and `setup.sh`. |
