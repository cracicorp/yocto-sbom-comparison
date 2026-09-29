# Yocto SBOM comparison: Yocto's built-in SPDX SBOM vs a CRACI build-time SBOM

A side-by-side comparison of the SBOM that the Yocto Project writes for `core-image-sato` with the SBOM that
[CRACI](https://craci.com/features/yocto-builds) recorded while the same image was being built. Both SBOMs, the
component-by-component comparison and the build setup are in this repository.

## TL;DR

Built on a native ARM64 runner with 32 vCPUs, Yocto Project 6.0 (Wrynose), `poky`, `core-image-sato` for `qemuarm64`:

| | CRACI SBOM | Yocto built-in SBOM |
| --- | --- | --- |
| Components | **953** | 865 |
| Components the other SBOM does not have | **88** | 0 |
| Rust crate versions listed | 653 | 653 |
| Rust crates with a `pkg:cargo` package URL | **653** | 0 |
| Rust crates and host packages monitorable by package URL | **Yes** (`pkg:cargo`, `pkg:deb`) | No |
| Format | CycloneDX 1.7, one file | SPDX 3.0.1, 1,632 files |

- **CRACI's SBOM contains every component in Yocto's SBOM, plus 88 more (10% more).** The extra components come from
  outside BitBake's metadata: 23 Ubuntu host packages on the build machine (such as `cpio`), 10 GitHub sources
  including the GitHub Action `actions/upload-artifact`, and 55 other sources and tools fetched during the build.
- **Yocto's SBOM cannot really be used for supply chain vulnerability monitoring.** Vulnerability monitoring matches
  components to advisories by package URL (PURL). Yocto's SBOM gives its recipes Yocto-specific `pkg:yocto` URLs,
  lists its 653 Rust crate versions only as `.crate` files and download locations, and has no entries at all for the
  host packages. CRACI records the crates and host packages with the package URLs advisories are published against,
  such as `pkg:cargo/...` and `pkg:deb/ubuntu/...`, so they can be monitored for vulnerabilities.
- **The build with Yocto's SBOM generation and its source-detail options took 155 min 40 s. The build without them
  took 88 min 48 s, 43% less.** At CRACI's rate of €0.002 per vCPU-minute that is €9.96 against €5.68 per build.

## What was built

| | |
| --- | --- |
| Image | `core-image-sato` |
| Machine | `qemuarm64` |
| Yocto Project release | 6.0 (Wrynose), `poky` distro, set up with `bitbake-setup` (`poky-wrynose` configuration) |
| Runner | CRACI managed runner, native ARM64, 32 vCPUs and 96 GB RAM (`cracicorp/setup@v1`, `size: "32"`, `image: craci-slim-arm`) |
| CI | GitHub Actions workflow on a CRACI runner |
| Date | 25 September 2026 |

Sources were fetched through the official Yocto source mirror, with BitBake's fetcher limited to it:

```
INHERIT += "own-mirrors"
SOURCE_MIRROR_URL = "https://downloads.yoctoproject.org/mirror/sources/"
BB_ALLOWED_NETWORKS = "downloads.yoctoproject.org"
```

The build machine got the host packages from the Yocto Project quick start (`build-essential`, `chrpath`, `cpio`,
`diffstat`, `gawk`, `python3-git`, `python3-jinja2`, `python3-pexpect`, `python3-subunit`, `python3-websockets`,
`socat` and others) and the `en_US.UTF-8` locale.

### The Yocto built-in SBOM

The build that produced Yocto's SBOM added `create-spdx` and the options that make it as detailed as possible:

```
INHERIT += "create-spdx"
SPDX_PRETTY = "1"
SPDX_INCLUDE_SOURCES = "1"
SPDX_INCLUDE_COMPILED_SOURCES = "1"
SPDX_INCLUDE_KERNEL_CONFIG = "1"
SPDX_INCLUDE_PACKAGECONFIG = "1"
```

Yocto wrote the SBOM in SPDX 3.0.1 as 1,632 JSON documents: the image-level document links to per-recipe and
per-package documents in `tmp/deploy/spdx/`.

### The CRACI SBOM

CRACI records an SBOM of every build it runs, from the traffic coming into the build job: every package and source
the build downloads. No change to the Yocto configuration is needed. The CRACI SBOM in this repository was recorded
during the same build that produced Yocto's SBOM, so both describe exactly the same build.

## Results

### Components

After normalizing component names across all 1,632 files of Yocto's output (not only by package URL, since most
Yocto entries have none), the two SBOMs compare as follows:

