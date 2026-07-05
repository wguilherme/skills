---
name: asdf-plugin
description: Use this skill when the user wants to create, develop, or publish an asdf plugin for any tool or technology. Trigger for requests like "create asdf plugin for X", "how to write an asdf plugin", "publish asdf plugin", "plugin template", "bin/install script", "bin/list-all", or any mention of developing/building a new asdf plugin from scratch. Distinct from asdf usage — this is about authoring plugins, not consuming them.
---

# asdf Plugin Development Skill

Guide for creating asdf plugins — adapters that teach asdf how to manage versions of any tool or runtime.

---

## When to Use This Skill

- Creating a new asdf plugin for a tool not yet in the registry
- Understanding the required and optional plugin scripts
- Implementing version resolution, download, and install logic
- Testing a plugin locally before publishing
- Publishing the plugin to the asdf-plugins registry
- Debugging a broken or incomplete plugin

---

## Core Concepts

An asdf plugin is a **Git repository** with a `bin/` directory containing shell scripts. asdf calls these scripts with a well-defined interface. You implement the logic; asdf handles the rest (shims, `.tool-versions` resolution, `asdf install`, etc.).

### How asdf Resolves Versions (plugin dev must understand this)

When a user runs `<tool>`, asdf resolves the version in this order:

```
ASDF_<TOOL>_VERSION env var   ← highest priority
  ↓ (if not set)
.tool-versions in cwd
  ↓ (if not found)
.tool-versions in parent dirs (walks up to $HOME)
  ↓ (if not found)
$HOME/.tool-versions           ← global fallback
  ↓ (if not found)
error: "No version set"
```

Your plugin does **not** implement this logic — asdf does. But you must understand it to debug installs and test correctly.

### Shims

After `asdf install`, asdf creates shim scripts in `~/.asdf/shims/` for each binary. If shims are missing or stale after your `bin/install` runs, users call `asdf reshim <tool>`. Plugin devs must `asdf reshim <tool>` when testing locally after changes.

### Plugin Repository Layout

```
asdf-<tool>/
├── README.md
├── LICENSE
└── bin/
    ├── list-all          # REQUIRED — list installable versions
    ├── install           # REQUIRED — download and install a version
    ├── latest-stable     # recommended — return the latest stable version
    ├── download          # recommended (asdf v0.7+) — download to ASDF_DOWNLOAD_PATH
    ├── list-bin-paths    # optional — directories to add to PATH
    ├── exec-env          # optional — set env vars before exec
    ├── exec-path         # optional — override binary path per version
    ├── uninstall         # optional — cleanup after uninstall
    └── parse-legacy-file # optional — support legacy version files (.nvmrc, etc.)
```

### Naming Convention

```
asdf-<toolname>          # e.g. asdf-deno, asdf-zig, asdf-golangci-lint
```

Repo must be public on GitHub. The plugin name registered in the asdf-plugins index is just `<toolname>` (without `asdf-`).

---

## Quickstart: Use the Official Template

```bash
# Clone the template
git clone https://github.com/asdf-vm/asdf-plugin-template asdf-<tool>
cd asdf-<tool>

# Run the setup script — fills in tool name, URL patterns, test stubs
./setup.bash

# Remove template git history and start fresh
rm -rf .git
git init
git add .
git commit -m "feat: initial asdf-<tool> plugin"
```

The template generates `bin/install`, `bin/list-all`, and a GitHub Actions CI workflow.

---

## Required Scripts

### `bin/list-all`

Return all installable versions, space-separated or newline-separated, oldest first.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Fetch version list from GitHub releases API (most common pattern)
curl -fsSL "https://api.github.com/repos/OWNER/REPO/releases" \
  | grep '"tag_name"' \
  | sed 's/.*"tag_name": "v\?\([^"]*\)".*/\1/' \
  | sort -V
```

**Tips:**
- Strip leading `v` so `asdf install tool 1.2.3` works (not `v1.2.3`)
- Use `sort -V` (version sort) for correct ordering
- Cache aggressively — this is called often; hit GitHub API, not HTML
- Use `GITHUB_API_TOKEN` env var to avoid rate limits: `-H "Authorization: token ${GITHUB_API_TOKEN:-}"`

### `bin/install`

Download and install a specific version. asdf sets these env vars:

| Variable | Value |
|---|---|
| `ASDF_INSTALL_TYPE` | `version` or `ref` |
| `ASDF_INSTALL_VERSION` | The version string (e.g. `1.2.3`) |
| `ASDF_INSTALL_PATH` | Where to install (e.g. `~/.asdf/installs/tool/1.2.3`) |
| `ASDF_DOWNLOAD_PATH` | Where `bin/download` put the files (asdf v0.7+) |
| `ASDF_CONCURRENCY` | Parallel jobs hint |

```bash
#!/usr/bin/env bash
set -euo pipefail

