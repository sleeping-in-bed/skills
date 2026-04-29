---
name: wsl-chrome-ime
description: Install Google Chrome in WSL2/Ubuntu and make Chinese input work in WSLg windows using fcitx5, fonts, locale setup, and a Chrome launcher tuned for WSLg limitations.
---

# WSL Chrome IME

## Purpose

Use this skill when the user wants to:

- install Google Chrome inside WSL2 Ubuntu
- launch Chrome as a Linux GUI app through WSLg
- fix Chinese or other non-ASCII text rendering in Linux GUI apps
- make Chinese input work inside Chrome on WSLg

This skill is specifically for WSL2 + Ubuntu + WSLg on Windows 11. It is optimized for the real behavior we validated on a working setup, not for a generic Linux desktop.

## Important Boundary

Do not assume Windows IME can be reused directly inside Linux Chrome.

On WSLg, Chromium/Chrome input over native Wayland IME is often unreliable. A validated workaround is:

1. install Linux-side `fcitx5`
2. disable `fcitx5`'s `wayland` and `waylandim` addons for this case
3. launch Chrome through XWayland with `--ozone-platform=x11`
4. add `--disable-gpu` if Chrome crashes under X11 on WSLg

This is the path to prefer when the user says:

- "Chrome in WSL can only type English"
- "Chinese IME does not work in WSLg Chrome"
- "WSL Chrome opens, but Windows input method does nothing"

## Preconditions

Check the environment first:

```bash
cat /etc/os-release
wsl.exe --version
echo "DISPLAY=$DISPLAY"
echo "WAYLAND_DISPLAY=$WAYLAND_DISPLAY"
echo "XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR"
```

If `WAYLAND_DISPLAY` is empty and GUI apps do not launch, stop and fix WSLg first.

## Install Google Chrome

Install Chrome from Google's official Debian package:

```bash
cd /tmp
wget -O google-chrome-stable_current_amd64.deb \
  https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo apt-get update
sudo apt-get install -y ./google-chrome-stable_current_amd64.deb
google-chrome --version
```

Useful verification:

```bash
command -v google-chrome
google-chrome --version
```

## Fix Fonts And Locale

If Chinese or non-ASCII characters render as tofu or boxes, install CJK fonts and generate UTF-8 locales:

```bash
sudo apt-get update
sudo apt-get install -y fonts-noto-cjk
sudo localedef -i en_US -f UTF-8 en_US.UTF-8 || true
sudo localedef -i zh_CN -f UTF-8 zh_CN.UTF-8 || true
locale -a | grep -Ei 'en_US|zh_CN|C\.utf8'
```

Optionally let Linux fontconfig reuse Windows fonts:

Create `$HOME/.config/fontconfig/fonts.conf`:

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<fontconfig>
  <dir>/mnt/c/Windows/Fonts</dir>

  <alias>
    <family>sans-serif</family>
    <prefer>
      <family>Microsoft YaHei UI</family>
      <family>Microsoft YaHei</family>
      <family>SimSun</family>
      <family>Segoe UI</family>
    </prefer>
  </alias>
</fontconfig>
```

Refresh cache:

```bash
fc-cache -f ~/.config/fontconfig /mnt/c/Windows/Fonts >/dev/null 2>&1 || true
fc-match 'sans:lang=zh-cn'
```

## Install Fcitx5

Install the input method stack:

```bash
sudo apt-get update
sudo apt-get install -y \
  fcitx5 \
  fcitx5-chinese-addons \
  fcitx5-frontend-gtk2 \
  fcitx5-frontend-gtk3 \
  fcitx5-frontend-gtk4 \
  fcitx5-frontend-qt5 \
  fonts-noto-cjk \
  dbus-x11 \
  im-config
```

Set the distro's default input method:

```bash
im-config -n fcitx5
cat ~/.xinputrc
```

`~/.xinputrc` should contain:

```bash
run_im fcitx5
```

## Use A WSLg-Specific Fcitx5 Profile

Create `$HOME/.config/fcitx5/profile`:

```ini
[Groups/0]
Name=Default
Default Layout=us
DefaultIM=pinyin

[Groups/0/Items/0]
Name=keyboard-us
Layout=

[Groups/0/Items/1]
Name=pinyin
Layout=

