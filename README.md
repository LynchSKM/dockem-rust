# DOCKEM-RS

![Dockem](docs/logo.png)

A Rust CLI that helps optimise the build process of Docker containers, if the image has
been built before, it will just
copy the tag across on the registry which is much faster than pulling and pushing to the
new image tag.

[![Build Status](https://GitHub.com/LynchSKM/dockem-rs/actions/workflows/build.yaml/badge.svg?branch=main)](https://GitHub.com/LynchSKM/dockem-rs/actions/workflows/build.yaml)
[![Unit Tests](https://GitHub.com/LynchSKM/dockem-rs/actions/workflows/testing.yaml/badge.svg?branch=main)](https://GitHub.com/LynchSKM/dockem-rs/actions/workflows/testing.yaml)

This repository was a challenge to rewrite [dockem](https://GitHub.com/kerren/dockem)
from
Golang to Rust. It is my first Rust CLI and I may have not done it correctly. I aim to
revisit my implementation later on and make any Rust specific improvements to make it even
better.

# The Argument

For my full argument, refer to [The Long Argument](#the-long-argument) below. But in
short, you can control what files or directories get hashed and only trigger a rebuild if
they change. If there is no change, the `cli` will quickly copy the tag to the new one on
the registry and you'll be good to go with your new tag.

# Getting Started

This library has been built in `go` in order for me to be able to build binaries for a
bunch of different platforms. The easiest way to use this would be to go to
the [Releases Page](https://GitHub.com/LynchSKM/dockem-rs/releases) and download the
binary
that suits you.

## Quick Install

I've created a quick install script that will download the latest version of the binary
for you. This is still a work in progress and I need to test this on Mac and Windows. If
you're running a Linux system, this should work without a hassle. For the other systems, I
suggest you check out the [Releases Page](https://GitHub.com/LynchSKM/dockem-rs/releases)
and
download the binary from there.

```shell
curl -s https://raw.githubusercontent.com/LynchSKM/dockem-rs/main/scripts/get_dockem.sh | bash
```

Note, the above script **requires sudo** to move the binary to `/usr/local/bin`. If you
don't want to use sudo, you can download the binary to the current directory using,

```shell
curl -s https://raw.githubusercontent.com/LynchSKM/dockem-rs/main/scripts/get_dockem_local.sh | bash
```

### Installing a Specific Version

If you're running an x86_64/amd64 Linux system and don't want the hassle of figuring
things out,
you can use the quick installation script by running the following in terminal.

```shell
wget https://GitHub.com/LynchSKM/dockem-rs/releases/download/v1.1.1/dockem-v1.1.1-linux-x86_64
chmod 755 dockem-rs-v1.1.1-linux-x86_64
sudo mv ./dockem-rs-v1.1.1-linux-x86_64 /usr/local/bin/dockem-rs
```

If you're running an aarch64/arm64 Linux system, you can run the following,

```shell
wget https://GitHub.com/LynchSKM/dockem-rs/releases/download/v1.1.1/dockem-v1.1.1-linux-aarch64
chmod 755 dockem-rs-v1.1.1-linux-aarch64
sudo mv ./dockem-rs-v1.1.1-linux-aarch64 /usr/local/bin/dockem-rs
```

## Usage

```
Usage:
  dockem-rs build [flags]


Flags:
  -d, --directory string                 (required) The directory that should be used as the context for the Docker build (default "./")
  -p, --docker-password string           The password that should be used to authenticate the docker client. Ignore if you have already logged in.
  -u, --docker-username string           The username that should be used to authenticate the docker client. Ignore if you have already logged in.
  -f, --dockerfile-path string           (required) The path to the Dockerfile that should be used to build the image (default "./Dockerfile")
  -h, --help                             help for build
  -I, --ignore-build-directory           Whether to ignore the build directory in the hashing process, this is useful when you are watching a specific file or directory.
  -i, --image-name string                (required) The name of the image you are building
  -l, --latest                           Whether to push the latest tag with this image
  -m, --main-version                     Whether to push this as the main version of the repository. This is done automatically if you do not specify tags or the latest flag.
  -r, --registry string                  The registry that should be used when pulling/pushing the image, Dockerhub is used by default
  -t, --tag stringArray                  The tag or tags that should be attached to image
  -F, --version-file string              (required) The name of the JSON file that holds the version to be used in the build. This JSON file must have the 'version' key. (default "./package.json")
  -W, --watch-directory stringArray      Watch for changes in a directory or directories
  -w, --watch-file stringArray           Watch for changes on a specific file or files

```

Here on some examples on how you would use this `cli`,

```shell
dockem-rs build --directory=./apps/backend --dockerfile-path=./devops/prod/backend/Dockerfile --image-name=my-repo/backend --tag=stable --main-version

dockem-rs build --directory=./apps/backend --watch-directory=./libs/shared --dockerfile-path=./apps/backend/Dockerfile --image-name=my-repo/backend --tag=dev --latest

dockem-rs build --image-name=my-repo/backend --registry=eu.reg.io --docker-username=uname --docker-password=1234 --tag=alpha --tag=test
```

## Usage in Actions

I've also created a GitHub action for this, check
out [LynchSKM/setup-dockem-rs](https://GitHub.com/LynchSKM/setup-dockem-rs) to see
details. In
essence, you'll just need to add the following to your pipeline,

```yaml
    - name: Setup Dockem-rs
      uses: LynchSKM/setup-dockem-rs@v1

    - name: Run Dockem
      run: dockem build --directory=./apps/backend --dockerfile-path=./devops/prod/backend/Dockerfile --image-name=my-repo/backend --tag=stable --main-version
```

## Concepts

In this section, I'll run through the different concepts to fully explain the `cli` and
how it can be used.

### The Version File

The version file is a `JSON` file that holds a `"version"` key. The version inside the key
could be anything, however, it's most likely generated using semantic versioning. When a
build is run, this version is extracted from the key and added to the tag.

*NOTE*: The version should not start with a `v` as this is added automatically.

An example of the version file is as follows,

```json
{
  "version": "1.0.0"
}
```

### Ignore Build Directory

In most cases (I think), you'd want to trigger a build when the build directory hash has
changed. However, there are times that you may not want to do that, and instead you would
like to watch the hash of other directories or files.

In this case, you can use the `--ignore-build-directory` flag to ignore the build
directory in the hashing process.

An example of where this may be useful is if you build base images that other Docker
images use in the `FROM` statement. In this case, you may only want to trigger a build
when the `Dockerfile` changes and not the code that is copied into the base image.

### Main Version

The `--main-version` flag is used to specify that this build should be the main version of
the repository.

So for instance, if you have an image called `example-org/backend` and you use the
`--main-version` flag, it would push the following image to the registry,

```
example-org/backend:v1.0.0
```

Assuming the version in the version file is `1.0.0`.

### Latest

The `--latest` flag is used to specify that this build should be the latest version of the
repository.

So for instance, if you have an image called `example-org/backend` and you use the
`--latest` flag, it would push the following image to the registry,

```
example-org/backend:latest
```

### Watch File / Watch Directory

The hash is generated from the files and/or directories you specify. You can specify as
many as you'd like.

When you use the `--watch-file` and/or the `--watch-directory` flags, the build will
trigger whenever something in the specified files or directories change.

An example of where this might be useful is if you have a base image that other
`Dockerfiles` build from. You may only want to watch the `package-lock.json` file or some
other lock file to trigger a build because you don't care about the source but you do care
when the base dependencies change.

### Tag

The `--tag` flag can be used to push to a specific tag on the image. At the moment, the
version is appended to the tag before it pushes.

So for instance, if you have an image name `example-org/backend` and you use the
`--tag=alpha` flag, it would push the following image to the registry,

```
example-org/backend:alpha-v1.0.0
```

Assuming the version in the version file is `1.0.0`.

# Roadmap

There are a few tweaks and features I'd like to implement to improve the overall project.

- [x] Create a GitHub Action that pulls the `dockem-rs` binary
- [x] Add to documentation on how to install for different platforms, like ARM and Windows
- [x] Add more examples to the documentation on how to use the `cli` effectively
- [ ] Create end-to-end tests to ensure the core is working, this allows for faster
  refactoring and feature development
- [ ] Test that the CLI can read docker `config.json` files on a server if docker username
  and password are not given.
- [ ] Setup a Custom GitHub runner that can build the Darwin (macOS) versions. Steps to do
  so can be found in
  this [article](https://blog.crafteo.io/2024/02/29/my-rust-cross-compilation-journey/).
- [ ] Add a Homebrew tap
- [ ] Add the ability to enable `buildx` caching for GitHub Actions. This could make the
  builds faster in the future.
- [ ] Add the ability to specify the platform(s) you'd like to build for using a `buildx`
  builder. This would be cool to be able to build ARM images using a standard runner. For
  now, I recommend deploying a custom ARM runner and building on that (it'll also be a lot
  faster)

# The Long Argument

So now you may ask, why? What's the point?

I've always found Docker builds to be quite slow (and frustrating), especially when the
build doesn't even need to take place because nothing has changed but the action triggers
on push or when you open a PR. At this point you may say, "well why don't you enable
caching using `buildx` and let that speed it up for you?". And to be fair, that is a valid
question. It does make the builds faster because all of the layers are cached, BUT, why
even push to the registry if you don't have to?

So that's when I started thinking, why don't we push the hash of whatever we want to
trigger a build as a tag to the registry? At that point, if the hash is the same, there is
no need to rebuild. If there is a different hash, then we trigger a build and push the new
hash in the process.

What I really love about this is that we can choose what "changes" the hash. It doesn't
even have to be code in your Docker image. For instance, if you have a "base" image that
other images extend off, potentially, you'll only want to trigger a build when the
`Dockerfile` changes, not the code that would be copied into the base because maybe that's
on different layers further up.

---

## Development

This section is helpful if you want to pull down the code and make your own changes.

#### Pre-requisites

* [rustup](https://rustup.rs/#)
* [cross](https://GitHub.com/cross-rs/cross)
* [cargo-set-version](https://crates.io/crates/cargo-set-version)

### Release

To tag a new version run the following on the main branch:

```shell
task release
```

or

```shell
task release-major
```

### Build

Make sure [cross](https://GitHub.com/cross-rs/cross) is installed to help with
cross-platform compilation. When ready, run:

```shell
task build
```

#### Supported Platforms

* Linux x86_64 / Linux amd64
* Linux aarch64 / Linux arm64
* Windows

I wanted to support Darwin (macOS) Targets, but it seemed like too much effort for now.
Maybe in a future release I will work on figuring it out by following
this [article](https://blog.crafteo.io/2024/02/29/my-rust-cross-compilation-journey/).
