# pmos-packages

[Website](https://porthole-dev.github.io/porthole/) · [Downloads](https://porthole-dev.github.io/porthole/images/) · [Device support](https://porthole-dev.github.io/porthole/devices/)

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
Taimen firmware is published under the recorded
[Google and Qualcomm grant](https://github.com/porthole-dev/firmware-google-taimen/blob/main/REDISTRIBUTION.md).
The exception covers the base package and optional fingerprint subpackage.
Other device firmware remains excluded.
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
mkdir -p "$work/config_apk_keys"
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

## Verify where a package came from

Every file the build workflow uploads
here (packages and indexes) carries a signed build provenance attestation
naming the workflow run and commit that produced it:

```sh
gh attestation verify mesa-26.2.2-r51.apk -R porthole-dev/pmaports
```

## Contributing

Packages are changed in
[porthole-dev/pmaports](https://github.com/porthole-dev/pmaports), not here.
See [CONTRIBUTING.md](CONTRIBUTING.md).