[GroupOrder]
0=Default
```

This ensures the default group contains both English keyboard and Pinyin.

## Set Session Environment Variables

Create `$HOME/.config/environment.d/90-fcitx5.conf`:

```ini
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
SDL_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
INPUT_METHOD=fcitx
LANG=en_US.UTF-8
LC_CTYPE=zh_CN.UTF-8
XDG_SESSION_TYPE=wayland
```

These help Linux GUI apps find the `fcitx5` input method.

## Use An X11 Chrome Launcher

Do not rely on Chrome's native Wayland IME path for WSLg.

Validated reason:

- `fcitx5` may fail with `permission to bind input_method denied`
- this comes from WSLg's Wayland input-method restrictions

Create `$HOME/.local/bin/start-fcitx5-wslg`:

```bash
#!/usr/bin/env bash
set -euo pipefail

export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export SDL_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export INPUT_METHOD=fcitx
export LANG=en_US.UTF-8
export LC_CTYPE=zh_CN.UTF-8

pkill -x fcitx5 >/dev/null 2>&1 || true
exec fcitx5 -d -r --disable=wayland,waylandim
```

Create `$HOME/.local/bin/google-chrome-wslg`:

```bash
#!/usr/bin/env bash
set -euo pipefail

export LANG=en_US.UTF-8
export LC_CTYPE=zh_CN.UTF-8
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export SDL_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export INPUT_METHOD=fcitx

if ! pgrep -x fcitx5 >/dev/null 2>&1; then
  nohup fcitx5 -d -r --disable=wayland,waylandim >/tmp/fcitx5-wslg.log 2>&1 &
  sleep 1
fi

exec /usr/bin/google-chrome-stable \
  --ozone-platform=x11 \
  --disable-gpu \
  "$@"
```

Mark them executable:

```bash
chmod +x ~/.local/bin/start-fcitx5-wslg ~/.local/bin/google-chrome-wslg
```

## Override The Desktop Launcher

Create `$HOME/.local/share/applications/google-chrome.desktop`:

```ini
[Desktop Entry]
Version=1.0
Name=Google Chrome
GenericName=Web Browser
Comment=Access the Internet
Exec=/home/<linux-user>/.local/bin/google-chrome-wslg %U
StartupNotify=true
Terminal=false
Icon=google-chrome
Type=Application
Categories=Network;WebBrowser;
```

Also add autostart for fcitx5:

Create `$HOME/.config/autostart/fcitx5-wslg.desktop`:

```ini
[Desktop Entry]
Type=Application
Version=1.0
Name=Fcitx5 (WSLg)
Exec=/home/<linux-user>/.local/bin/start-fcitx5-wslg
Terminal=false
X-GNOME-Autostart-enabled=true
```

Replace `<linux-user>` with the real Linux username, for example `/home/a/...`.

Refresh desktop metadata:

```bash
update-desktop-database ~/.local/share/applications >/dev/null 2>&1 || true
```

## Start And Verify

Start `fcitx5`:

```bash
nohup ~/.local/bin/start-fcitx5-wslg >/tmp/fcitx5-wslg.log 2>&1 &
sleep 2
pgrep -af '^fcitx5'
sed -n '1,80p' /tmp/fcitx5-wslg.log
```

Healthy indicator:

- `fcitx5` process stays alive
- log shows `Override Disabled Addons: {waylandim, wayland}`
- log no longer ends with `permission to bind input_method denied`

Optional check:

```bash
fcitx5-configtool
```

Start Chrome:

```bash
~/.local/bin/google-chrome-wslg
```

Healthy indicator:

```bash
pgrep -fa '/opt/google/chrome/chrome' | sed -n '1,20p'
```

Expected flags include:

- `--ozone-platform=x11`
- `--disable-gpu`

## User-Facing Notes

- This setup uses Linux-side `fcitx5`, not direct Windows IME passthrough.
- It is acceptable if other Linux GUI apps still use Wayland; this workaround is specifically for Chrome/Chromium input.
- If Chrome works but input still stays English, tell the user to try `Ctrl+Space` to toggle Pinyin.
- If text renders fine but input is still broken, inspect `/tmp/fcitx5-wslg.log`.

## Troubleshooting

- `permission to bind input_method denied`
  - Cause: WSLg rejected the native Wayland input-method binding.
  - Fix: keep `fcitx5` started with `--disable=wayland,waylandim` and keep Chrome on `--ozone-platform=x11`.

- Chrome crashes on startup under X11
  - Add or keep `--disable-gpu`.

- Chinese still renders as tofu
  - Ensure `fonts-noto-cjk` is installed.
  - Rebuild font cache.
  - Recheck `fc-match 'sans:lang=zh-cn'`.

- `fcitx5` starts but there is no Pinyin
  - Recreate `~/.config/fcitx5/profile`.
  - Run `fcitx5-configtool`.

- User asks for direct Windows IME reuse
  - Explain that this is not the reliable path on WSLg Chrome.
  - Prefer Linux-side `fcitx5`.
