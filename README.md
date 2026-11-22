# Cisco Packet Tracer 8.2.2 on an Immutable Distro (Distrobox + Podman)

Install Packet Tracer 8.x inside an offline Ubuntu 20.04 container. Cutting the
container's network after installation bypasses the Cisco login requirement, and
your host keeps internet.

Tested on Universal Blue (Bazzite/Bluefin). Works with **Packet Tracer 8.x only**
(9.x needs 22.04/24.04 and ships as an AppImage inside the .deb).

## Prerequisites

- `distrobox` and `podman` installed (default on Universal Blue images)
- The Packet Tracer 8.2.2 `.deb` downloaded from Cisco, e.g. in `~/Downloads/`

> Adjust the `.deb` filename/path in step 4 to match your file.

## 0. Teardown (only if rebuilding)

```bash
distrobox stop pt 2>/dev/null; distrobox rm -f pt 2>/dev/null
podman network rm -f no-network network 2>/dev/null
rm -f ~/.local/share/applications/*packettracer*.desktop
```

## 1. Create podman networks (host)

One internal network with no internet, one regular. Podman can't remove all
networks from a container, so we attach both and disconnect the online one later.

```bash
podman network create --internal no-network
podman network create network
```

## 2. Create the container (host)

`--unshare-netns` is essential. Without it, the container shares the host's
network and can never be taken offline.

```bash
distrobox create --name pt --image ubuntu:20.04 --unshare-netns --additional-flags "--network network"
distrobox enter pt
```

## 3. Install dependencies (inside container)

Ubuntu 20.04 still ships `libssl1.1` and the Qt5 libraries PT 8.2.2 needs.

```bash
sudo apt update
sudo apt install -y \
  wget libgl1 libxi6 libxtst6 libnss3 libx11-xcb1 libpulse0 pulseaudio \
  libssl1.1 libqt5core5a libqt5gui5 libqt5widgets5 libqt5network5 \
  libqt5xml5 libqt5script5 libqt5scripttools5 libqt5multimedia5 \
  libqt5websockets5 libqt5svg5 libqt5sql5 libqt5webenginewidgets5 \
  libqt5texttospeech5 libqt5printsupport5 libqt5webenginecore5 \
  libqt5webchannel5 libqt5positioning5 libqt5quick5 libqt5qml5 \
  libqt5networkauth5 qml-module-qtquick2 qml-module-qtquick-controls \
  qml-module-qtquick-controls2 qml-module-qtquick-dialogs \
  qml-module-qtquick-xmllistmodel qml-module-qt-labs-settings
```

## 4. Install Packet Tracer + export to host menu (inside container)

Accept the EULA when prompted, then exit back to the host.

```bash
sudo apt install -y ~/Downloads/PacketTracer822_amd64_signed_en-US_*.deb
distrobox-export -a packettracer
exit
```

## 5. Cut the container's internet (host)

```bash
podman network connect no-network pt
podman network disconnect network pt
podman container restart pt
```

## 6. Launch

From the "Packet Tracer" entry in your app menu, or:

```bash
distrobox enter pt -- packettracer
```

With no network, PT skips the login screen and runs in guest mode.

## Maintenance

- Nothing to update. The container is offline by design and PT doesn't need it.
- **Never run `podman system prune` / `podman container prune`**. It can delete
  the container if it's stopped.
- `.pt` project files live in your home directory and survive rebuilds.
- Host OS updates don't affect the container.

## Troubleshooting

- **Login screen still appears** → the container still has internet. Verify with
  `distrobox enter pt -- ping -c1 cisco.com` (should fail). If it succeeds, the
  container was created without `--unshare-netns`. Rebuild from step 0.
- **`packettracer` prints "Starting..." then nothing / core dump** → missing
  libraries. Re-run step 3; check with
  `distrobox enter pt -- bash -c 'cd /opt/pt/bin && ldd PacketTracer | grep "not found"'`.
- **No output when debugging** → the launcher script discards all output. Run
  the binary directly:
  `cd /opt/pt/bin && LD_LIBRARY_PATH=/opt/pt/bin ./PacketTracer`

## Credits

- Dependency list: [tdb11235 on Universal Blue forums](https://universal-blue.discourse.group/t/installing-cisco-packet-tracer-in-distrobox-not-working/10579/6)
- No-network method: [andersoncstd/packettracer-on-distrobox-guide](https://github.com/andersoncstd/packettracer-on-distrobox-guide)****
