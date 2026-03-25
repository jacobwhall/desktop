# Converting a BlueBuild recipe from Silverblue to Sway

**The official Universal Blue Sway images were removed in October 2025, making Wayblue (`ghcr.io/wayblueorg/sway`) the primary actively maintained option for Sway in the ublue ecosystem.** Switching a BlueBuild recipe from `silverblue-main` to a Sway base requires removing the `gnome-extensions` module, keeping `gtk-layer-shell` (it becomes *more* essential), configuring `xdg-desktop-portal-wlr` properly, and adding Sway-specific tooling like waybar, fuzzel, and mako. GNOME flatpaks work on Sway but need portal configuration. Docker, Tailscale, and Mullvad all function but demand more manual setup than on Bluefin/Silverblue.

## Available Sway base images as of March 2026

Universal Blue officially deprecated its Sway Atomic images in June 2025, with full removal in October 2025 (issue #927 in ublue-os/main). Only three first-party base images remain: `base-main` (no desktop), `silverblue-main` (GNOME), and `kinoite-main` (KDE). The formerly available `ghcr.io/ublue-os/sway-atomic-main` and its predecessor `ghcr.io/ublue-os/sericea-main` no longer exist.

**Wayblue is now the recommended path.** This community project at `github.com/wayblueorg/wayblue` (293 stars, 26 forks, latest release v0.4.3 from November 2025) builds on `ublue-os/base-main` using BlueBuild and provides these Sway images:

| Image | ghcr.io path | GPU support |
|-------|-------------|------------|
| sway | `ghcr.io/wayblueorg/sway:latest` | Intel/AMD only |
| sway-nvidia | `ghcr.io/wayblueorg/sway-nvidia:latest` | Proprietary NVIDIA |
| sway-nvidia-open | `ghcr.io/wayblueorg/sway-nvidia-open:latest` | Open NVIDIA (Turing+) |
| sway-gdm | `ghcr.io/wayblueorg/sway-gdm:latest` | Intel/AMD (GDM variant) |
| sway-nvidia-gdm | `ghcr.io/wayblueorg/sway-nvidia-gdm:latest` | Proprietary NVIDIA (GDM) |
| sway-nvidia-open-gdm | `ghcr.io/wayblueorg/sway-nvidia-open-gdm:latest` | Open NVIDIA (GDM) |

The **only supported tag is `latest`**, which tracks the current Fedora release (Fedora 42). SDDM variants are recommended over GDM variants (the GDM images pull in GNOME Shell). Wayblue self-describes as **beta quality** but has active CI, cosign-based image signing, and Trivy security scanning.

A third option is the **DIY approach**: use `ghcr.io/ublue-os/base-main` as your base image and install `@sway-desktop-environment` via the `dnf` module. This gives maximum control but requires configuring everything yourself. Upstream Fedora's `quay.io/fedora/fedora-sway-atomic` is also available but lacks Universal Blue's RPMFusion codecs, NVIDIA drivers, and auto-update infrastructure.

## Module and package changes required for the conversion

### The `gnome-extensions` module must be removed

The BlueBuild `gnome-extensions` module queries the running `gnome-shell --version`, fetches extensions from `extensions.gnome.org`, and installs them into `/usr/share/gnome-shell/extensions`. **It will fail outright on a Sway image** because `gnome-shell` is not present. This module has no purpose in a Sway environment — Sway has no concept of shell extensions. Similarly, the `gschema-overrides` module is GNOME-oriented and should be removed unless you specifically need GSettings overrides for GTK applications.

### Keep `gtk-layer-shell` — it's more useful on Sway than GNOME

This is a counterintuitive but critical point. `gtk-layer-shell` implements the `zwlr_layer_shell_v1` Wayland protocol, which is the protocol wlroots-based compositors (including Sway) use for desktop shell components. **Waybar, wofi, SwayNotificationCenter, and nwg-shell all depend on `gtk-layer-shell`** to position panels, launchers, and notification popups. Ironically, GNOME does *not* support wlr-layer-shell, so this library is essentially useless on Silverblue but becomes a **core infrastructure dependency on Sway**. Keep it. Consider also adding `gtk4-layer-shell` for GTK4-based tools.

### GNOME flatpaks work on Sway with portal configuration

**`org.gnome.Loupe`**, **`org.gnome.Papers`**, and **`org.gnome.font-viewer`** all function on Sway because Flatpak bundles the `org.gnome.Platform` runtime inside the sandbox — they don't need GNOME Shell services on the host. These apps use standard Wayland window protocols and support client-side decorations, which Sway renders correctly.

However, proper **xdg-desktop-portal** configuration is essential. Without it, file chooser dialogs break, dark mode fails to apply, and Electron apps may hang on startup. The correct setup for Sway requires:

- Install `xdg-desktop-portal`, `xdg-desktop-portal-wlr`, and `xdg-desktop-portal-gtk` (use `-gtk`, **not** `-gnome` — the GNOME backend causes hangs outside GNOME sessions)
- Create `/usr/share/xdg-desktop-portal/sway-portals.conf` with:
  ```ini
  [preferred]
  default=gtk
  org.freedesktop.impl.portal.Screenshot=wlr
  org.freedesktop.impl.portal.ScreenCast=wlr
  ```
- Ensure `XDG_CURRENT_DESKTOP=sway` is exported and propagated to D-Bus/systemd

If you want lighter alternatives popular in the tiling WM community: **`imv`** for images (keyboard-driven, included in Fedora Sway Atomic), **`zathura`** for PDFs (minimal, vim-keybindings), and `fc-list` or `fontmatrix` for fonts.

### Sway-specific packages to add

When building on Wayblue, most core Sway packages ship by default (sway, waybar, grim, slurp, etc.). When building on `base-main`, add these via the `dnf` module:

**Essential core** — `sway`, `swaybg`, `waybar`, `swaylock`, `swayidle`, `foot` (terminal), `grim`, `slurp`, `wl-clipboard`. **Application launcher** — choose one: `fuzzel` (lightweight, fast, Wayland-native), `rofi-wayland` (full-featured, included in Fedora Sway Atomic), or `wofi` (GTK-based, uses gtk-layer-shell). **Notifications** — choose one: `mako` (minimal, default in Fedora Sway Atomic) or `SwayNotificationCenter` (feature-rich control center, available via Copr). **Display management** — `wdisplays` (GUI) or `kanshi` (auto-profile switching). **Networking GUI** — `network-manager-applet` (run as `nm-applet --indicator` for waybar tray). **Polkit agent** — `polkit-gnome` (must be manually started in sway config; GNOME does this automatically). **Clipboard history** — `cliphist`. **Brightness** — `brightnessctl` or `light`. **Media control** — `playerctl`.

## Wayblue is maintained and is the best recipe example

Wayblue (last release November 2025, **795 commits**, 10 contributors) is actively maintained with passing CI builds. It is itself a BlueBuild project, making it both the primary Sway image provider and the best recipe example. The project generates **30+ image variants** through a matrix strategy combining 5 compositors × 3 GPU options × 2 display managers.

Wayblue **strongly recommends against forking**. Instead, create your own BlueBuild repo from the `blue-build/template` and set the base image to Wayblue:

```yaml
# recipe.yml
name: my-custom-sway
description: My customized Sway desktop
base-image: ghcr.io/wayblueorg/sway
image-tag: latest

modules:
  - type: dnf
    install:
      - fuzzel
      - cliphist
      - brightnessctl
      - playerctl

  - type: files
    files:
      - source: sway
        destination: /etc/sway/config.d/
      - source: waybar
        destination: /etc/xdg/waybar/

  - type: default-flatpaks
    configurations:
      - scope: system
        repo:
          title: Flathub
        install:
          - org.mozilla.firefox
          - org.gnome.Loupe
          - org.gnome.Papers
          - com.github.tchx84.Flatseal

  - type: signing
```

Other community examples include `benhoman/bluebuild` (personal Sway image, 135 commits) and `secureblue` (security-hardened images with Sway variants that disable XWayland by default).

## The `default-flatpaks` module works identically on Sway

The `default-flatpaks` module is **completely desktop-environment agnostic**. It installs Flatpaks from configured remotes (typically Flathub) on every boot via systemd services. It has no dependency on any particular desktop environment — the logic is purely about Flatpak IDs, scope (system/user), and repository configuration.

Note that `default-flatpaks` was **rewritten with breaking changes** (announced July 2025). The current v2 uses a `configurations:` list format rather than the old `user:`/`system:` top-level format. Using `type: default-flatpaks` gets v2; to pin the old version use `type: default-flatpaks@v1`. The v2 format works identically whether your base is GNOME, KDE, Sway, or bare `base-main`.

## Docker, Tailscale, and Mullvad all work but need more setup

The biggest difference from Silverblue is that **no Sway-based ublue image has a "DX" developer variant** equivalent to Bluefin-DX or Aurora-DX, which ship Docker CE pre-configured. On a Sway image, Docker must be manually layered via `rpm-ostree install docker-ce docker-ce-cli containerd.io docker-compose-plugin` and enabled with `systemctl enable docker.socket`, or you can use Podman (included by default on all Fedora Atomic images).

**Tailscale** works well on Sway. The official `tailscale systray` command (available since Tailscale v1.96+) explicitly supports waybar and other Wayland bars implementing the StatusNotifierItem specification. Configure auto-start with `tailscale configure systray --enable-startup=systemd`. Community waybar modules for Tailscale status also exist. For Fedora Atomic, Tailscale should ideally be baked into the image via `rpm-ostree install tailscale` in your BlueBuild recipe, with `systemctl enable tailscaled` in the `systemd` module. Tailscale on Sway also requires `sudo firewall-cmd --add-interface=tailscale0 --zone=trusted --permanent` for proper firewall configuration.

**Mullvad VPN 2026.1** runs natively on Wayland (no more `--ozone-platform=wayland` workaround needed). The Electron-based GUI works on Sway, and its tray icon appears in waybar's `tray` module (requires `libappindicator-gtk3` installed). Two known issues: the tray icon can occasionally become **unresponsive after extended use** (an Electron/tray interaction bug, not Sway-specific), and launch failures with `org.freedesktop.portal.FileChooser` errors if xdg-desktop-portal is misconfigured. The fix is the same `sway-portals.conf` described above. Mullvad is RPM-only (no Flatpak) and requires `rpm-ostree install mullvad-vpn` plus `systemctl enable mullvad-daemon`.

**Running Tailscale and Mullvad together** presents a conflict regardless of desktop environment: Mullvad captures all outgoing traffic, breaking Tailscale connectivity. The cleanest solution is **Tailscale's built-in Mullvad integration** (Mullvad servers appear as Tailscale exit nodes at $5/month for 5 devices), or using Mullvad's split tunneling to exclude Tailscale's CGNAT range.

## Conclusion

The migration from Silverblue to Sway in the ublue ecosystem is well-supported but requires deliberate choices. Use **`ghcr.io/wayblueorg/sway:latest`** as your base image — it's the only actively maintained Sway option with Universal Blue's enhancements (RPMFusion, codecs, auto-updates). Strip out `gnome-extensions` and `gschema-overrides` modules. Keep `gtk-layer-shell` and `default-flatpaks` as-is. The single most important configuration step is getting **xdg-desktop-portal** right with `sway-portals.conf` and `xdg-desktop-portal-gtk` (not `-gnome`) — this one file determines whether Flatpak apps, Electron apps like Mullvad, and file dialogs work correctly. For Docker, Tailscale, and Mullvad, bake all three into your BlueBuild recipe via `dnf`/`rpm-ostree` modules and enable their systemd services, since there's no DX variant to do this for you. The overall maturity gap between Sway and GNOME on ublue is real but manageable — you trade Bluefin's polish for tiling WM efficiency, with Wayblue providing a solid foundation.