install_tool() {
  local install_type="$1"
  local version="$2"
  local install_path="$3"

  local platform
  platform="$(uname -s | tr '[:upper:]' '[:lower:]')"  # linux, darwin
  local arch
  arch="$(uname -m)"  # x86_64, arm64, aarch64

  # Normalize arch for tools that use "amd64"/"arm64" naming
  case "$arch" in
    x86_64) arch="amd64" ;;
    aarch64) arch="arm64" ;;
  esac

  local url="https://github.com/OWNER/REPO/releases/download/v${version}/tool_${version}_${platform}_${arch}.tar.gz"
  local tmp_dir
  tmp_dir="$(mktemp -d)"
  trap 'rm -rf "$tmp_dir"' EXIT

  curl -fsSL "$url" -o "$tmp_dir/archive.tar.gz"
  tar -xzf "$tmp_dir/archive.tar.gz" -C "$tmp_dir"

  mkdir -p "$install_path/bin"
  cp "$tmp_dir/tool" "$install_path/bin/tool"
  chmod +x "$install_path/bin/tool"
}

install_tool "$ASDF_INSTALL_TYPE" "$ASDF_INSTALL_VERSION" "$ASDF_INSTALL_PATH"
```

---

## Recommended Scripts

### `bin/download` (asdf v0.7+)

Separate download from install — allows asdf to cache downloads.

```bash
#!/usr/bin/env bash
set -euo pipefail

# ASDF_DOWNLOAD_PATH is set by asdf
# Download to $ASDF_DOWNLOAD_PATH, do NOT install here
mkdir -p "$ASDF_DOWNLOAD_PATH"

url="https://github.com/OWNER/REPO/releases/download/v${ASDF_INSTALL_VERSION}/tool.tar.gz"
curl -fsSL "$url" -o "$ASDF_DOWNLOAD_PATH/archive.tar.gz"
tar -xzf "$ASDF_DOWNLOAD_PATH/archive.tar.gz" -C "$ASDF_DOWNLOAD_PATH" --strip-components=1
rm "$ASDF_DOWNLOAD_PATH/archive.tar.gz"
```

Then in `bin/install`, copy from `$ASDF_DOWNLOAD_PATH` instead of re-downloading:

```bash
cp -r "$ASDF_DOWNLOAD_PATH/." "$ASDF_INSTALL_PATH/"
```

### `bin/latest-stable`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Return a single version string — the latest stable release
curl -fsSL "https://api.github.com/repos/OWNER/REPO/releases/latest" \
  | grep '"tag_name"' \
  | sed 's/.*"tag_name": "v\?\([^"]*\)".*/\1/'
```

Accepts an optional filter prefix as `$1` (e.g., `asdf latest tool 1.` calls with `"1."`).

---

## Optional Scripts

### `bin/list-bin-paths`

By default asdf adds `$ASDF_INSTALL_PATH/bin` to PATH. Override if the tool uses a different layout:

```bash
#!/usr/bin/env bash
# Return space-separated list of dirs relative to $ASDF_INSTALL_PATH
echo "bin sbin"
```

### `bin/exec-env`

Set environment variables before executing a shim:

```bash
#!/usr/bin/env bash
export TOOL_ROOT="$ASDF_INSTALL_PATH"
export TOOL_HOME="$ASDF_INSTALL_PATH"
```

### `bin/exec-path`

Override the binary path for a given command:

```bash
#!/usr/bin/env bash
# $1 = install_path, $2 = version, $3 = command
echo "$1/bin/$3"
```

### `bin/parse-legacy-file`

Support legacy version files (e.g. `.nvmrc`, `.ruby-version`). Only called when the **user** has `legacy_version_file = yes` in their `~/.asdfrc`. Your plugin must declare support by implementing this script.

```bash
#!/usr/bin/env bash
# $1 = path to legacy file
# Print the version string to stdout — strip any leading "v"
cat "$1" | sed 's/^v//'
```

Common legacy files by ecosystem:

| File | Ecosystem |
|---|---|
| `.nvmrc` | Node.js |
| `.node-version` | Node.js |
| `.ruby-version` | Ruby |
| `.python-version` | Python |
| `.go-version` | Go |

### `bin/uninstall`

Cleanup after `asdf uninstall`. `$ASDF_INSTALL_PATH` is set.

```bash
#!/usr/bin/env bash
rm -rf "$ASDF_INSTALL_PATH"
```

---

## Platform / Architecture Matrix

Common patterns for binary URL construction:

```bash
platform="$(uname -s)"
arch="$(uname -m)"

case "$platform" in
  Linux)  os="linux" ;;
  Darwin) os="darwin" ;;
  *)      echo "Unsupported OS: $platform" >&2; exit 1 ;;
esac

case "$arch" in
  x86_64)          cpu="amd64" ;;
  arm64|aarch64)   cpu="arm64" ;;
  armv7l)          cpu="armv7" ;;
  *)               echo "Unsupported arch: $arch" >&2; exit 1 ;;
esac
```

