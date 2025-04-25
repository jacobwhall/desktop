# Fedora _(Jacob's Version)_

This is a custom [Fedora Silverblue](https://fedoraproject.org/atomic-desktops/silverblue/) image, built with things I like.
This repository was forked from the [BlueBuild Template project](https://github.com/blue-build/template).

A fresh image is built every day using GitHub Actions. Since this image is built on top [Fedora Atomic Desktops](https://fedoraproject.org/atomic-desktops/), it is easy to roll back when something breaks.

## Installation

1. [Install Fedora Silverblue](https://fedoraproject.org/atomic-desktops/silverblue/download)
2. _(optional)_ Pin the Silverblue installation incase you want to roll back to it
   ```
   sudo ostree admin pin 0
   ```
3. `rpm-ostree rebase ostree-unverified-registry:ghcr.io/jacobwhall/desktop:latest`
4. Reboot
5. Switch to signed image
   ```
   rpm-ostree rebase ostree-image-signed:docker://ghcr.io/jacobwhall/desktop:latest
   ```
   I have no idea why `docker://` is necessary here, but returns an error if included in the previous command.
6. Reboot

## Upgrading

1. `rpm-ostree upgrade`
2. Reboot to apply upgrade
