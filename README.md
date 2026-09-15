# pmos-packages

> **Unofficial.** Not affiliated with or endorsed by postmarketOS, Google, or
> Qualcomm. Do not report problems with this port to postmarketOS; open an
> issue here.
>
> **Experimental.** Flashing can brick the device or erase data. No warranty,
> see LICENSE.
>
> **AI-assisted.** See [AI.md](AI.md).

Prebuilt postmarketOS packages for the porthole ports, published as a signed
apk repository that pmbootstrap and apk use directly. CI builds them from the
`taimen-bringup` branch of
[porthole-dev/pmaports](https://github.com/porthole-dev/pmaports); how that
works is in [ARCHITECTURE.md](ARCHITECTURE.md).

## Repositories

Each apk repository is one GitHub release, and the release tag is the path apk
requests, so the download URLs are ordinary apk repository URLs:

| repository | release tag | contains |
| --- | --- | --- |
| postmarketOS edge | `main/aarch64` | forks of Alpine and pmaports packages |
| postmarketOS edge, systemd | `systemd/main/aarch64` | forks of `extra-repos/systemd` packages, such as phosh |

`main` is the pmaports branch of the edge channel (`branch_pmaports` in
[channels.cfg](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/blob/main/channels.cfg)).
Firmware packages are never published here; pmbootstrap builds them locally.

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

## Contributing

Packages are changed in
[porthole-dev/pmaports](https://github.com/porthole-dev/pmaports), not here.
See [CONTRIBUTING.md](CONTRIBUTING.md).
