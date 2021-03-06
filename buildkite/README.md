# Bazel Continuous Integration

Bazel uses [Buildkite] for continuous integration. The user interface and the orchestration of CI
builds is fully managed by Buildkite, but Bazel brings its own CI machines. The [buildkite folder]
contains all the scripts and configuration files necessary to setup Bazel's CI on Buildkite.

## Bazel on Buildkite 101

When you first log into [Buildkite] you are presented with a list of pipelines. A pipeline is a
template of steps that are executed either in sequence or in parallel and that all need to succeed in
order for the pipeline to succeed. The Bazel organisation has dozens of pipelines. Here are a selected
few:

![pipelines]

* The *bazel postsubmit* pipeline builds and tests each commit to Bazel's repository on all supported
platforms.
* The *bazel presubmit* pipeline is triggered on every pull request to Bazel.
* The *rules_go postsubmit* pipeline is triggered on every commit to the [rules_go] repository.
* The *TensorFlow* pipeline builds and tests TensorFlow at `HEAD` every four hours.

### Builds

When you click on a pipeline you can see the last few builds of this pipeline. Clicking on a build
then gives you access to the details of the build. For example, the below image shows a failed build
step on Ubuntu 16.04.

![failed build step]

One can see which tests failed by clicking on the *Test* section. In the below example, the
`//src/test/shell/bazel:external_path_test` was flaky as it failed in 1 out of 5 runs.

![flaky test]

You can view the failed test attempt's `test.log` file in the *Artifacts* tab.

![flaky test log]

### Useful Links

![buildkite useful buttons]

## Pull Requests

Bazel accepts contributions via pull requests. Contributions by members of the [bazelbuild]
organisation as well as members of individual repositories (i.e. rule maintainers) are whitelisted
automatically and will immediately be built and tested on [Buildkite].

An external contribution, however, first needs to be verified by a project member and therefore will
display a pending status named *Verify Pull Request*.

![status verify pull request]

A member can verify a pull request by clicking on *Details*, followed by *Verify Pull Request*.

![buildkite verify pull request]

*Please vet external contributions carefully as they can execute arbitrary code on our CI machines*

### Build and Test Results

