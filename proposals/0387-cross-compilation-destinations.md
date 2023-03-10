# Cross-Compilation Destination Bundles

- Proposal: [SE-0387](0387-cross-compilation-destinations.md)
- Authors: [Max Desiatov](https://github.com/MaxDesiatov), [Saleem Abdulrasool](https://github.com/compnerd/), [Evan
  Wilde](https://github.com/etcwilde)
- Review Manager: [Mishal Shah](https://github.com/shahmishal)
- Status: **Active Review (January 31st...Feburary 14th, 2023**)
- Implementation: [apple/swift-package-manager#5911](https://github.com/apple/swift-package-manager/pull/5911),
  [apple/swift-package-manager#5922](https://github.com/apple/swift-package-manager/pull/5922), [apple/swift-package-manager#6023](https://github.com/apple/swift-package-manager/pull/6023)
- Review: ([pitch](https://forums.swift.org/t/pitch-cross-compilation-destination-bundles/61777)) ([review](https://forums.swift.org/t/se-0387-cross-compilation-destination-bundles/62875))

## Introduction

Cross-compilation is a common development use case. When cross-compiling, we need to refer to these concepts:

- **toolchain** is a set of tools used to build an application or a library;
- **triple** describes features of a given machine such as CPU architecture, vendor, OS etc, corresponding to LLVM's
  triple;
- **build-time triple** describes a machine where application or library code is built;
- **run-time triple** describes a machine where application or library code is running;
- **SDK** is a set of dynamic and/or static libraries, headers, and other resources required to generate code for the
  run-time triple.

Let’s call a toolchain and an SDK bundled together a **destination**.

Authors of the proposal are aware of the established "build/host/target platform" naming convention, but feel that
"target" already has a different meaning within the build systems nomenclature. In addition, "platform"
itself is quite loosely defined. For the avoidance of possible confusion, we're using "build-time triple" and "run-time
triple" terms in this proposal.

## Motivation

Swift cross-compilation (CC) destinations are currently produced on an ad-hoc basis for different combinations of build-time
and run-time triples. For example, scripts that produce macOS → Linux CC destinations were created by both [the Swift
team](https://github.com/apple/swift-package-manager/blob/swift-5.7-RELEASE/Utilities/build_ubuntu_cross_compilation_toolchain)
and [the Swift community](https://github.com/SPMDestinations/homebrew-tap). At the same time, the distribution process
of CC destinations is cumbersome. After building a destination tree on the file system, required metadata files rely on
hardcoded absolute paths. Adding support for relative paths in destination's metadata and providing a unified way to
distribute and install destinations as archives would clearly be an improvement to the multi-platform Swift ecosystem.

The primary audience of this pitch are people who cross-compile from macOS to Linux. When deploying to single-board
computers supporting Linux (e.g. Raspberry Pi), building on such hardware may be too slow or run out of available
memory. Quite naturally, users would prefer to cross-compile on a different machine when developing for these platforms.

In other cases, building in a Docker container is not always the best solution for certain development workflows. For
example, when working with Swift AWS Lambda Runtime, some developers may find that installing Docker just for building a
project is a daunting step that shouldn’t be required.

The solution described below is general enough to scale for any build-time/run-time triple combination.

## Proposed solution

Since CC destination is a collection of binaries arranged in a certain directory hierarchy, it makes sense to distribute
it as an archive. We'd like to build on top of
[SE-0305](https://github.com/apple/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md) and
extend the `.artifactbundle` format to support this.

Additionally, we propose introducing a new `swift destination` CLI command for installation and removal of CC
destinations on the local filesystem.

We introduce a notion of a top-level toolchain, which is the toolchain that handles user’s `swift destination`
invocations. Parts of this top-level toolchain (linker, C/C++ compilers, and even the Swift compiler) can be overridden
with tools supplied in `.artifactbundle` s installed by `swift destination` invocations.

When the user runs `swift build` with the selected CC destination, the overriding tools from the corresponding bundle
are invoked by `swift build` instead of tools from the top-level toolchain.

## Detailed design

### CC Destination Artifact Bundles

As a quick reminder for a concept introduced in
[SE-0305](https://github.com/apple/swift-evolution/blob/main/proposals/0305-swiftpm-binary-target-improvements.md), an
**artifact bundle** is a directory that has the filename suffix `.artifactbundle` and has a predefined structure with
`.json` manifest files provided as metadata.

The proposed structure of artifact bundles containing CC destinations looks like:

```
<name>.artifactbundle
├ info.json
├ <destination artifact>
│ ├ <host variant>
│ │ ├ destination.json
│ │ └ <destination file tree>
│ └ <host variant>
│   ├ destination.json
│   └ <destination file tree>
├ <destination artifact>
│ └ <host variant>
│   ├ destination.json
│   └ <destination file tree>
├ <destination artifact>
┆ └┄
```

For example, a destination bundle allowing to cross-compile Swift 5.7 source code to recent versions of Ubuntu from
macOS would look like this:

```
swift-5.7_ubuntu.artifactbundle
├ info.json
├ ubuntu_jammy
│ ├ arm64-apple-darwin
│ │ ├ destination.json
│ │ ├ toolset.json
│ │ └ <destination file tree>
│ └ x86_64-apple-darwin
│   ├ destination.json
│   ├ toolset.json
│   └ <destination file tree>
├ ubuntu_focal
│ └ x86_64-apple-darwin
│   ├ destination.json
│   ├ toolset.json
│   └ <destination file tree>
├ ubuntu_bionic
┆ └┄
```

Here each artifact directory is dedicated to a specific CC destination, while binaries for a specific host platform are
placed in `arm64-apple-darwin` and `x86_64-apple-darwin` subdirectories.

`info.json` bundle manifests at the root of artifact bundles should specify `"type": "crossCompilationDestination"`
for corresponding artifacts. Artifact identifiers in this manifest uniquely identify a CC destination. The rest of the
properties of bundle manifests introduced in SE-0305 are preserved.

### `destination.json` files

Note the presence of `destination.json` files in each `<host variant>` subdirectory. These files should contain a JSON
dictionary with an evolved version of the schema of [existing `destination.json` files that SwiftPM already
supports](https://github.com/apple/swift-package-manager/pull/1098) and `destination.json` files presented in the pitch
version of this proposal, hence `"schemaVersion": "3.0"`:

```json
{
  "schemaVersion": "3.0",
  "buildTimeTriples": ["<an array of supported build-time platform triples>"],
  "runTimeTriples": ["<an array of supported run-time platform triples>"],
  "swiftResourcesPaths": ["<array with relative paths containing Swift resources in the destination tree>"],
  "includeSearchPaths": ["<array with relative paths containing headers in the destination tree>"],
  "librarySearchPaths": ["<array with relative paths containing libraries in the destination tree>"],
}
```

Thanks to the availability of Universal Binaries on macOS and multiarch layouts on Linux, `buildTimeTriples` and
`runTimeTriples` properties use plural in their naming and their values are arrays.

We propose that all relative paths in `destination.json` files should be validated not to "escape" the destination
bundle for security reasons. That is, `../` components, if present in paths, will not be allowed to reference files and
directories outside of a corresponding destination bundle. Symlinks will also be validated to prevent them from escaping
out of the bundle.

Since not all platforms can support self-contained destination bundles, users will be able to provide their own
additional paths on the filesystem outside of bundles after a destination is installed. The exact options for specifying
paths are proposed in a subsequent section for a newly introduced `swift destination configure` command.

### `toolset.json` files

We find that properties dedicated to tools configuration are useful outside of the cross-compilation context. Due to
that, a separate `toolset.json` file is introduced:

```json5
{
  "schemaVersion": "1.0",
  "toolsetRootPath": "optional path to a root directory containing toolchain executables",
  // If `toolsetRootPath` is specified, all relative paths below will be resolved relative to `toolsetRootPath`.
  "swiftCompiler": {
    "path": "<optional path to the Swift compiler>",
    "extraFlags": ["<optional array of additional flags passed to the Swift compiler>"]
  },
  "cCompiler": {
    "path": "<optional path to the C compiler>",
    "extraFlags": ["<optional array of additional flags passed to the C compiler>"]
  },
  "cxxCompiler": {
    "path": "<optional path to the C++ compiler>",
    "extraFlags": ["<optional array of additional flags passed to the C++ compiler>"]
  },
  "linker": {
    "path": "<optional path to the linker>",
    "extraFlags": ["<optional array of additional flags passed to the linker>"]
  }
}
```

Users familiar with CMake can draw an analogy between toolset files and CMake toolchain files. Toolset files are
designed to supplant previous ad-hoc ways to specify paths and flags in SwiftPM, such as `SWIFT_EXEC` and `CC`
environment variables, which were applied in use cases unrelated to cross-compilation. We propose that
users should be able to pass `--toolset <path_to_toolset.json>` option to both `swift build` and `swift test`.

We'd like to allow using multiple toolset files at once. With this users can "assemble" toolchains on the fly out of
tools that in certain scenarios may even come from different vendors.

All of the properties related to names of the tools are optional, which also allows stacking multiple toolset files.
For example, consider `toolset1.json`:

```json
{
  "schemaVersion": "1.0",
  "swiftCompiler": {
    "path": "/usr/bin/swiftc",
    "extraFlags": ["-Xfrontend", "-enable-cxx-interop"]
  },
  "cCompiler": {
    "path": "/usr/bin/clang",
    "extraFlags": ["-pedantic"]
  }
}
```

and `toolset2.json`:

```json
{
  "schemaVersion": "1.0",
  "swiftCompiler": {
    "path": "/custom/swiftc",
    "extraFlags": []
  }
}
```


With multiple `--toolset` options, passing both of those files will merge them into a single configuration. Tools passed
in subsequent `--toolset` options will shadow tools from previous options with the same names. That is, 
`swift build --toolset toolset1.json --toolset toolset2.json` will build with `/custom/swiftc` and no extra flags, as
specified in `toolset2.json`, but `/usr/bin/clang -pedantic` from `toolset1.json` will still be used.

Tools not specified in any of the supplied toolset files will be looked up in existing implied search paths that are
used without any presence of toolset files, even when `toolsetRootPath` is present in any toolset file. We'd like
toolsets to be explicit enough in this regard: if any tool would like to participate in toolsets path lookups, it must
have its path provided, either relative or absolute, in any of the provided toolset files.

When cross-compiling, paths in `toolset.json` files supplied in destination artifact bundles should be self-contained
with same rules applied as in `destination.json`: no absolute paths and no escaping symlinks are allowed. Users are
still able to provide their own `toolset.json` files outside of artifact bundles to specify additional developer tools
for which no relative path can be provided within the bundle. 

### Destination Bundle Installation and Configuration

To manage CC destinations, we'd like to introduce a new `swift destination` command with three subcommands:

- `swift destination install <bundle URL or local filesystem path>`, which downloads a given bundle if needed and
  installs it in a location discoverable by SwiftPM. For destinations installed from remote URLs an additional
  `--checksum` option is required, through which users of destinations can specify a checksum provided by publishers of
  destinations. The latter can produce a checksum by running `swift package compute-checksum` command (introduced in
  SE-0272) with the destination artifact bundle archive as an argument.
- `swift destination list`, which prints a list of already installed CC destinations with their identifiers.
- `swift destination configure <identifier>`, which allows users to provide additional search paths and toolsets to be
used subsequently when building with a given destination. Specifically, multiple `--swift-resources-path`,
`--include-search-path`, `--library-search-path`, and `--toolset` options with corresponding paths can be provided,
which then will be stored as configuration for this destination. 
`swift destination configure <identifier> --show-configuration` will print currently set paths, while
`swift destination configure <identifier> --reset` will reset all of those at once.
- `swift destination delete <identifier>` will delete a given destination from the filesystem.

### Using a CC Destination

After a destination is installed, users can refer to it via its identifier passed to the `--destination` option, e.g.

```
swift build --destination ubuntu-jammy
```

We'd also like to make `--destination` flexible enough to recognize run-time triples when there's only a single CC
destination installed for such triple:

```
swift build --destination x86_64-unknown-linux-gnu
```

When multiple destinations support the same triple, an error message will be printed listing these destinations and
asking the user to select a single one via its identifier instead.

### CC Destination Bundle Generation

CC destinations can be generated quite differently, depending on build-time and run-time triple combinations and user's
needs. We intentionally don't specify how destination artifact bundles should be generated.

Authors of this document intend to publish source code for a macOS → Linux CC destination generator, which community is
welcome to fork and reuse for their specific needs. This generator will use Docker for setting up the build environment
locally before copying it to the destination tree. Relying on Docker in this generator makes it easier to reuse and
customize existing build environments. Important to clarify, that Docker is only used for bundle generation, and users
of CC destinations do not need to have Docker installed on their machine to utilize it.

As an example, destination publishers looking to add a library to an Ubuntu 22.04 destination environment would modify a
`Dockerfile` similar to this one in CC destination generator source code:

```dockerfile
FROM swift:5.7-jammy

apt-get install -y \
  # PostgreSQL library provided as an example.
  libpq-dev
  # Add more libraries as arguments to `apt-get install`.
```

Then to generate a new CC destinations, a generator executable delegates to Docker for downloading and installing
required tools and libraries, including the newly added ones. After a Docker image with destination environment is
ready, the generator copies files from the image to a corresponding `.artifactbundle` destination tree.

## Security

The proposed `--checksum` flag provides basic means of verifying destination bundle's validity. As a future direction,
we'd like to consider sandboxing and codesigning toolchains running on macOS.

## Impact on existing packages

This is an additive change with no impact on existing packages.

## Prior Art

### Rust

In the Rust ecosystem, its toolchain and standard library built for a run-time triple are managed by [the `rustup`
tool](https://github.com/rust-lang/rustup). For example, artifacts required for cross-compilation to
`aarch64-linux-unknown-gnu` are installed with
[`rustup target add aarch64-linux-unknown-gnu`](https://rust-lang.github.io/rustup/cross-compilation.html). Then
building for this target with Rust’s package manager looks like `cargo build --target=aarch64-linux-unknown-gnu` .

Mainstream Rust tools don’t provide an easy way to create your own destinations/targets. You’re only limited to the list
of targets provided by Rust maintainers. This likely isn’t a big problem per se for Rust users, as Rust doesn’t provide
C/C++ interop on the same level as Swift. It means that Rust packages much more rarely than Swift expect certain
system-provided packages to be available in the same way that SwiftPM allows with `systemLibrary`.

Currently, Rust doesn’t supply all of the required tools when running `rustup target add`. It’s left to a user to
specify paths to a linker that’s suitable for their build-time/run-time triple combination manually in a config file. We
feel that this should be unnecessary, which is why destination bundles proposed for Swift can provide their own tools
via toolset configuration files.

### Go

Go’s standard library is famously self-contained and has no dependencies on C or C++ standard libraries. Because of this
there’s no need to install artifacts. Cross-compiling in Go works out of the box by passing `GOARCH` and `GOOS`
environment variables with chosen values, an example of this is `GOARCH=arm64 GOOS=linux go build` invocation.

This would be a great experience for Swift, but it isn’t easily achievable as long as Swift standard library depends on
C and C++ standard libraries. Any code interoperating with C and/or C++ would have to link with those libraries as well.
When compared to Go, our proposed solution allows both dynamic and, at least on Linux when Musl is supported, full
static linking. We’d like Swift to allow as much customization as needed for users to prepare their own destination
bundles.

## Alternatives Considered

### Extensions Other Than `.artifactbundle`

Some members of the community suggested that destination bundles should use a more specific extension. Since we're
relying on the existing `.artifactbundle` format and extension, which is already used for binary targets, we think a
specialized extension only for destinations would introduce an inconsistency. On the other hand, we think that specific
extensions could make sense with a change applied at once. For example, we could consider `.binarytarget` and
`.ccdestination` extensions for respective artifact types. But that would require a migration strategy for existing
`.artifactbundle`s containing binary targets.

### Building Applications in Docker Containers

Instead of coming up with a specialized bundle format for destinations, users of Swift on macOS building for Linux could
continue to use Docker. But, as discussed in the [Motivation](#motivation) section, building applications in Docker
doesn’t cover all of the possible use cases and complicates onboarding for new users. It also only supports Linux, while
we’re looking for a solution that can be generalized for all possible platforms.

### Alternative Bundle Formats

One alternative is to allow only a single build-time/run-time combination per bundle, but this may complicate
distribution of destinations bundles in some scenarios. The existing `.artifactbundle` format is flexible enough to
support bundles with a single or multiple combinations.

Different formats of destination bundles can be considered, but we don't think those would be significantly different
from the proposed one. If they were different, this would complicate bundle distribution scenarios for users who want to
publish their own artifact bundles with executables, as defined in SE-0305.

## Future Directions

### Identifying Platforms with Dictionaries of Properties

Platform triples are not specific enough in certain cases. For example, `aarch64-unknown-linux` host triple can’t
prevent a user from installing a CC destination bundle on an unsupported Linux distribution. In the future we could
deprecate `hostTriple` and `destinationTriple` JSON properties in favor of dictionaries with keys and values that
describe aspects of platforms that are important for destinations. Such dictionaries could look like this:

```json5
"destination": {
  "kernel": "Linux",
  "libcFlavor": "Glibc",
  "libcMinVersion": "2.36",
  "cpuArchitecture": "aarch64"
  // more platform capabilities defined here...
}
```

A toolchain providing this information could allow users to refer to these properties in their code for conditional
compilation and potentially even runtime checks.

### SwiftPM Plugins for Remote Running, Testing, Deployment, and Debugging

After an application is built with a CC destination, there are other development workflow steps to be improved. We could
introduce new types of plugins invoked by `swift run` and `swift test` for purposes of remote running, debugging, and
testing. For Linux run-time triples, these plugins could delegate to Docker for running produced executables.

### `swift destination select` subcommand

While `swift destination select` subcommand or a similar one make sense for selecting a CC destination instead of
passing `--destination` to `swift build` every time, users will expect `swift run` and `swift test` to also work for any
destination previously passed to `swift destination select`. That’s out of scope for this proposal on its own and
depends on making plugins (from the previous subsection) or some other remote running and testing implementation to
fully work.

### SwiftPM and SourceKit-LSP improvements

It is a known issue that SwiftPM can’t run multiple concurrent builds for different run-time triples. This may cause
issues when SourceKit-LSP is building a project for indexing purposes (for a host platform by default), while a user may
be trying to build for a run-time for example for testing. One of these build processes will fail due to the
process locking the build database. A potential solution would be to maintain separate build databases per platform.

Another issue related to SourceKit-LSP is that [it always build and indexes source code for the host
platform](https://github.com/apple/sourcekit-lsp/issues/601). Ideally, we want it to maintain indices for multiple
platforms at the same time. Users should be able to select run-time triples and corresponding indices to enable
semantic syntax highlighting, auto-complete, and other features for areas of code that are conditionally compiled with
`#if` directives.

### Source-Based CC Destinations

One interesting solution is distributing source code of a minimal base destination, as explored by [Zig programming
language](https://andrewkelley.me/post/zig-cc-powerful-drop-in-replacement-gcc-clang.html). In this scenario, a
cross-compilation destination binaries are produced on the fly when needed. We don't consider this option to be mutually
exclusive with solutions proposed in this document, and so it could be explored in the future for Swift as well.
However, this requires reducing the number of dependencies that Swift runtime and core libraries have.
