# rust-vmm-ci

The `rust-vmm-ci` repository contains [integration tests](#integration-tests)
and [Buildkite pipeline](#buildkite-pipeline) definitions that are used for
running the CI for all rust-vmm crates.

Having a centralized place for the tests is one of the enablers for keeping the
same quality standard for all crates in rust-vmm.

## Getting Started with rust-vmm-ci

To run the integration tests defined in the pipeline as part of the CI:

1. Add rust-vmm-ci as a git submodule to your repository

```bash
# Add rust-vmm-ci as a submodule. This will point to the latest rust-vmm-ci
# commit from the master branch. The following command will also add a
# `.gitmodules` file and the `rust-vmm-ci` to the index.
git submodule add https://github.com/rust-vmm/rust-vmm-ci.git
# Commit the changes to your repository so that the CI can run using the
# rust-vmm-ci pipeline and tests.
git commit -s -m "Added rust-vmm-ci as submodule"
```

1. Create the coverage test configuration file named
`coverage_config_ARCH.json` in the root of the repository, where `ARCH` is the
architecture of the machine.
There are two coverage test configuration files, one per each platform.
The example of the configuration file for the `x86_64` architecture can be
found in
[coverage_config_x86_64.json.sample](coverage_config_x86_64.json.sample),
and the example of the configuration file for the `aarch64` architecture can be
found in
[coverage_config_aarch64.json.sample](coverage_config_aarch64.json.sample).

The json must have the following fields:

- `coverage_score`: The coverage of the repository.
- `exclude_path`: This field is used for excluding files from the report. It
  should be used to exclude autogenerated files. Files in `exclude_path` are
  separated by one comma. If the repository does not have any autogenerated
  files, `exclude_path` should be an empty string.
- `crate_features`: `cargo kcov` does not build crate features by default. To
  get the coverage report including optional features, these need to be
  specified in `crate_features` separated by comma. If the crate does not have
  any features, this field should be empty.

This file is required for the coverage integration so it needs to be added
to the repository as well.

1. Create a new pipeline definition in Buildkite. For this step ask one of the
rust-vmm Buildkite [admins](CODEOWNERS) to create one for you. Add a pipeline
step that is uploading the rust-vmm-ci pipeline:

```bash
buildkite-agent pipeline upload rust-vmm-ci/.buildkite/pipeline.yml
```

1. The code owners of the repository will have to setup a WebHook for
triggering the CI on
[pull request](https://developer.github.com/v3/activity/events/types/#pullrequestevent)
and [push](https://developer.github.com/v3/activity/events/types/#pushevent)
events.

## Buildkite Pipeline

The [Buildkite](https://buildkite.com) pipeline is the definition of tests to
be run as part of the CI. It includes steps for running unit tests and linters
(including coding style checks), and computing the coverage.

Currently the tests can run on Linux `x86_64` and `aarch64` hosts.

Example of step that checks the build:

```yaml
steps:
  - label: "build-gnu-x86"
    commands:
     - cargo build --release
    retry:
      automatic: false
    agents:
      platform: x86_64.metal
    plugins:
      - docker#v3.0.1:
          image: "rustvmm/dev:v${LATEST}"
          always-pull: true
```

To see all steps in the pipeline check the
[.buildkite/pipeline.yml](.buildkite/pipeline.yml) file.

### Custom Pipeline

Some crates might need to test functionality that is specific to that
particular component and thus cannot be added to the common pipeline.

In this situation, the repositories need to create a custom pipeline (besides
the rust-vmm-ci pipeline) and add it in the repository. The preferred path for
the custom pipeline is `.buildkite/pipeline.yml`.

For example to test the build with one non-default
[feature](https://doc.rust-lang.org/1.19.0/book/first-edition/conditional-compilation.html)
enabled, the following step can be added in the custom pipeline under
`.buildkite/pipeline.yml`.

```yaml
steps:
  - label: "build-gnu-x86-bzimage"
    commands:
     - cargo build --release --features bzimage
    retry:
      automatic: false
    agents:
      platform: x86_64.metal
    plugins:
      - docker#v3.0.1:
          image: "rustvmm/dev:${LATEST}"
always-pull: true
```

### Custom Repository Hooks

The integration tests of some repositories have dependencies on external
resources. One example is
[`linux-loader`](https://github.com/rust-vmm/linux-loader/) which needs to
download a bzImage before running the unit tests. Because this is specific
to the `linux-loader` crate, the logic for downloading the required resources
cannot be part of the common pipeline. The mechanism used here is
[Repository Hooks](https://buildkite.com/docs/agent/v3/hooks#repository-hooks).
The hooks are defined per repository and live in the crate repository under
`.buildkite/hooks`.

Example of post-checkout hook that downloads and extracts a bzImage:

```bash
#!/bin/bash

DEB_NAME="linux-image-4.9.0-9-amd64_4.9.168-1_amd64.deb"
DEB_URL="http://ftp.debian.org/debian/pool/main/l/linux/${DEB_NAME}"

REPO_PATH="${BUILDKITE_BUILD_CHECKOUT_PATH}"
DEB_PATH="${REPO_PATH}/${DEB_NAME}"
EXTRACT_PATH="${REPO_PATH}/src/bzimage-archive"
BZIMAGE_PATH="${EXTRACT_PATH}/boot/vmlinuz-4.9.0-9-amd64"

mkdir -p ${EXTRACT_PATH}

wget ${DEB_URL} -P ${REPO_PATH}
dpkg-deb -x ${DEB_PATH} ${EXTRACT_PATH}

mv ${BZIMAGE_PATH} ${REPO_PATH}/src/bzimage
rm -r ${EXTRACT_PATH}
rm -f ${DEB_PATH}
```

In this example the post-checkout hook downloads a deb image, extracts its
contents and places it in `linux-loader/src/bzimage`. The unit tests will use
the relative path `src/bzimage` which does not depend on the image
being downloaded.

## Integration Tests

In addition to the one-liner tests defined in the
[Buildkite Pipeline](#buildkite-pipeline), the rust-vmm-ci also has more
complex tests defined in [integration_tests](integration_tests).

### Test Profiles

The integration tests support two test profiles:

- **devel**: this is the recommended profile for running the integration tests
  on a local development machine.
- **ci** (default option): this is the profile used when running the
  integration tests as part of the the Continuous Integration (CI).

The test profiles are applicable to
[`pytest`](https://docs.pytest.org/en/latest/), the integration test framework
used with rust-vmm-ci. Currently only the
[coverage test](tests/test_coverage.py) follows this model as all the other
integration tests are run using the Buildkite pipeline.

The difference between is declaring tests as passed or failed:

- with the **devel** profile the coverage test passes if the current coverage
  is equal or higher than the upstream coverage value. In case the current
  coverage is higher, the coverage file is updated to the new coverage value.
- with the **ci** profile the coverage test passes only if the current coverage
  is equal to the upstream coverage value.

Further details about the coverage test can be found in the
[Adaptive Coverage](#adaptive-coverage) section.

### Adaptive Coverage

The line coverage is saved in [tests/coverage](tests/coverage). To update the
coverage before submitting a PR, run the coverage test:

```bash
crate="kvm-ioctls"
container_version=5
docker run --device=/dev/kvm \
           -it \
           --security-opt seccomp=unconfined \
           --volume $(pwd)/${crate}:/${crate} \
           rustvmm/dev:v{$container_version}
cd ${crate}
pytest --profile=devel rust-vmm-ci/integration_tests/test_coverage.py
```

If the PR coverage is higher than the upstream coverage, the coverage file
needs to be manually added to the commit before submitting the PR:

```bash
git add tests/coverage
```

Failing to do so will generate a fail on the CI pipeline when publishing the
PR.

**NOTE:** The coverage file is only updated in the `devel` test profile. In
the `ci` profile the coverage test will fail if the current coverage is higher
than the coverage reported in [tests/coverage](tests/coverage).

### Performance tests

`rust-vmm-ci` includes an integration test that can run a battery of
benchmarks at every pull request, comparing the results with the tip of the
upstream `master` branch. The test is not included in the default Buildkite
pipeline. Each crate that requires the test to be run as part of the CI must
add a [custom pipeline](#custom-pipeline).

An example of a pipeline that runs the test for ARM platforms and prints the
results:

```yaml
steps:
  - label: "bench-aarch64"
    commands:
      - pytest rust-vmm-ci/integration_tests/test_benchmark.py -s
    retry:
      automatic: false
    agents:
      platform: arm.metal
    plugins:
      - docker#v3.0.1:
          image: "rustvmm/dev:v${LATEST}"
          always-pull: true
```

The test requires [`criterion`](https://github.com/bheisler/criterion.rs)
benchmarks to be exported by the crate. The test expects the entry point
into the performance benchmarks to be named `main`. In other words, the
following configuration is expected in `Cargo.toml`:

```toml
[[bench]]
name = "main"
```

All benchmarks need to be collected in a main.rs file placed in `benches/`.

`criterion` collects performance results by running a function for a
user-configured number of iterations, timing the runs, and applying statistics.
The individual benchmark tests must be added in the crate. They can be run
outside the CI with:

```bash
cargo bench [--all-features] OR [--features <features>]
```

`rust-vmm-ci` uses [`critcmp`](https://github.com/BurntSushi/critcmp) to
compare the results yielded by `cargo bench --all-features` on the PR being
tested with those from the tip of the upstream `master` branch. The test
runs `cargo bench` twice, once on the current `HEAD`, then again after
`git checkout origin/master`. `critcmp` takes care of the comparison, making
use of `criterion`'s stable format for
[output files](https://bheisler.github.io/criterion.rs/book/user_guide/csv_output.html).
The results are printed to `stdout` and can be visually inspected in the
pipeline output. In its present form, the test cannot fail.

To run the test locally:

```bash
docker run --device=/dev/kvm \
           -it \
           --security-opt seccomp=unconfined \
           --volume $(pwd)/${CRATE}:/${CRATE} \
           rustvmm/dev:v${LATEST}
cd ${CRATE}
pytest rust-vmm-ci/integration_tests/test_benchmark.py -s
```

Note that performance is highly dependent on the underlying platform that the
tests are running on. The raw numbers obtained are likely to differ from their
counterparts on a CI instance.
