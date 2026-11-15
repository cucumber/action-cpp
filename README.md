# action-cpp

Reusable GitHub Actions shared across the Cucumber C++ projects.

## `install-cpp-dep`

Checks out a Cucumber C++ sub-project at a given ref, builds it with a CMake
workflow preset, and installs it so downstream projects find it via
`find_package()`. This replaces the copy-pasted "Checkout + Build and install
cucumber_*" step pairs currently duplicated across every `test-cpp.yaml` job.

### Inputs

| Input        | Required | Default        | Description                                             |
| ------------ | -------- | -------------- | ------------------------------------------------------- |
| `dependency` | yes      | —              | File containing `owner/name@<40-character commit>` with an optional ` # comment`. |
| `working-directory` | yes | —            | Directory in which the dependency file is read. |
| `path`       | no       | `<name>-src`   | Checkout directory. Defaults to the repo name + `-src`. |
| `preset`     | no       | `host-system`  | CMake workflow preset used to build the dependency.     |

Inputs are passed to commands as literal data, not executable shell source.
The resolved repository, commit and checkout path must not contain CR or LF;
multiline values (including multiple matching dependency references) fail before
checkout outputs are written. Preset names remain subject to CMake's rules.

### Usage

A downstream workflow references it by `owner/repo@ref`. For example, `query`'s
`test-cpp.yaml` only needs to declare its direct dependency — `messages`:

```yaml
steps:
  - uses: actions/checkout@v7
    with:
      persist-credentials: false

  # ... install system packages (nlohmann-json, gtest, ...) ...

  - name: Install cucumber_messages
    uses: cucumber/action-cpp/install-cpp-dep@v1
    with:
      dependency: dependencies/messages.version
      working-directory: .

  - name: Build and unit test
    working-directory: cpp
    run: cmake --workflow --preset test-system
```

`pretty-formatter`, which depends on `query` (and transitively `messages`),
would add one call per dependency it builds:

```yaml
  - name: Install cucumber_messages
    uses: cucumber/action-cpp/install-cpp-dep@v1
    with:
      dependency: dependencies/messages.version
      working-directory: .

  - name: Install cucumber_query
    uses: cucumber/action-cpp/install-cpp-dep@v1
    with:
      dependency: dependencies/query.version
      working-directory: .
```

For example, `dependencies/messages.version` contains:

```text
cucumber/messages@e85a9eaf3fa3787c25bc62b6ea67672e709ee4fa # v34.2.1
```

## Keeping the pinned refs up to date with Renovate

Dependency files contain plain git commits. Configure a Renovate custom manager
for these files in the downstream repository (or centrally in
`cucumber/renovate-config`), for example:

```json
{
  "customManagers": [
    {
      "customType": "regex",
      "managerFilePatterns": ["/^dependencies/.+\\.version$/"],
      "matchStrings": [
        "(?<depName>[0-9a-z]+/[0-9a-z]+)@(?<currentDigest>[0-9a-f]{40})"
      ],
      "datasourceTemplate": "git-refs",
      "packageNameTemplate": "https://github.com/{{{depName}}}",
      "currentValueTemplate": "main"
    }
  ]
}
```

This follows the repository's `main` branch.

The `actions/checkout` pin inside `install-cpp-dep/action.yml` is handled
automatically by Renovate's built-in `github-actions` manager — no annotation
required.

## Regression tests

Run `python -m pip install -r tests/requirements.txt`, then
`python -m unittest discover -s tests -v` from this repository.
The tests execute the action scripts with stubbed commands; they do not install
dependencies or invoke real sudo. PowerShell tests require `pwsh` and are skipped
when it is unavailable. On Windows, set `ACTION_TEST_BASH` to the Git Bash
executable. CI runs the tests on Linux and Windows.
