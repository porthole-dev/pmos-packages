# pmos-packages

Prebuilt postmarketOS packages for the porthole ports, as a signed apk
repository in the layout pmbootstrap and apk expect.

## Layout

Packages live in GitHub Releases, one release per `<pmaports branch>/<arch>`
(git tags may contain a slash), so each release is exactly the directory apk
reads:

    releases/download/master/aarch64/APKINDEX.tar.gz
    releases/download/master/aarch64/<package>.apk

`APKINDEX.tar.gz` is signed with `keys/porthole-dev-packages-20260915.rsa.pub`.
The packages themselves carry the signature of the workspace that built them
(`keys/pmos@local-*.rsa.pub`); install both keys.

## Using it

While this repository is private, apk cannot download from it anonymously.
Sync a release into a directory with the GitHub CLI and use that:

    gh release download master/aarch64 -R porthole-dev/pmos-packages -D repo/master/aarch64

Once it is public, point pmbootstrap at it directly (docs/mirrors.md in
pmbootstrap), with both keys copied into `$WORKDIR/config_apk_keys` first:

    pmbootstrap config mirrors.pmaports_custom https://github.com/porthole-dev/pmos-packages/releases/download

pmbootstrap appends the channel's pmaports branch (`master`) and apk the arch,
which lands on the release above. The keys are installed into images, so a
phone installed that way keeps updating from it.
