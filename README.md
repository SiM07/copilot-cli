## :warning: Upcoming end-of-support :warning:

AWS Copilot CLI will reach end-of-support on June 12, 2026. After this date, the tool will no longer receive updates, security patches, or technical support. We recommend migrating to alternative solutions as soon as possible to ensure continued support and access to the latest features.
For more information, refer to our [blogpost](https://aws.amazon.com/blogs/containers/announcing-the-end-of-support-for-the-aws-copilot-cli/).

## :warning: About this fork (`SiM07/copilot-cli`)

> [!WARNING]
> This is an **unofficial fork** of [`aws/copilot-cli`](https://github.com/aws/copilot-cli), maintained to keep `copilot deploy` working after upstream's end-of-support.
> It isn't affiliated with or supported by AWS. It only contains the changes listed below.
> The rest of this README is the upstream README: the installation instructions and links below still point to the official `aws/copilot-cli` releases, **which don't include these fixes**.

### Fix: `copilot deploy` hangs after a successful ECS deployment

**Symptom:** the ECS rollout completes and the stack reaches `UPDATE_COMPLETE`, but `copilot deploy` doesn't return. It keeps printing the same progress, and every 30s in CI. It eventually fails with an error such as `ExpiredToken` when the AWS credentials expire, or after the internal 1h30 timeout.

**Cause:** since 2026-09-28, CloudFormation sends an additional `UPDATE_IN_PROGRESS` event on `AWS::ECS::Service` with the reason `Eventual consistency check initiated`, right before `UPDATE_COMPLETE`. Copilot treated it as a new ECS deployment and started a second deployment streamer with the event's timestamp as its start time. The service's primary deployment was last updated *before* that time, so the streamer never considered the rollout done, and the progress renderer waited for it forever.

**Fix** ([`7d42a68f`](https://github.com/SiM07/copilot-cli/commit/7d42a68f49b1a2505ed834df4e9dd6d799149792), [#1](https://github.com/SiM07/copilot-cli/pull/1)): `ecsServiceResourceComponent.Listen()` in `internal/pkg/term/progress/cloudformation.go` ignores events whose reason is `Eventual consistency check initiated`. A unit test replays the real event sequence and checks that only one deployment renderer is created.

> [!CAUTION]
> The fix matches the **exact** status reason sent by CloudFormation. If AWS changes this wording, the hang may come back.
> The deployment itself isn't affected by the bug: a failed `copilot deploy` caused by this hang may still have deployed successfully. Check the ECS service and the stack before retrying or rolling back.

### CI/CD workflows

| Workflow | Trigger | What it does |
|---|---|---|
| [`build.yml`](.github/workflows/build.yml) | Push to any branch, manual run | Runs `make build` and checks that the binary runs (`copilot --version`). |
| [`release.yml`](.github/workflows/release.yml) | Push of a `v*` tag | Runs `make release` and publishes the binaries below with a `checksums.txt` (SHA-256) as a GitHub release. |
| [`ci.yml`](.github/workflows/ci.yml) | Pull requests | Upstream CI: builds, tests, mocks, license and conventional-commit checks. |

Release assets:

| Asset | Platform |
|---|---|
| `copilot-linux-amd64` / `copilot-linux-arm64` | Linux x86_64 / ARM64 |
| `copilot-darwin-amd64` / `copilot-darwin-arm64` | macOS Intel / Apple Silicon |
| `copilot-windows-386.exe` | Windows (32-bit build, runs on x64 and ARM64) |

To publish a release, push a tag with a suffix that doesn't collide with upstream tags. The tag is also the version reported by `copilot --version`:

```sh
git tag v1.34.1-fix.1
git push origin v1.34.1-fix.1
```

> [!WARNING]
> - Binaries aren't signed or notarized. On macOS, a binary downloaded from a browser is blocked by Gatekeeper. Run `xattr -d com.apple.quarantine copilot` to allow it (downloads with `curl` or `gh` aren't affected).
> - The Windows binary is a 32-bit (`386`) build, as produced by the upstream `Makefile`.
> - Releases are public: anything pushed as a `v*` tag is published.

### Reusable action: `setup-copilot`

The composite action [`.github/actions/setup-copilot`](.github/actions/setup-copilot/action.yml) downloads the release asset matching the runner OS and architecture (Linux, macOS, Windows). It verifies the asset against `checksums.txt` and adds `copilot` to the `PATH`.

```yaml
steps:
  - uses: SiM07/copilot-cli/.github/actions/setup-copilot@v1.34.1-fix.1
    with:
      version: v1.34.1-fix.1   # optional, defaults to the latest release
  - run: copilot deploy --name api --env prod
```

| Input | Default | Description |
|---|---|---|
| `version` | `latest` | Release tag to install. |
| `repository` | `SiM07/copilot-cli` | Repository hosting the releases. |
| `token` | `${{ github.token }}` | Token used to download the release assets. |

The action has one output, `version`: the version reported by the installed binary.

> [!NOTE]
> Pin both the action ref (`@<tag>`) and `version` to get reproducible builds. `latest` follows the most recent release of the fork.

### Building locally

The build needs Go 1.23 or later and Node.js. A per-directory Go version can be set with [mise](https://mise.jdx.dev/):

```sh
mise use --env local go@1.23   # creates mise.local.toml (keep it out of git)
make build                     # binary in ./bin/local/copilot
```

The workflows can be run locally with [act](https://github.com/nektos/act):

```sh
act workflow_dispatch -W .github/workflows/build.yml -P ubuntu-latest=catthehacker/ubuntu:act-latest
```

On Apple Silicon, act uses linux/arm64 images. Add `--container-architecture linux/amd64` to match GitHub-hosted runners.

##  <img align="left" alt="AWS Copilot CLI" src="./site/content/assets/images/copilot-logo-48-light.svg" width="85" /> AWS Copilot CLI
###### _Build, Release and Operate Containerized Applications on AWS._ 

![latest version](https://img.shields.io/github/v/release/aws/copilot-cli)
[![Join the chat at https://gitter.im/aws/copilot-cli](https://badges.gitter.im/aws/copilot-cli.svg)](https://gitter.im/aws/copilot-cli?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

* **Documentation**: [https://aws.github.io/copilot-cli/](https://aws.github.io/copilot-cli/)

The AWS Copilot CLI is a tool for developers to build, release and operate production-ready containerized applications
on AWS App Runner or Amazon ECS on AWS Fargate.

Use Copilot to:
* Deploy production-ready, scalable services on AWS from a Dockerfile in one command.
* Add databases or inject secrets to your services.  
* Grow from one microservice to a collection of related microservices in an application.
* Set up test and production environments, across regions and accounts.
* Set up CI/CD pipelines to release your services to your environments.
* Monitor and debug your services from your terminal.

<p align="center">
    <img alt="init" src="./site/content/assets/images/init-cropped.gif" width="600"/>
</p>

## Installation

To install with homebrew:
```sh
$ brew install aws/tap/copilot-cli
```
To install manually, we're distributing binaries from our GitHub releases:

<details>
  <summary>Instructions for installing Copilot for your platform</summary>


| Platform | Command to install |
|---------|---------
| macOS | `curl -Lo copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-darwin && chmod +x copilot && sudo mv copilot /usr/local/bin/copilot && copilot --help` |
| Linux x86 (64-bit) | `curl -Lo copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux && chmod +x copilot && sudo mv copilot /usr/local/bin/copilot && copilot --help` |
| Linux (ARM) | `curl -Lo copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux-arm64 && chmod +x copilot && sudo mv copilot /usr/local/bin/copilot && copilot --help` |
| Windows | `Invoke-WebRequest -OutFile 'C:\Program Files\copilot.exe' https://github.com/aws/copilot-cli/releases/latest/download/copilot-windows.exe` |

</details>


## Getting started

Make sure you have the AWS command line tool installed and have already run `aws configure` before you start.

To get a sample app up and running in one command, run the following:

```sh
$ git clone git@github.com:aws-samples/aws-copilot-sample-service.git demo-app
$ cd demo-app
$ copilot init --app demo                \
  --name api                             \
  --type 'Load Balanced Web Service'     \
  --dockerfile './Dockerfile'            \
  --deploy
```

This will create a VPC, Application Load Balancer, an Amazon ECS Service with the sample app running on AWS Fargate.
This process will take around 8 minutes to complete - at which point you'll get a URL for your sample app running! 🚀

## Learning more 

Want to learn more about what's happening? Check out our documentation [https://aws.github.io/copilot-cli/](https://aws.github.io/copilot-cli/) for a getting started guide, learning about Copilot concepts, and a breakdown of our commands. 

## Feedback

Have any feedback at all? 🙏 Drop us an [issue](https://github.com/aws/copilot-cli/issues/new) or join us on [gitter](https://gitter.im/aws/copilot-cli).

We're happy to hear feedback or answer questions, so reach out, anytime!

## Security disclosures

If you think you’ve found a potential security issue, please do not post it in the Issues. Instead, please follow the instructions [here](https://aws.amazon.com/security/vulnerability-reporting/) or email AWS security directly at [aws-security@amazon.com](mailto:aws-security@amazon.com).

## License
This library is licensed under the Apache 2.0 License.
