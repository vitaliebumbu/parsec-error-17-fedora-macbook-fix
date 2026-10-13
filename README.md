# Fix Parsec Error -17 on Fedora Linux and Intel MacBook Pro

Resolve Parsec **failed connection error -17** on Fedora with an older Intel GPU by restoring H.264 decoding support. Verified on Fedora 44 Workstation and a 2015 MacBook Pro (MacBookPro11,4).

The working configuration uses **Intel i965 VA-API hardware decoding, H.264, and an FFmpeg compatibility library**. A remote desktop connection succeeded after these changes, and Parsec logged `FFMPEG 4 Hardware` without another error -17 during the observed session.

## Tested system

| Component | Tested configuration |
| --- | --- |
| Computer | Apple MacBook Pro 15-inch, Mid-2015; MacBookPro11,4 |
| CPU | Intel Core i7-4770HQ @ 2.20 GHz, 4 cores / 8 threads |
| GPU | Intel Iris Pro / Haswell Crystal Well, PCI `8086:0d26` |
| Memory | 16 GB |
| OS | Fedora Linux 44 Workstation |
| Kernel | `7.2.9-201.fc44.x86_64` |
| Desktop | GNOME on Wayland |
| Parsec | `150-105c` after automatic update; initial failure also observed on `150-104a` |
| Intel VA-API driver | `libva-intel-driver-2.4.1^20241027gitd30e013-3.fc44` |
| Codec package | `libavcodec-freeworld-8.1.3-1.fc44` |
| Compatibility library | `compat-ffmpeg4-4.4.8-1.fc44` |

Verified October 12, 2026. Other machines and Fedora releases have not been tested. Use this fix for decoder failures on comparable Intel hardware; it does not repair network negotiation errors.

## Symptoms and cause

The network connection reached the host, but video startup ended with:

```text
* decode_frame[341] = -17
```

Parsec defines -17 as a video decoder problem. This installation had three missing pieces:

1. The Haswell GPU needed the legacy `i965` driver. The installed `iHD` driver failed to initialize on this GPU.
2. Fedora's installed `ffmpeg-free` codec library did not expose H.264 decoding. `libavcodec-freeworld` supplied the missing codecs.
3. The Parsec binary requested `avcodec_close`, which was unavailable in the installed FFmpeg 8 library. `compat-ffmpeg4` supplied the older library and required functions. Parsec subsequently loaded `libavcodec.so.58` and `libavutil.so.56` successfully.

The GPU reported H.264 decode profiles, but no HEVC decode profile. H.265, 10-bit color, and 4:4:4 were disabled for the tested connection.

## Installation

Install Parsec separately using its official download. Fully quit Parsec before editing its configuration. Do not run the user configuration steps with sudo.

### 1. Enable RPM Fusion Free

