# claude-desktop-fedora

Install and update the **Claude Desktop** app on **Fedora** by converting Anthropic's Debian package to an RPM.

One command downloads the newest release, checks it against Anthropic's published checksum, converts it,
and installs it:

```bash
claude-desktop-update
```

> **Unofficial.** This project is not made or endorsed by Anthropic. "Claude" is a trademark of Anthropic.
> The script does not contain or redistribute any Anthropic software: it downloads the package from
> Anthropic's own server each time you run it.

## Why this exists

Anthropic ships the Claude Desktop app for Linux (beta) as a `.deb` from an apt repository, and
[their documentation](https://code.claude.com/docs/en/desktop-linux) says only Debian-based distributions
are supported today. On Fedora you can convert that `.deb` to an `.rpm` with `alien`, but a plain conversion
fails to install (see [Troubleshooting](#troubleshooting)). This script does the conversion with the fixes.

## Requirements

- Fedora on x86_64 with `dnf5` (the default package manager since Fedora 41)
- These packages:

```bash
sudo dnf install alien fakeroot rpm-build dpkg curl python3
```

Status: the whole process (build and install) has been used on Fedora 44. On Fedora 45 (RPM 6) the build
and a dry-run install work; if the final install step fails for you, please open an issue with the output.

## Install the script

```bash
git clone https://github.com/hk47isback/claude-desktop-fedora.git
cd claude-desktop-fedora
install -Dm755 claude-desktop-update ~/.local/bin/claude-desktop-update
```

`~/.local/bin` is on your `PATH` by default on Fedora. You can also just run `./claude-desktop-update` from the
cloned folder.

## Use it

Install or update Claude Desktop:

```bash
claude-desktop-update
```

Build the RPM but do not install it (it prints the install command instead):

```bash
NO_INSTALL=1 claude-desktop-update
```

The script asks for your `sudo` password at the install step. When it finishes, **quit Claude completely and
open it again**. If you ran the script from inside a Claude chat, that chat ends when the app restarts.

Run the same command again whenever you want to update. The app does not update itself on Linux, and Fedora's
updater will not see it, so you update it by running this script. If you are already on the newest version, it
says so and does nothing.

If you already downloaded the `.deb` into your home folder or `~/Downloads`, the script reuses it (after
checking it) instead of downloading it again.

## What the script does

1. Reads the package index of Anthropic's apt repository
   (`https://downloads.claude.ai/claude-desktop/apt/stable`) and finds the newest `claude-desktop` version,
   its file path and its SHA-256.
2. Stops if that version is already installed.
3. Downloads the `.deb` (or reuses a matching copy), and **checks its SHA-256**. A mismatch aborts the run.
4. Converts it with `alien` to an RPM build tree.
5. Edits the generated spec file so the package no longer claims system directories such as `/usr/bin` and
   `/usr/lib`. `alien` lists them as part of the package, and Fedora's `filesystem` package owns them, which
   makes the install fail.
6. Builds the RPM with `rpmbuild` and checks that none of those system directories ended up in it.
7. Installs it with `sudo dnf5 install --no-gpgchecks`.

Everything is built in `~/.cache/claude-desktop-fedora` (wiped at the start of each run).

## What is and is not verified

- The SHA-256 comes from the package index that Anthropic's server publishes over HTTPS. This confirms you got
  the file that server listed, so it protects against a corrupted or swapped download. It does **not** verify
  Anthropic's apt GPG signature. You are trusting the `downloads.claude.ai` host.
- The RPM is built on your machine, so it is **unsigned**. Fedora 45 and newer (RPM 6) refuse unsigned
  packages unless told otherwise, so the script installs with `--no-gpgchecks`. That flag applies to the whole
  transaction. The script prints a reminder: the list should show **only** `claude-desktop` being installed
  or upgraded. If dnf wants to add other packages, answer `N`.
- Read the script before you run it. It is one short file, and it runs `sudo` at the end.

## What is different from the Debian package

`alien` does not convert the Debian install scripts, and this script does not add them back. On Debian they
would register Anthropic's apt repository and signing key, write an AppArmor profile, register a GNOME Shell
search provider, add a `ccd` command link, and register the `.mcpb`, `.dxt` and `.skill` file types. None of
that is needed to run the app on Fedora, but the search provider and file-type associations will be missing.

The app's optional **Cowork** feature needs QEMU, KVM and related packages. See Anthropic's
[requirements](https://code.claude.com/docs/en/desktop-linux#cowork-requirements) and install the Fedora
equivalents yourself if you use it.

## Troubleshooting

**`file /usr/bin from install of claude-desktop ... conflicts with file from package filesystem`**
A plain `alien --to-rpm` package claims system directories. This script removes them. If you see this, you
are installing an RPM that was not built by the script.

**`package ... does not verify: no signature`** (or the install is refused for lack of a signature)
Expected for a locally built package on RPM 6. The script passes `--no-gpgchecks`. If you install by hand,
add the same flag: `sudo dnf5 install --no-gpgchecks <the rpm>`.

**`directory //: remove failed: Device or resource busy`** while upgrading
Harmless. Older conversions listed the root directory as part of the package, and the system cannot remove it.

**`Remote file name has no length` or the index cannot be fetched**
Your network cannot reach `downloads.claude.ai`, or Anthropic changed the repository layout. Open an issue.

## Uninstall

```bash
sudo dnf remove claude-desktop
rm ~/.local/bin/claude-desktop-update
rm -rf ~/.cache/claude-desktop-fedora
```

Your chats and settings are in `~/.config/Claude`. The commands above leave them alone.

## License

[MIT](LICENSE). This license covers only the script in this repository, not Anthropic's software.
