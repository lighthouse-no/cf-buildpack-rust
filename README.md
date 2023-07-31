# SAP Cloud Foundry Buildpack for Rust

This buildpack deploys a Rust application to a Cloud Foundry environment.

Although this buildpack was developed specifically for the deployment of Rust programs to an SAP Cloud Foundry environment, it does not contain any SAP-specific dependencies.
Therefore, it should be suitable for deploying a Rust program to ***any*** Cloud Foundry environment; however, this aspect has not been tested.

## Assumptions

The objective of this buildpack is to compile your Rust program such that it will run as an application in your Cloud Foundry space.
Consequently, your `Cargo.toml` file should specify a single package that compiles to a single binary.

Adding configuration to `Cargo.toml` that causes `cargo build` to create multiple binaries will not have the desired outcome, because the `release` phase of this buildpack must supply Cloud Foundry with the name of the single binary to be executed &mdash; and that binary is identified by the package name.

## Build Phases

This buildpack uses all 4 standard phases:

1. `detect`<br>If the file `Cargo.toml` exists in the build directory, the string `Rust` is returned with an exit code of `0`, else exit code `1` is returned and no further build phases are performed.
1. `supply`<br>Installs or updates the version of the Rust toolchain defined in file `RustToolchain`.<br>If this file is missing, the default value of `stable` is used.
1. `finalize`<br>Runs `cargo build` using any additional build settings found in the file `RustConfig`.<br>If this file is missing or empty, it simply runs `cargo build --release`
1. `release`<br>Returns a YAML string that points Cloud Foundry to the compiled binary.

## Usage

### `manifest.yml`

In the `manifest.yml` of your application, point to this `buildpack` and allocate sufficient memory for compilation to succeed.

```yaml
---
applications:
- name: my-cool-rust-project
  memory: 4096M
  buildpacks:
  - https://github.com/lighthouse-no/cf-buildpack-rust
```

### `Cargo.toml`

In your application's `Cargo.toml` file, make sure the `name` property is listed immediately after the `[package]` section on line 2 of the file.

```toml
[package]
name = "my-cool-rust-app"
version = "0.1.0"
authors = ["Chris Whealy <chris@lighthouse.no>"]
edition = "2021"

```

This is because the buildpack's `release` phase assumes that the `name` property can be found exactly on line 2.

## Configuring the Rust Toolchain

In the vast majority of cases, you will want your application built using the `stable` Rust channel.
If this is the case, then no explicit configuration is needed.

However, should you wish to use either the `nightly` channel, or pin your application to a specific Rust version, then create a file called `RustToolchain` in your repo's top level directory containing the specific toolchain name you wish to use.
For example:

```sh
$ cat RustToolchain
nightly
```

If the `RustToolchain` file exists but is empty, it will be ignored.
You should only add a single value to this file.

See [`Rust toolchains`](https://rust-lang.github.io/rustup/concepts/toolchains.html) for more details about Rust channels.

## Configuring Cargo Environment Variables

Any environment variables used by `cargo` can be defined in a file called `RustConfig` in your repo's top level directory.
For instance:

```sh
$ cat RustConfig
RUST_LIB_BACKTRACE=1
RUST_BACKTRACE=full
RUST_LOG=warn
```

If the `RustConfig` file exists but is empty, it will be ignored.
Otherwise, you may specify as many `cargo` environment variables as needed.

***WARNING***<br>
Do not set your own value for `CARGO_TARGET_DIR` as the buildpack's `finalize` phase defines its own value for this variable.

`RustConfig` may also contain additional variables used by this buildpack:

| `RustConfig` Variable | Default Value | Description
|---|---|---
| `RUST_CARGO_BUILD_PROFILE` | `"release"` | Rust build profile
| `RUST_CARGO_BUILD_FLAGS` | `""` | Optional build flags.<br>For example `"--features feature1 feature2"`

See [Rust Environment Variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html) for more details.

### Build Profiles
For all build profiles other than `release`, it is good practice to add an explicit profile definition in `Cargo.toml`; for example:

```toml
[profile.my_dev]
opt-level = 1
debug = true
debug-assertions = true
overflow-checks = true
lto = false
panic = 'unwind'
incremental = true
codegen-units = 256
rpath = false
```

Then add the line `RUST_CARGO_BUILD_PROFILE=my_dev` to `RustConfig`.

### Build Command

If `$RUST_CARGO_BUILD_PROFILE == release`, then the follow build command is used:

```sh
cargo build --release $RUST_CARGO_BUILD_FLAGS
```

For any other value of `$RUST_CARGO_BUILD_PROFILE`, this command will be used:

```sh
cargo build --profile $RUST_CARGO_BUILD_PROFILE $RUST_CARGO_BUILD_FLAGS
```

## Testing with Docker

Changes to the buildpack can be tested using the included shell script `test-buildpack.sh`.
This uses the standard `rust` Docker image.

-----
&copy; 2023 Lighthouse Consulting AS
