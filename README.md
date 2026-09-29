# pmos-packages

> **Unofficial.** Not affiliated with or endorsed by Nura, Google, or
> Qualcomm. Do not report problems with this port to Nura; open an
> issue here.
>
> **Experimental.** Flashing can brick the device or erase data. No warranty,
> see LICENSE.
>
> **AI-assisted.** See [AI.md](AI.md).

Prebuilt Nura packages for the porthole Pixel 2 XL port, published as a signed
apk repository that pmbootstrap and apk use directly. CI builds them from the
`taimen-bringup` branch of
[porthole-dev/pmaports](https://github.com/porthole-dev/pmaports); how that
works is in [ARCHITECTURE.md](ARCHITECTURE.md).

## Find a build

This repository contains packages. For full images, hardware test dates, and
known limitations, see [Porthole](https://github.com/porthole-dev/porthole).
A release asset here does not imply support for every device.

## Repositories

Each apk repository is one GitHub release, and the release tag is the path apk
requests, so the download URLs are ordinary apk repository URLs:

| repository | release tag | contains |
| --- | --- | --- |
| Nura edge (legacy APK path) | `main/aarch64` | forks of Alpine and pmaports packages |
| Nura edge, systemd (legacy APK path) | `systemd/main/aarch64` | forks of `extra-repos/systemd` packages, such as phosh |
| (both, for x86_64 hosts) | `main/x86_64`, `systemd/main/x86_64` | an empty signed index |

`main` is the pmaports branch of the edge channel (`branch_pmaports` in
[channels.cfg](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/blob/main/channels.cfg)).
Firmware remains excluded by default. After the maintainer records the full
Google and Qualcomm Taimen grant, setting `FIRMWARE_GRANT_TAIMEN=approved` in
the pmaports repository variables permits only `firmware-google-taimen` and
its optional fingerprint subpackage. Other firmware remains excluded.
Neither are host tools such as `crossdirect`, which run in pmbootstrap's native
chroot: pmbootstrap on an x86_64 host also reads the mirror's x86_64 index, so
the empty one is there to answer instead of a 404, and pmbootstrap builds those
tools itself.

The GitHub repository and APK release paths keep their existing `pmos-packages` and
`main/<arch>` identifiers for compatibility; the device catalogue uses Nura as
the display name.

The repository index is signed with
[`keys/porthole-dev-packages-20260915.rsa.pub`](keys/porthole-dev-packages-20260915.rsa.pub).
That is the only key you need: apk trusts each package through its checksum in
the signed index. `keys/pmos@local-6a93544e.rsa.pub` signed packages uploaded
by hand before CI existed; installing it is harmless but not required.

## Use it with pmbootstrap

Install the key, then add the repositories in front of the official ones:

```sh
work=$(pmbootstrap config work)
curl -fLo "$work/config_apk_keys/porthole-dev-packages-20260915.rsa.pub" \
    https://raw.githubusercontent.com/porthole-dev/pmos-packages/main/keys/porthole-dev-packages-20260915.rsa.pub
pmbootstrap config mirrors.pmaports_custom https://github.com/porthole-dev/pmos-packages/releases/download
pmbootstrap config mirrors.systemd_custom https://github.com/porthole-dev/pmos-packages/releases/download/systemd
pmbootstrap config build_pkgs_on_install False
```

pmbootstrap appends the pmaports branch and apk appends the architecture, so
this reads `.../releases/download/main/aarch64/APKINDEX.tar.gz`. pmbootstrap
does not build an aport whose exact version is already in a binary
repository, so `pmbootstrap install` takes every package published here and
builds only what is missing (firmware, or a fork newer than the last publish).
The key and the repositories are installed into the image, so the device keeps
updating from them.

To undo: `pmbootstrap config mirrors.pmaports_custom none` and
`pmbootstrap config mirrors.systemd_custom none`.

## Use it on a device

```sh
doas wget -O /etc/apk/keys/porthole-dev-packages-20260915.rsa.pub \
    https://raw.githubusercontent.com/porthole-dev/pmos-packages/main/keys/porthole-dev-packages-20260915.rsa.pub
echo https://github.com/porthole-dev/pmos-packages/releases/download/main \
    | doas tee -a /etc/apk/repositories
# systemd images only:
echo https://github.com/porthole-dev/pmos-packages/releases/download/systemd/main \
    | doas tee -a /etc/apk/repositories
doas apk update
doas apk upgrade
```

## While this repository is private

Release downloads need authentication, which apk and pmbootstrap cannot send.
Download a release with the GitHub CLI and use the directory as a local
repository instead:

```sh
gh release download main/aarch64 -R porthole-dev/pmos-packages -D repo/main/aarch64
```

## Verify where a package came from

Once the pmaports repository is public, every file the build workflow uploads
here (packages and indexes) carries a signed build provenance attestation
naming the workflow run and commit that produced it:

```sh
gh attestation verify mesa-26.2.2-r51.apk -R porthole-dev/pmaports
```

## Contributing

Packages are changed in
[porthole-dev/pmaports](https://github.com/porthole-dev/pmaports), not here.
See [CONTRIBUTING.md](CONTRIBUTING.md).