| | Count |
| --- | --- |
| In both SBOMs | 865 |
| Only in the CRACI SBOM | 88 |
| Only in the Yocto built-in SBOM | 0 |

Yocto's SBOM is thorough about what BitBake builds: the image, every recipe including the build-time `-native` ones,
their sources, applied patches and the CVEs a recipe marks as fixed. What it misses is what BitBake's metadata does not
describe. The 88 components only CRACI listed:

<details>
<summary><strong>23 Ubuntu host packages</strong> on the build machine</summary>

`cpio`, `diffstat`, `libc-bin`, `libc-dev-bin`, `libc6`, `libc6-dev`, `libsigsegv2`, `locales`, `python3-extras`,
`python3-fixtures`, `python3-git`, `python3-gitdb`, `python3-jinja2`, `python3-markupsafe`, `python3-pbr`,
`python3-pexpect`, `python3-ptyprocess`, `python3-six`, `python3-smmap`, `python3-subunit`, `python3-testtools`,
`python3-websockets`, `socat`

For example, CRACI records `cpio` as
`pkg:deb/ubuntu/cpio@2.15+dfsg-1ubuntu2.1?arch=arm64&distro=noble`. BitBake runs with these tools, and Yocto's SBOM
does not list them.

</details>

<details>
<summary><strong>10 GitHub sources</strong>, including a GitHub Action</summary>

`actions/upload-artifact` (pinned to the commit the workflow ran), `btrfs-progs`, `calver`, `createrepo_c`, `neard`,
`patchelf`, `pkcs11-json`, `spirv-llvm-translator`, `unfs3`, `vim`

</details>

<details>
<summary><strong>55 other sources and tools</strong> fetched during the build</summary>

`aarch64-nativesdk-libc`, `alsa-topology-conf`, `colorama`, `coreutils`, `dbus-python`, `debianutils`, `dejagnu`,
`diffutils`, `dtc`, `editables`, `expect5.45.4`, `findutils`, `gcc`, `gnome-desktop-testing`, `grep`, `hatch_vcs`,
`hatchling`, `iniconfig`, `iproute2`, `iw`, `libmnl`, `libslirp`, `llvm-project`, `markupsafe`, `packaging`, `patch`,
`pathspec`, `pigz`, `pluggy`, `pretend`, `procps`, `pseudo`, `pseudo-prebuilt`, `psmisc`, `ptest-runner2`, `pycairo`,
`pygments`, `pygobject`, `pytest`, `python-unittest-automake-output`, `python_dbusmock`, `quilt`, `sdl2`, `sed`,
`setuptools_scm`, `tcl-core8.6.17-src`, `trove_classifiers`, `tzcode2026c`, `tzdata2026c`, `virglrenderer`,
`wireless-regdb`, `xml-namespacesupport`, `xml-sax`, `xml-sax-base`, `zip30`

</details>

The full lists, including the 865 shared components, are in
[`results/component-comparison.txt`](results/component-comparison.txt).

### Package URLs and vulnerability monitoring

Tools that monitor an SBOM for vulnerabilities, such as SBOM platforms and vulnerability databases, identify each
component by its package URL: `pkg:cargo/cairo-rs@0.21.2` for a Rust crate, `pkg:deb/ubuntu/cpio@...` for a Debian
package. Advisories are published against those identifiers. A component without one, or with an identifier no
advisory uses, cannot be matched, so a new vulnerability in it goes unreported.

Package URLs across all 1,632 files of Yocto's SBOM, counted once each:

| Package URL type | Count |
| --- | --- |
| `pkg:yocto` (Yocto-specific, one per recipe) | 347 |
| `pkg:github` | 32 |
| `pkg:pypi` | 10 |
| `pkg:cargo` | 5 |
| `pkg:cpan` | 2 |

More than half of the 865 components have no package URL, and most of the rest have a Yocto-specific one. So
Yocto's SBOM cannot really be used for supply chain vulnerability monitoring. Yocto's own `cve-check` class takes a
separate route for recipes, matching them against NVD by CPE product name, but that covers what BitBake builds, not
the Rust crates compiled into them or the tools on the build machine.

