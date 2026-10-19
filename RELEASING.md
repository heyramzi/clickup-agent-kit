# Releasing

This repository is generated. The `cu` source lives in a private repository, and
each release builds the bundle there and pushes it here. Please don't open a pull
request against these files: the next release would carry them away. Open an
issue instead, and say what you ran and what you expected.

## Versions

The number in `package.json`, the tag (`v1.12.0`), `cu --version` and the heading in
[CHANGELOG.md](CHANGELOG.md) are always the same number.

- Patch: a fix, nothing you have to change.
- Minor: a new command or flag. It can also change or remove one, so read the changelog
  before you update a script.
- Major: only when the maintainer asks for it.

## Pin a version

`npm i -g heyramzi/clickup-agent-kit` installs the latest commit on `main`. To stay on
one release, name its tag:

```
npm i -g heyramzi/clickup-agent-kit#v1.13.0
```

Each release also carries an installable `.tgz` and a `SHA256SUMS` file.

## How a release happens

1. The maintainer merges the release PR in the private repo. That bumps the version
   and writes the changelog entry.
2. The publish script builds the bundle and refuses to continue if it carries a
   token, a workspace id, a client name or a local path.
3. It adds a commit here, tags it `vX.Y.Z` and creates the GitHub release.
4. The `release` workflow checks that the tag, `package.json`, the binary and the
   changelog agree, that no install hook is set, and that the package installs and
   runs on Node 20 and 24. Then it attaches the tarball.

If that workflow is red, the release is wrong. Don't use it.

## Nothing runs on install

`package.json` has no `preinstall`, `install` or `postinstall` hook and no runtime
dependencies. `cu` is one file, and it only talks to ClickUp.
