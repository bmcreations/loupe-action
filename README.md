# loupe preview action

Installs a pull request's APK on a loupe target and leaves one comment on the
PR with a link that opens that target. Open the link on an iPhone and the
build is on the device in front of you.

```
Gradle build ──> this action ──POST /api/install──> loupe on the Mac ──> emulator
                     │
                     └── PR comment: link, target, package, install time
```

## What the comment links to, and what it does not carry

The link is `https://<server>/?target=<id>` with **no token**. A PR comment is
readable by everyone who can see the PR, and loupe's token is a single shared
secret that drives every target on the server, so putting it in the comment
would hand device control to all of them.

So the link only works in a browser that already holds loupe's cookie: Safari
on a phone that has opened the startup link once, or the Home Screen app
installed from it. Anyone else gets a 401. That stays true until scoped share
links exist (roadmap item 7 in `docs/product-direction.md`).

## Inputs

| Input | Required | Default | |
| --- | --- | --- | --- |
| `server` | yes | | Base URL, e.g. `https://bmc-m3-max.tail69c1a2.ts.net:18456` |
| `token` | yes | | The server's token. Pass a secret |
| `target` | yes | | Target id, e.g. `avd:Pixel_10`. Must be running |
| `apk` | yes | | Paths or globs, whitespace separated. Two or more files go as one base-plus-splits install |
| `allow-test` | | `false` | Accept an `android:testOnly` APK (`adb install -t`) |
| `launch` | | `false` | Sends `?launch=1`. The server ignores it until `feat/install-launch` lands |
| `package` | | | Package name for the comment. Read from the base APK with `aapt2` when empty |
| `comment` | | `true` | Post or update the PR comment |
| `insecure` | | `false` | Skip TLS verification, for a local test server only |
| `github-token` | | `github.token` | Needs `pull-requests: write` |

Outputs: `outcome` (`ok`, `refused`, `error`), `url`, `package`, `code` (the
`INSTALL_FAILED_*` reason on a refusal) and `took-ms`.

The comment is keyed on the target, so a rerun edits it in place and a matrix
over two targets keeps two comments. A failed install updates it too, so it
never keeps pointing at a build that is not on the device. The step still
fails the job.

## Limits

- **The target must already be running.** `/api/install` answers 502 for a
  shut-down AVD ("open it in the viewer to boot it"), and nothing in the API
  boots one without a viewer session. The comment shows that message.
- **Launch is not wired.** The server has no launch step yet. With
  `launch: true` the comment says the app is installed and has to be opened
  from the launcher.
- **`bmcreations/loupe` is private.** Another repo can only `uses:` this action
  after Settings, Actions, General, Access on this repo allows it.

## Reaching the server from a runner

The server is on a Mac inside a tailnet, and GitHub-hosted runners cannot
reach it. Two ways in:

| | Tailscale on a hosted runner | Self-hosted runner on the Mac |
| --- | --- | --- |
| Where the PR's code runs | GitHub's VM | The Mac that also runs loupe and the emulators |
| Secrets in the consumer repo | Tailscale OAuth client id and secret, `LOUPE_TOKEN` | `LOUPE_TOKEN` |
| Network reach | One ephemeral tagged node; an ACL can limit it to TCP 18456 on the Mac | Everything the Mac can reach |
| Setup | OAuth client with the `auth_keys` scope, a tag, an ACL grant | Register a runner on the Mac |
| Install speed | Over the tailnet, possibly through a DERP relay | Loopback |

This action uses the Tailscale route. A self-hosted runner executes whatever
the PR's workflow says on the Mac, so a PR that edits the workflow gets a
shell next to the emulators. The Tailscale node lives for one job and the ACL
can stop it at a single port.

Only TCP is needed: the install is one HTTPS POST. WebTransport's UDP matters
to the phone watching the stream, not to the runner uploading the build.

### Tailscale setup

1. A tag for CI nodes, and a grant that lets it reach only loupe:

   ```json
   "tagOwners": { "tag:loupe-ci": ["autogroup:admin"] },
   "grants": [
     { "src": ["tag:loupe-ci"], "dst": ["100.91.141.96"], "ip": ["tcp:18456"] }
   ]
   ```

2. An OAuth client with the `auth_keys` scope and tag `tag:loupe-ci`. Nodes it
   creates are ephemeral and removed when the job ends.
   `tailscale/github-action` v4 also takes an `audience` for OIDC workload
   identity, which removes the stored secret; that has not been tried here.
3. In the consumer repo, secrets `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET` and
   `LOUPE_TOKEN`.

## Example: a Gradle Android project

```yaml
name: Preview on loupe

on:
  pull_request:

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: loupe-preview-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  preview:
    # Fork PRs get no secrets; skip them rather than fail.
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21

      - uses: gradle/actions/setup-gradle@v4

      - run: ./gradlew assembleDebug

      - uses: tailscale/github-action@v4
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
          oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
          tags: tag:loupe-ci

      - uses: bmcreations/loupe/action@main
        with:
          server: https://bmc-m3-max.tail69c1a2.ts.net:18456
          token: ${{ secrets.LOUPE_TOKEN }}
          target: avd:Pixel_10
          apk: app/build/outputs/apk/debug/*.apk
```

`assembleDebug` writes one universal APK, so `apk` matches one file. A build
from an App Bundle produces a base and config splits; point `apk` at all of
them and they go up as one `install-multiple`, since the base alone is refused
with `INSTALL_FAILED_MISSING_SPLIT`.

## Running the scripts locally

`install.sh` and `comment.sh` hold all the logic; `action.yml` only maps
inputs to environment variables. Against a spare server:

```sh
LOUPE_SERVER=https://127.0.0.1:18472 LOUPE_TOKEN=... LOUPE_INSECURE=true \
  LOUPE_TARGET=avd:api34_emoji_check LOUPE_APKS='build/*.apk' \
  LOUPE_RESULT=/tmp/r.json action/install.sh
action/comment.sh --dry-run /tmp/r.json
```

Never run `install.sh` under `set -x`: it would print the token. The token
reaches curl on stdin, not its command line, so `ps` on a shared runner does
not show it.