CRACI's SBOM records the 653 Rust crate versions as `pkg:cargo`, the 25 host packages as `pkg:deb` and 43 GitHub
sources as `pkg:github`. The upstream source tarballs and Git repositories BitBake fetched (345 components) are
recorded as `pkg:generic` with their download location, which identifies exactly what was fetched but is not
something advisories are published against either.

### Rust crates

Both SBOMs contain the same 653 versions of 559 Rust crates, for example `cairo-rs` and `zune-jpeg`, which `librsvg`
compiles in. The difference is how they are identified:

- **Yocto's SBOM** lists each crate as a `.crate` file with a download location such as
  `https://static.crates.io/crates/cairo-rs/0.21.2/download`, inside the source list of the recipe that uses it. None
  of the 653 has a `pkg:cargo` package URL. (Its only `pkg:cargo` URLs are 5 recipe-level ones, such as `cargo` and
  `librsvg` themselves.)
- **CRACI's SBOM** records every crate with a package URL, such as `pkg:cargo/cairo-rs@0.21.2`.

Vulnerability databases and SBOM tools match components to advisories by package URL. A crate that appears only as a
file name inside another recipe's source list gives them nothing to match, so a vulnerable crate compiled into the
image goes unreported. With CRACI's SBOM, each crate can be checked for vulnerabilities like any other package.

### Build time and cost

Both builds ran on the same kind of CRACI runner, native ARM64 with 32 vCPUs.

| Build | Time | Cost per build |
| --- | --- | --- |
| Without Yocto's SBOM generation and source-detail options, CRACI recording | 88 min 48 s | €5.68 |
| With Yocto's SBOM generation and the options above | 155 min 40 s | €9.96 |
| With Yocto's SBOM on a GitHub-hosted 32-core ARM64 runner (estimate) | about 5 h 11 min | about €13.45 ($15.60) |

- CRACI costs €0.002 per vCPU-minute, €0.064 a minute on 32 vCPUs, metered per second.
- The GitHub-hosted row is an estimate, not a measured run: it assumes the build takes about twice as long as on CRACI,
  billed as 312 whole minutes at GitHub's list price of $0.050 a minute for a 32-core ARM64 runner (September 2026),
  converted at €1 = $1.16. Included GitHub minutes do not cover larger runners.
- Yocto's SPDX output does carry recipe metadata CRACI does not record, such as applied patches and the CVEs a recipe
  marks as fixed. If your process depends on those, keep `create-spdx` on alongside CRACI.

## Files

| Path | What it is |
| --- | --- |
| [`sboms/craci-core-image-sato-arm64.cdx.json`](sboms/craci-core-image-sato-arm64.cdx.json) | The CRACI SBOM, CycloneDX 1.7 |
| [`sboms/yocto-builtin/core-image-sato-qemuarm64-rootfs.spdx.json`](sboms/yocto-builtin/core-image-sato-qemuarm64-rootfs.spdx.json) | Yocto's image-level SPDX 3.0.1 document for the root filesystem |
| [`results/component-comparison.txt`](results/component-comparison.txt) | Components only in each SBOM and in both, after normalization |
| [Release asset `yocto-builtin-sbom.zip`](https://github.com/cracicorp/yocto-sbom-comparison/releases) | All 1,632 SPDX documents of Yocto's SBOM (185 MB zipped, 4 GB unpacked) |

## Why this matters

What goes into a build decides what comes out of it, whether or not it ends up installed in the image. The
[xz utils backdoor (CVE-2024-3094)](https://www.cisa.gov/news-events/alerts/2024/03/29/reported-supply-chain-compromise-affecting-xz-utils-data-compression-library-cve-2024-3094)
went into the library while it was being built, and in March 2025
[CISA reported](https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction)
that every version of the GitHub Action `tj-actions/changed-files` was compromised for three days. An SBOM that
misses the build machine's packages and the workflow's Actions, or lists crates in a form vulnerability tools cannot
match, leaves gaps exactly where these attacks go in.

## About CRACI

[CRACI](https://craci.com) runs your GitHub Actions builds on its own runners and records what every build pulls in,
as an SBOM with a package URL for each component, signed provenance and an egress policy that blocks connections the
build does not need. Read more about [Yocto builds on CRACI](https://craci.com/features/yocto-builds), or
[book a demo](https://craci.com/book-demo) and bring one of your BitBake builds.