Different tools use different naming schemes — always check the upstream release naming.

---

## Testing Locally

```bash
# Isolated environment — avoids polluting ~/.asdf during development
export ASDF_DATA_DIR="$HOME/.asdf-dev"

# Add plugin from local path (no need to push to GitHub)
asdf plugin add <tool> /path/to/asdf-<tool>

# List versions — calls bin/list-all
asdf list all <tool>

# Install — calls bin/download (if present) + bin/install
asdf install <tool> 1.2.3

# Set version for current dir
asdf set <tool> 1.2.3

# Verify version resolution and install path
asdf current <tool>          # shows active version + source file
asdf where <tool> 1.2.3     # shows install path on disk
asdf which <tool>            # shows which shim binary is resolved

# Test the binary
<tool> --version

# Regenerate shims if binary isn't found after install
asdf reshim <tool>

# Test version override without .tool-versions
ASDF_<TOOL>_VERSION=1.2.3 <tool> --version

# Reload plugin after local changes
asdf plugin update <tool>

# Full cleanup
asdf plugin remove <tool>
unset ASDF_DATA_DIR
```

> Replace `ASDF_<TOOL>_VERSION` with the uppercased tool name, e.g. `ASDF_DENO_VERSION`, `ASDF_KUBECTL_VERSION`.

### Unit Testing with bats

The official template uses [bats-core](https://github.com/bats-core/bats-core):

```bash
# tests/list-all.bats
@test "list-all returns versions" {
  result="$(bash "${BATS_TEST_DIRNAME}/../bin/list-all")"
  [ -n "$result" ]
}

@test "list-all output is version-sorted" {
  result="$(bash "${BATS_TEST_DIRNAME}/../bin/list-all")"
  sorted="$(echo "$result" | tr ' ' '\n' | sort -V | tr '\n' ' ' | xargs)"
  spaced="$(echo "$result" | xargs)"
  [ "$sorted" = "$spaced" ]
}
```

```bash
# Run tests
bats tests/
```

---

## GitHub Actions CI

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install asdf
        uses: asdf-vm/actions/setup@v3
      - name: Add plugin
        run: asdf plugin add ${{ github.event.repository.name }} $GITHUB_WORKSPACE
      - name: Install latest
        run: asdf install ${{ github.event.repository.name }} latest
      - name: Run tests
        run: |
          cd tests && bats .
```

---

## Publishing to the asdf-plugins Registry

1. Create the plugin repo: `github.com/<you>/asdf-<tool>`
2. Add a `README.md` with: requirements, install instructions, what the tool is
3. Fork [asdf-vm/asdf-plugins](https://github.com/asdf-vm/asdf-plugins)
4. Add entry to `plugins/<tool>`:

```
repository = https://github.com/<you>/asdf-<tool>
```

5. Open a PR — maintainers will verify the plugin works before merging

---

## Common Pitfalls

| Problem | Cause | Fix |
|---|---|---|
| `No compatible download` | Version string has `v` prefix | Strip `v` in `list-all` |
| Script not executable | Forgot `chmod +x` | `chmod +x bin/*` |
| Works on macOS, fails Linux | Platform check wrong | Test `uname -s` output case |
| Rate limited on `list-all` | No auth header | Add `GITHUB_API_TOKEN` header |
| Wrong binary after install | `list-bin-paths` missing | Add `bin/list-bin-paths` if tool uses non-standard dir |
| `asdf plugin update` needed constantly | Installing from local path | Install from published GitHub URL instead |
| Shim not found after install | Shims not regenerated | Run `asdf reshim <tool>` |
| `asdf current` shows wrong version | `ASDF_<TOOL>_VERSION` env set | `unset ASDF_<TOOL>_VERSION` |
| `asdf where` returns empty | Version not installed | Run `asdf install <tool> <version>` first |

---

## Real Plugin Examples

| Tool | Repo | Good for learning |
|---|---|---|
| deno | asdf-community/asdf-deno | Simple single binary |
| zig | asdf-community/asdf-zig | Multiple release formats |
| golangci-lint | hypnoglow/asdf-golangci-lint | Standard Go tool pattern |
| kubectl | asdf-community/asdf-kubectl | Kubernetes-style versioning |
| terraform | asdf-community/asdf-hashicorp | Multiple tools, one plugin |

---

## Reference

- Plugin API spec: https://asdf-vm.com/plugins/create.html
- Official template: https://github.com/asdf-vm/asdf-plugin-template
- Plugin registry: https://github.com/asdf-vm/asdf-plugins
- bats-core: https://github.com/bats-core/bats-core
- GitHub releases API: https://docs.github.com/en/rest/releases
