# GitHub Actions for Fastforge

This repository stores the GitHub Actions for the [Fastforge](https://github.com/fastforgedev/fastforge) tool. which can be used to do a variety of tasks.

## List of Actions

1. [fastforgedev/actions/setup](setup/action.yml) - Setup the Fastforge environment for your app.
2. [fastforgedev/actions/package](package/action.yml) - Package your app into OS-specific bundles.
3. [fastforgedev/actions/publish](publish/action.yml) - Publish your app into popular testing or distribution platforms.

The actions drive the standalone `fastforge` binary. `setup` downloads it from
the [Fastforge releases](https://github.com/fastforgedev/fastforge/releases) —
there is no `dart pub global activate` step and no Dart dependency beyond the
Flutter SDK your app already builds with.

## Usage

```yaml
jobs:
  build:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          channel: stable

      - uses: fastforgedev/actions/setup@main
        with:
          fastforge-version: "0.7.0" # optional, defaults to the latest release

      - name: Package
        id: package
        uses: fastforgedev/actions/package@main
        with:
          platform: macos
          target: dmg
          working-directory: .
          artifact-name: "my-app-{{build_name}}-{{platform}}.{{ext}}"

      - name: Publish
        uses: fastforgedev/actions/publish@main
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          path: ${{ steps.package.outputs.artifact-paths }}
          target: github
          github-repo: ${{ github.repository }}
          github-release-title: v1.0.0
          publish-args: '{"release-tag":"v1.0.0"}'
```

## `fastforgedev/actions/setup`

| Input               | Description                                                                                         |
| ------------------- | --------------------------------------------------------------------------------------------------- |
| `fastforge-version` | Version to install, without a leading `v`. Defaults to the latest release for the runner platform.   |
| `install-dir`       | Where to install the binary. Defaults to `$HOME/.local/bin` (macOS/Linux) or `%LOCALAPPDATA%\fastforge\bin` (Windows). |
| `token`             | GitHub token used to look up the latest release. Defaults to the workflow token.                     |

| Output    | Description                              |
| --------- | ---------------------------------------- |
| `version` | The installed Fastforge version.         |
| `path`    | The directory the binary was installed into. |

## `fastforgedev/actions/package`

`platform` and `target` are required and map to `fastforge package`. `target`
accepts a comma separated list (`deb,appimage`), and every target of one
platform shares a single `flutter build`.

| Input                        | Description                                                                          |
| ---------------------------- | ------------------------------------------------------------------------------------ |
| `platform`                   | `android`, `ios`, `linux`, `macos`, `ohos`, `web` or `windows`.                       |
| `target`                     | `apk`, `aab`, `app`, `appimage`, `deb`, `dmg`, `exe`, `hap`, `ipa`, `msix`, `pkg`, `rpm`, `zip`, `direct` or `custom`. |
| `working-directory`          | Directory holding `pubspec.yaml`. Defaults to `.`.                                    |
| `artifact-name`              | Mustache template for the artifact name, for example `{{name}}-{{build_name}}-{{platform}}.{{ext}}`. |
| `channel`                    | Distribution channel, replaces the flavor segment of the default artifact name.        |
| `skip-clean`                 | `true` skips `flutter clean` before packaging.                                        |
| `build-target`               | The `--target` passed to `flutter build`, for example `lib/main_prod.dart`.            |
| `build-flavor`               | The `--flavor` passed to `flutter build`.                                             |
| `build-target-platform`      | The `--target-platform` passed to `flutter build`.                                    |
| `build-export-options-plist` | The `--export-options-plist` passed to `flutter build`.                               |
| `flutter-build-args`         | Extra `flutter build` arguments, comma separated, for example `verbose,obfuscate`.     |
| `dart-define`                | JSON object of `--dart-define` values, for example `{"APP_ENV":"production"}`.         |
| `hook-pre` / `hook-post`     | Shell commands around the packaging step, with `BUILD_OUTPUT_DIRECTORY`, `OUTPUT_DIRECTORY`, … in their environment. |

| Output             | Description                                                                |
| ------------------ | -------------------------------------------------------------------------- |
| `artifact-paths`   | Comma separated list of the packaged artifacts, relative to `working-directory`. |
| `artifact-count`   | Number of artifacts that were packaged.                                    |
| `output-directory` | Directory the artifacts were written to (the `output` key of `distribute_options.yaml`, default `dist/`). |

Artifacts are written to `<output-directory>/<app version>/<artifact name>`.
Every packaging step of a release should pass `artifact-name` explicitly: the
`artifact_name` key of `distribute_options.yaml` is only read by the legacy
`fastforge release` command.

## `fastforgedev/actions/publish`

`path` and `target` are required and map to `fastforge publish`. `path` accepts
a comma separated list — the `artifact-paths` output of the package action can
be passed straight through, and every artifact is published in turn. Because
`fastforge publish` takes one artifact at a time, the action also runs the
publisher once per artifact.

| Input               | Description                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `path`              | Artifact(s) to publish, relative to `working-directory`.                                         |
| `target`            | `appgallery`, `appstore`, `cos`, `custom`, `fir`, `firebase`, `firebase-hosting`, `github`, `minio`, `oss`, `pgyer`, `playstore`, `qiniu`, `s3` or `vercel`. |
| `working-directory` | Directory to run `fastforge` from. Defaults to `.`.                                              |
| `app-version`       | Version of the app. Only needed when there is no `pubspec.yaml` to read it from.                  |
| `publish-args`      | JSON object of additional publisher arguments, for example `{"release-tag":"v1.0.0"}`. Values must be strings. |

The provider inputs of the action (`github-repo`, `github-release-title`,
`firebase-*`, `minio-*`, `playstore-*`, `qiniu-*`, `vercel-*`) are kept from the
Dart era for backwards compatibility. Anything they do not cover — the GitHub
`release-tag`, or every option of the `s3`, `oss`, `cos`, `appstore`, `pgyer`
and `custom` targets — is passed with `publish-args`, which takes precedence
over them. `fastforge publish --help` lists all of the available arguments.

Credentials always come from the environment of the step, never from inputs:

| Target                      | Environment                                            |
| --------------------------- | ------------------------------------------------------ |
| `github`                    | `GITHUB_TOKEN`                                         |
| `firebase`, `firebase-hosting` | `FIREBASE_TOKEN`                                    |
| `pgyer`                     | `PGYER_API_KEY`                                        |
| `playstore`                 | `PLAYSTORE_CREDENTIALS` (path to a service account JSON) |
| `s3`, `minio`, `qiniu`, `oss`, `cos` | `S3_*`, `MINIO_*`, `QINIU_*`, `OSS_*`, `COS_*` |

## Troubleshooting

### `setup` fails with `No release provides a prebuilt binary for <target>`

The installer reads the [Fastforge releases](https://github.com/fastforgedev/fastforge/releases) and picks the newest one that ships an archive for the runner platform. It reports this when

- the newest release is still a **draft**. Drafts are invisible to the releases API unless the token has access to the `fastforgedev/fastforge` repository, and the automatic `GITHUB_TOKEN` of another repository does not, or
- that release carries no archive for the runner platform: `install.sh` covers macOS and Linux, `install.ps1` covers Windows, an archive per OS and architecture has to exist in the release.

Fix it by publishing the release and making sure it carries an archive for every platform the workflow builds on. `fastforge-version` only helps when the version is a *published* release, since draft assets cannot be downloaded anonymously.

## License

[MIT](./LICENSE)
