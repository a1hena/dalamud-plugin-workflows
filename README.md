# dalamud-plugin-workflows

Reusable GitHub Actions workflows that build a Dalamud plugin and publish it to
[dalamud-plugin-server](https://dalamud-plugins.hyuma.dev).

- `check.yaml` builds the plugin and confirms the packaged manifest carries `InternalName`,
  `AssemblyVersion` and `DalamudApiLevel`. Run it on pull requests.
- `release.yaml` picks the next version from conventional commits, builds with it, publishes,
  puts the version on the testing channel of every endpoint delivering the plugin, then tags the
  commit and creates a GitHub Release. Run it on pushes to the default branch.

Stable is never changed by these workflows. Promote a version to stable in the admin console.

## Using them

Each plugin repository keeps two short files that only set inputs and call the workflows.

`.github/workflows/check.yaml`

```yaml
name: check

on:
  pull_request:
  workflow_dispatch:

jobs:
  check:
    uses: a1hena/dalamud-plugin-workflows/.github/workflows/check.yaml@v1
    with:
      plugin-name: MyPlugin
```

`.github/workflows/release.yaml`

```yaml
name: release

on:
  push:
    branches: [master]
  workflow_dispatch:

jobs:
  release:
    uses: a1hena/dalamud-plugin-workflows/.github/workflows/release.yaml@v1
    permissions:
      contents: write
    with:
      plugin-name: MyPlugin
    secrets:
      PLUGIN_SERVER_API_KEY: ${{ secrets.PLUGIN_SERVER_API_KEY }}
```

The calling job has to grant `contents: write`. A called workflow cannot raise its own
permissions, and the release step creates the tag and the GitHub Release.

## Inputs

| Input | Workflows | Default | Meaning |
| --- | --- | --- | --- |
| `plugin-name` | both | (required) | The plugin's InternalName |
| `build-target` | both | empty | The solution or project to build. Empty builds the repository root |
| `artifact-dir` | both | empty | Where the packaged zip and `<plugin-name>.json` are. Empty finds the directory holding both |
| `dalamud-zip-url` | both | stable `latest.zip` | The Dalamud build to compile against |
| `dotnet-version` | both | `10.0.x` | The .NET SDK |
| `server-url` | release | `https://dalamud-plugins.hyuma.dev` | The server to publish to |
| `testing` | release | `true` | Whether to put the new version on testing |

## The API key

Issue a key in the admin console and store it as the repository secret `PLUGIN_SERVER_API_KEY`.

- Register versions, to publish
- Create plugins, for the first release of a plugin the server does not know yet
- Change channels, for the testing step

The testing step only reaches endpoints that already deliver the plugin. Subscribe an endpoint to
the plugin in the admin console once; from then on every release lands on its testing channel.

## Versions

The version is the latest `v*` tag bumped by the commits since: `fix:` a patch, `feat:` a minor,
`feat!:` or `BREAKING CHANGE:` a major. Commits such as `docs:`, `ci:` or `chore:` release nothing.
The build gets `-p:Version=<version>.0`, so the csproj `<Version>` is not used for releases.

## Versioning of this repository

Callers pin `@v1`. Changes that keep every input working move `v1`; a change that breaks a caller
gets `v2`.