After a pull request has been built and tested, the results will be displayed as a status message on
the pull request. A detailed view is available when clicking on the corresponding *Details*
link. Click [here](https://source.cloud.google.com/results/invocations/dc0510c0-afc6-42b3-8d2e-6d879dec526a/targets)
for an example.

![pull request details]

[Buildkite]: https://buildkite.com
[buildkite folder]: https://github.com/bazelbuild/continuous-integration/tree/master/buildkite
[rules_go]: https://github.com/bazelbuild/rules_go
[Bazel]: https://github.com/bazelbuild/bazel
[bazelbuild]: https://github.com/bazelbuild/

[pipelines]: https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/docs/assets/pipelines.png
[failed build step]: https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/docs/assets/failed-build-step.png
[flaky test]: https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/docs/assets/flaky-test.png
[flaky test log]: https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/docs/assets/flaky-test-log.png
[status verify pull request]: https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/docs/assets/status-verify-pull-request.png
[buildkite verify pull request]: https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/docs/assets/buildkite-verify-pull-request.png
[pull request details]: https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/docs/assets/pull-request-details.png
[buildkite useful buttons]: https://raw.githubusercontent.com/bazelbuild/continuous-integration/master/buildkite/docs/assets/buildkite-useful-buttons.png


### Culprit Finder

[Bazel downstream projects](https://buildkite.com/bazel/bazel-with-downstream-projects-bazel) is red? Use culprit finder to find out which bazel commit broke it!

First you should check if the project is green with the latest Bazel release. If not, probably it's their commits that broke the CI.

If a project is green with release Bazel but red with Bazel nightly, it means some Bazel commit broke it, then culprit finder can help!

Create "New Build" in the [Culprit Finder](https://buildkite.com/bazel/culprit-finder) project with the following environment variable:

- PROJECT_NAME (The project name must exist in DOWNSTREAM_PROJECTS in [bazelci.py](https://github.com/bazelbuild/continuous-integration/blob/master/buildkite/bazelci.py))
- TASK_NAME (The task name must exist in the project's config file, eg. [macos_latest](https://github.com/bazelbuild/rules_apple/blob/master/.bazelci/presubmit.yml#L3)). For old config syntax where platform name is essentially the task name, you can also set PLATFORM_NAME instead of TASK_NAME.
- GOOD_BAZEL_COMMIT (A full Bazel commit, Bazel built at this commit still works for this project)
- BAD_BAZEL_COMMIT (A full Bazel commit, Bazel built at this commit fails with this project)
- (Optional) NEEDS_CLEAN (Set NEEDS_CLEAN to `true` to run `bazel clean --expunge` before each build, this will help reduce flakiness)
- (Optional) REPEAT_TIMES (Set REPEAT_TIMES to run the build multiple times to detect flaky build failure, if at least one build fails we consider the commit as bad)


eg.
```
PROJECT_NAME=rules_go
PLATFORM_NAME=ubuntu1404
GOOD_BAZEL_COMMIT=b6ea3b6caa7f379778e74da33d1bd0ff6477f963
BAD_BAZEL_COMMIT=91eb3d207714af0ab1e5812252a0f10f40d6e4a8
```

Note: Bazel commit can only be set to commits after [63453bdbc6b05bd201375ee9e25b35010ae88aab](https://github.com/bazelbuild/bazel/commit/63453bdbc6b05bd201375ee9e25b35010ae88aab), Culprit Finder needs to download Bazel at specific commit, but we didn't prebuilt Bazel binaries before this commit.

## Configuring a Pipeline

Each pipeline is configured via a Yaml file. This file either lives in `$PROJECT_DIR/.bazelci/presubmit.yml` (for presubmits) or in an arbitrary location whose path or URL is passed to the CI script (as configured in the Buildkite settings of the respective pipeline). Projects should store the postsubmit configuration in their own repository, but we keep some configurations for downstream projects in https://github.com/bazelbuild/continuous-integration/tree/master/buildkite/pipelines.

### Basic Syntax

The most important piece of the configuration file is the `tasks` dictionary. Each task has a unique key, a platform and usually some build and/or test targets:

```yaml
---
tasks:
  ubuntu_build_only:
    platform: ubuntu1604
    build_targets:
    - "..."
  windows:
    platform: windows
    build_targets:
    - "..."
    test_targets:
    - "..."
```

If there is exactly one task per platform, you can omit the `platform` field and use its value as task ID instead. The following code snippet is equivalent to the previous one:

```yaml
---
tasks:
  ubuntu1604:
    build_targets:
    - "..."
  windows:
    build_targets:
    - "..."
    test_targets:
    - "..."
```

### Advanced Features

There are more advanced mechanisms for configuring individual tasks:

- The CI script can run arbitrary commands before building or testing.
  Simply add the `batch_commands` (Windows) or `shell_commands` field (all other platforms).
- The `run_targets` field contains a list of Bazel targets that should be run before building.
- The `build_flags` and `test_flags` fields contain lists of flags that should be used when building or testing (respectively).
- You can specify a display name in the `name` field. The resulting job will display this value and an emoji that represents the platform.

```yaml
---
tasks:
  ubuntu1404:
    shell_commands:
    - rm -f obsolete_file
    run_targets:
    - "//whatever"
    build_flags:
    - "--define=ij_product=clion-latest"
    build_targets:
    - "..."
    test_flags:
    - "--define=ij_product=clion-latest"
    test_targets:
    - ":clwb_tests"
  windows:
    name: "some :emoji:"
    batch_commands:
    - powershell -Command "..."
    build_targets:
    - "..."
```

### Legacy Format

Most existing configuration use the legacy format with a "platforms" dictionary:

```yaml
---
platforms:
  ubuntu1404:
    build_targets:
    - "..."
    test_targets:
    - "..."
```

The new format expects a "tasks" dictionary instead:

```yaml
---
tasks:
  arbitrary_id:
    platform: ubuntu1404
    build_targets:
    - "..."
    test_targets:
    - "..."
```

In this case we can omit the `platforms` field since there is a 1:1 mapping between tasks and platforms. Consequently, the format looks almost identical to the old one:

```yaml
---
tasks:
  ubuntu1404:
    build_targets:
    - "..."
    test_targets:
    - "..."
```

The CI script still supports the legacy format, too.

### Using a specific version of Bazel

The CI uses [Bazelisk](https://github.com/philwo/bazelisk) to support older versions of Bazel, too. You can specify a Bazel version for each pipeline (or even for individual platforms) in the pipeline Yaml configuration:

```yaml
---
bazel: 0.20.0
platforms:
  windows:
    build_targets:
    - "..."
  macos:
    build_targets:
    - "..."
  ubuntu1404:
    bazel: 0.18.0
    build_targets:
    - "..."
[...]
```
In this example the jobs on Windows and MacOS would use 0.20.0, whereas the job on Ubuntu would run 0.18.0.
Please see the [Bazelisk documentation](https://github.com/philwo/bazelisk/blob/master/README.md#how-does-bazelisk-know-which-version-to-run) for a list of all supported version values.

### Running Buildifier on CI

For each pipeline you can enable [Buildifier](https://github.com/bazelbuild/buildtools/tree/master/buildifier) to check all WORKSPACE, BUILD, BUILD.bazel and .bzl files for lint warnings and formatting violations. Simply add the following code to the top of the particular pipeline configuration:

```yaml
---
buildifier: latest
[...]
```

As a consequence, every future build for this pipeline will contain an additional "Buildifier" step that runs the latest version of Buildifier both in "lint" and "check" mode.
Alternatively you can specify a particular Buildifier version such as "0.20.0".

There is also a more advanced syntax that allows you to specify which [warnings](https://github.com/bazelbuild/buildtools/blob/master/WARNINGS.md) should be checked in [lint mode](https://github.com/bazelbuild/buildtools/tree/master/buildifier#linter):

```yaml
---
buildifier:
  version: latest
  warnings: "positional-args,duplicated-name"
[...]
```

### Using multiple Workspaces in a single Pipeline

Some projects may contain one or more `WORKSPACE` files in subdirectories, in addition to their top-level `WORKSPACE` file.
All of these workspaces can be tested in a single pipeline by using the `working_directory` task property.
Consider the configuration for a project that contains a second `WORKSPACE` file in the `examples_dir/` directory:

```yaml
---
tasks:
  production_code:
    name: "My Project"
    platform: ubuntu1804
    test_targets:
    - //...
  examples:
    name: Examples
    platform: ubuntu1804
    working_directory: examples_dir
    test_targets:
    - //...
```