If RPM Fusion Free is already configured, skip this command. Check the [RPM Fusion configuration instructions](https://rpmfusion.org/Configuration) for your release.

```bash
sudo dnf install "https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm"
```

### 2. Install decoder packages

```bash
sudo dnf install libva-intel-driver.x86_64 libavcodec-freeworld.x86_64 compat-ffmpeg4.x86_64 libva-utils
```

This adds the legacy Intel driver, missing codecs, and a compatibility library alongside the system FFmpeg installation. No kernel replacement or graphics-driver removal is required. If DNF reports a version conflict, resolve matching packages for your Fedora release; do not force installation with ignored dependencies.

### 3. Configure Parsec

In Parsec **Settings → Client**:

- Decoder: **Hardware**
- H.265: **Off**
- 10-bit color: **Off**
- Prefer 4:4:4 Color: **Off**

Alternatively, quit Parsec, back up its configuration, and append supported options:

```bash
cp ~/.parsec/config.json ~/.parsec/config.json.before-error17-fix
cp ~/.parsec/config.txt ~/.parsec/config.txt.before-error17-fix
cat >> ~/.parsec/config.txt <<'EOF'
client_decoder_index = 0
client_decoder_h265 = false
client_decoder_10bit = false
client_decoder_444 = false
EOF
```

The tested version migrated these settings into `config.json` on startup. `client_decoder_index=0` selected the working hardware decoder on this installation; decoder enumeration may vary between versions and platforms. Prefer selecting Hardware in the interface if the option differs.

The documented `decoder_software=1` option was not retained by the tested version and did not solve this installation's failure.

### 4. Launch with the legacy Intel driver

Use the path to your installed Parsec launcher. For example:

```bash
LIBVA_DRIVER_NAME=i965 parsecd
```

For the tested installation, the executable was `/usr/local/bin/parsecd`. Its existing user launcher also supplied `libjpeg.so.8` from a private application library directory. That existing compatibility setup was preserved; these decoder packages do not replace it.

If an existing launcher sets `LD_LIBRARY_PATH`, retain that line and add this before its `exec` line:

```bash
export LIBVA_DRIVER_NAME=i965
```

Apply this environment variable to Parsec's launcher rather than globally. Update the desktop shortcut to use that launcher if needed. Fully restart Parsec, then reconnect. No reboot was needed for this fix.

## Verification

Check hardware decoding support:

```bash
LIBVA_DRIVER_NAME=i965 vainfo --display drm --device /dev/dri/renderD128
```

Look for `Intel i965 driver for Intel(R) Haswell Mobile` and `VAProfileH264High : VAEntrypointVLD`. The render device can differ; inspect `/dev/dri/` first. A warning about optional `hybrid_drv_video.so` did not prevent H.264 decoding on the tested machine.

Check software codec availability:

```bash
ffmpeg -hide_banner -decoders | grep -E ' h264 | hevc '
```

After connecting, inspect the decoder lines:

```bash
grep -E 'FFMPEG|decode_frame' ~/.parsec/log.txt | tail -20
```

The successful session reported:

```text
FFMPEG 4 Hardware
```

Old error -17 entries remain in the log; compare timestamps with the latest restart. The complete verification included decoding 30 frames of a 1280×720 H.264 test clip through VA-API, followed by a successful remote desktop connection. This was a functional test, not a long-term stability or latency benchmark.

## Rollback

Quit Parsec, restore both saved configuration files, and remove the `LIBVA_DRIVER_NAME=i965` line from the launcher:

```bash
cp ~/.parsec/config.json.before-error17-fix ~/.parsec/config.json
cp ~/.parsec/config.txt.before-error17-fix ~/.parsec/config.txt
```

Restart Parsec. The additional packages can remain installed for other video applications. If you remove them, review DNF's proposed transaction first; other applications may use them. Keep any RPM Fusion configuration that was already present before this fix.

## Troubleshooting

- **Still -17:** confirm Parsec was completely restarted and that the working launcher sets `LIBVA_DRIVER_NAME=i965`. Confirm the compatibility packages installed successfully.
- **Driver initialization error:** verify the GPU model and render-device permissions. This driver recommendation is specific to older supported Intel GPUs.
- **Different connection error:** inspect Parsec's error documentation; router and network failures need separate diagnosis.
- **Missing `libjpeg.so.8`:** repair the Parsec installation's application dependencies separately. Do not symlink incompatible library versions.
- **Decoder works but performance is poor:** start with a lower host resolution and frame rate; the tested GPU is from an older hardware generation.

When reporting an issue, include Fedora release, kernel, GPU model, Parsec version, decoder package versions, and the relevant error line. Redact IP addresses, account identifiers, host names, session identifiers, and credentials from logs.

## References

- [Parsec error -17](https://support.parsec.app/hc/en-us/articles/34984639667092-Error-Codes-17-Your-device-had-trouble-using-the-video-decoder)
- [Parsec advanced configuration](https://support.parsec.app/hc/en-us/articles/32381443626516-All-Advanced-Configuration-Options)
- [Intel legacy VA-API driver](https://github.com/intel/intel-vaapi-driver)
- [RPM Fusion multimedia packages](https://rpmfusion.org/Howto/Multimedia)

## License

Documentation is provided under the MIT license. Parsec and the referenced third-party packages retain their own licenses. This repository does not redistribute Parsec, firmware, credentials, or application binaries.
