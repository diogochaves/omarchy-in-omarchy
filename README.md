# Omarchy-in-Omarchy

A disposable [Omarchy](https://omarchy.org) machine running under libvirt (QEMU/KVM) on your Omarchy desktop, for testing plugins, themes and system changes without touching the machine you actually work on. Everything in it may break: the `fresh` snapshot stays clean and a new VM is one command.

![Omarchy running inside a QEMU window on an Omarchy desktop, with fastfetch reporting KVM/QEMU](docs/social-preview.png)

The interesting part is that the guest installs itself and boots straight into Hyprland with nothing to type. No installer wizard, no disk passphrase, no login prompt. That takes a few deliberate choices, and the ones that are not obvious are written down below.

```bash
omavm install                    # unattended install from the ISO, ~30 min, once
omavm stop && omavm save fresh   # keep the result as the clean baseline

omavm boot                       # ~18 seconds to a running desktop, no window
omavm ssh 'uname -a'             # run something as root in the guest
omavm shot screen.png            # screenshot of the guest desktop
omavm view                       # look at it yourself
omavm stop
```

## Requirements

Arch or Omarchy on the host, with libvirt and an Omarchy ISO from [iso.omarchy.org](https://iso.omarchy.org): `omavm install` takes the newest `omarchy-*.iso` in `~/Downloads`, or the one `OMAVM_ISO` points at. KVM, 8 cores, 8 GB RAM and 40 GB of disk go to the guest by default (`OMAVM_CPUS`, `OMAVM_MEM`).

Setting up libvirt takes one round of sudo:

```bash
sudo pacman -S --needed libvirt virt-install virt-viewer dnsmasq qemu-desktop edk2-ovmf mtools
sudo systemctl enable --now libvirtd.socket
# let your user manage qemu:///system without a password prompt
echo 'polkit.addRule(function(a, s) { if (a.id == "org.libvirt.unix.manage" && s.user == "'$USER'") return polkit.Result.YES; });' |
  sudo tee /etc/polkit-1/rules.d/50-libvirt-$USER.rules
sudo virsh net-autostart default && sudo virsh net-start default
sudo install -d -o $USER -g libvirt -m 2775 /var/lib/libvirt/images/omavm
```

With `ufw` active, also let the VMs through: `sudo ufw allow in on virbr0` and `sudo ufw route allow in on virbr0`.

## The unattended install

The shipped Omarchy ISO can install itself without a keyboard: it looks for a drive labelled `CIDATA` (the cloud-init NoCloud convention) holding archinstall's own answer files, and if it finds one it skips the configurator entirely. `omavm install` builds that drive, boots the ISO with it attached, waits for the installed system to come up and then provisions it.

Two files are mandatory on that drive, `user_configuration.json` and `user_credentials.json`; the optional ones cover the git identity, SSH keys and a Tailscale auth key. See the [omarchy-iso README](https://github.com/omacom-io/omarchy-iso#autoinstall) for the full list.

**Leaving out the `disk_encryption` block is what removes the passphrase prompt.** Encryption is configured entirely by that block inside `user_configuration.json`; the separate `user_encrypt_installation.txt` flag only has to agree with it. An encrypted install is never fully unattended, because the LUKS prompt still needs someone at the first boot.

That trade is the whole point here. A test VM that stops for a passphrase cannot be started from a script and cannot be left to install itself. What you give up is confidentiality of the guest disk, so nothing secret may live on it (see below).

**The answer file names no kernel**, so the ISO installs its own: stock `linux` up to 4.0.3, `linux-omarchy` from 4.0.4. Naming one the ISO does not carry fails late and confusingly. 4.0.4 installs everything, then stops at "Validating boot setup" with `linux (…) has no kernel headers`, because it insists on headers and ships them for its own kernel only. `OMAVM_KERNEL` still names one explicitly, for an ISO that carries it and its headers.

### Two things a plain install does not give you

Both only show up once the encryption is gone, and both are handled by `omavm provision`.

**No autologin.** The installer wires up SDDM autologin for *encrypted* installs, on the reasoning that you already proved who you are by typing the passphrase. Without encryption it configures none, so the machine boots to a login screen and you are back to typing. The fix is a normal SDDM drop-in:

```ini
# /etc/sddm.conf.d/99-autologin.conf
[Autologin]
User=your-user
Session=hyprland-uwsm.desktop
Relogin=true
```

**`NOPASSWD` is not enough for sudo.** The installer writes its own `user ALL=(ALL) ALL` rule, and against that `sudo -v` keeps demanding a password even when a later rule grants `NOPASSWD: ALL`. `omarchy-update` calls exactly that to keep its sudo session warm, so system updates stall on a prompt on a machine that is supposed to need no typing. Add the stronger form:

```
Defaults:your-user !authenticate
your-user ALL=(ALL:ALL) NOPASSWD: ALL
```

## Secrets stay on the host

The guest has no disk encryption, has passwordless sudo and hands out root over SSH. Treat its disk as readable by anyone who gets at it, and never put a key, token or password in it, not even temporarily: a qcow2 keeps deleted content around and the snapshot travels with it.

When something in the guest genuinely needs one of your keys, forward the agent instead of copying the key:

```bash
omavm agent git -C ~/dotfiles push
omavm agent                          # interactive shell with the agent
```

The guest can ask the agent to sign, never to hand a key over, and that access disappears when the command returns. With 1Password's agent you also get a per-signature approval prompt on the host. Verified: `ssh -T git@github.com` authenticates from inside the guest while the VM holds no private key at all, and `ssh-add -l` outside `omavm agent` immediately reports no agent.

The same principle covers tokens: fetch them on the host and pass them as an environment variable on a single command, rather than writing them to a file in the guest.

## libvirt, and no window

Every VM is a libvirt domain on `qemu:///system`: `omavm` itself, or `omavm-<name>` for a second one (`OMAVM_NAME=review omavm boot`). That gives three things for free:

- **No window.** A VM runs in the background, so booting one for a test or for an agent puts nothing on your desktop. `omavm view` opens its screen in `virt-viewer`, and closing the viewer leaves the VM running.
- **It shows up everywhere.** Anything that lists libvirt machines sees it, with start, stop and console: `virsh list`, virt-manager, or a VM panel in the bar.
- **Booting copies nothing.** A VM's disk is a thin qcow2 overlay on the snapshot it came from, so `omavm boot` starts in seconds and a fresh VM takes a few megabytes until it starts writing.

The guest gets its address from libvirt's `default` NAT network, and omavm finds it in the DHCP leases (`omavm ip`). It renders at 1920x1200 (`OMAVM_RES`) on plain virtio video without GPU acceleration, so libvirt can always photograph its screen.

`omavm create <name>` boots a named VM and prints `<domain><TAB><ip>`, for scripts that want a machine and an address and nothing else.

## Looking at a guest that has no SSH yet

While the guest is installing or stuck at a prompt there is nothing to SSH into. `omavm screendump` (`virsh screenshot`) and `omavm sendkey` (QEMU's monitor, through libvirt) need no cooperation from the guest and never touch the host desktop.

```bash
omavm screendump screen.png
omavm sendkey ret
omavm sendkey ctrl-alt-f2
```

`screendump` needs a software framebuffer, which GPU acceleration removes: with `virtio-vga-gl` QEMU answers `no surface`. That is why the guest runs without acceleration; seeing the screen from a script matters more here than smooth animations.

Once the guest has a session, `omavm shot` is the better screenshot: it runs `grim` inside the guest, so it comes out sharp and correctly scaled.

## Commands

Run `omavm --help` for the full reference.

| | |
|---|---|
| `install`, `provision`, `dotfiles` | Build the machine; provision and dotfiles can be re-run against a running guest |
| `boot`, `create`, `resume`, `stop`, `destroy`, `status` | Lifecycle. `boot` starts clean from a snapshot, `resume` continues the VM's own disk, `destroy` removes it |
| `save`, `list` | Snapshots, in `/var/lib/libvirt/images/omavm/snapshots`. Overwriting one needs `--force`, and is refused while a VM still builds on it |
| `ssh`, `user`, `agent`, `push`, `pull`, `ip` | Work in the guest |
| `view`, `shot`, `screendump`, `sendkey` | See and drive the screen |
| `hypr`, `qs`, `restart-shell`, `plugin` | Hyprland, Quickshell and plugin testing |

## Gotchas

- The disks live in libvirt's images directory, because the qemu that libvirt starts runs as its own user and has to read them. A disk in your home directory gives a permission error at boot.
- Never overwrite a snapshot that a VM's overlay builds on: the overlay only stores differences, so it would silently turn into garbage. `omavm save` refuses.
- Snapshot only a powered-off VM. Copying a live disk gives you an inconsistent image.
- A blanked guest screen hangs `grim` forever instead of failing, so `omavm shot` appears to freeze. `provision` therefore disables the screensaver and keeps the screen awake.
- A fresh install has empty pacman databases, because everything came from the mirror bundled on the ISO. Anything you install afterwards needs a `pacman -Sy` first.
- Moving files: `omavm push` and `omavm pull`. From inside the guest the host is the default network's gateway, `192.168.122.1`.
- Hyprland in the guest is Quattro, so `hyprctl dispatch` takes lua: `hl.dsp.focus({ workspace = 2 })`.

## Using it with an agent

`skill/SKILL.md` is an [agent skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) describing this setup, so Claude Code, Codex or another CLI agent can drive the VM without rediscovering the pitfalls above. Point your agent's skills directory at it:

```bash
# from the root of this repository
ln -s "$PWD/skill" ~/.claude/skills/vm
```

## Credits

The unattended install builds on how [omacom-io/omarchy-iso](https://github.com/omacom-io/omarchy-iso) installs itself from a `CIDATA` drive. Earlier versions of omavm ran QEMU through that repository's `omarchy-iso-boot`; it now runs everything through libvirt.

This whole thing started with [DHH answering a question about it](https://x.com/dhh/status/2094856301662835158) on 1 September 2026:

> You can use QEMU. Talk to your agent about it 😄. Tell it to look at omarchy-iso-boot in omacom/omarchy-iso.

So that is what happened, and this repository is where that conversation ended up.

## License

MIT, see [LICENSE](LICENSE).
