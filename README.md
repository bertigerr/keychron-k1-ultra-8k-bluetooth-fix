# Keychron K1 Ultra 8K Bluetooth pairing fix (Realtek / Windows 11)

A reproducible workaround for Bluetooth pairing failures between the **Keychron K1 Ultra 8K ANSI (knob)** and a **Realtek Bluetooth adapter** on Windows 11.

**Tested outcome:** The keyboard failed to pair with the original configuration, but paired successfully after a custom ZMK build with **`CONFIG_BT_AUTO_PHY_UPDATE=n`**. Following the change, Bluetooth also worked on multiple devices.

> **Warning:** Flashing custom firmware can leave a keyboard unusable. Verify the exact keyboard model, have a recovery plan, and do not flash an image built for another variant. This repository does **not** distribute firmware binaries.

## Tested hardware and software

| Component | Details |
| --- | --- |
| Keyboard | Keychron K1 Ultra 8K ANSI with knob |
| DFU-reported model | `KCZKK18K` |
| Keyboard USB VID:PID | `3434:1610` |
| Original reported firmware | `1.0.1-r` |
| Laptop | Lenovo Legion 5 15ACH6H (82JU) |
| Built-in Bluetooth | Realtek (`0BDA:4852`, RTL8852A family) |
| OS | Windows 11 |
| Diagnostic environment | Ubuntu in WSL2, BlueZ, `usbipd` USB passthrough |
| Firmware | [`Keychron/zmk`, `rtl8762g`](https://github.com/Keychron/zmk/tree/rtl8762g), tested commit `7f8d16732a3782e237915bff30a1f67de09268b7` |
| Board / shield | `rtl8762gtu_kb` / `keychron_k1_ultra_ansi` |

## Symptom

Windows discovered the keyboard, but Bluetooth pairing repeatedly failed. Changing between several Realtek driver versions did not solve the problem.

## Reproducing the PHY-related failure

The Realtek Bluetooth adapter was passed through to Ubuntu/BlueZ using `usbipd` in WSL2. PHY modes were varied with `btmgmt`.

**Allow LE 1M only:**

```bash
sudo btmgmt --index 0 phy BR1M1SLOT LE1MTX LE1MRX
```

**Observed:** Pairing and connection succeeded (`Bonded: yes`, `Connected: yes`).

**Allow LE 1M and LE 2M:**

```bash
sudo btmgmt --index 0 phy BR1M1SLOT LE1MTX LE1MRX LE2MTX LE2MRX
```

**Observed:** Pairing failed. HCI logs showed a transition to **LE 2M**, followed approximately 0.48 seconds later by an HCI disconnect with reason **`0x08` (`Connection Timeout`)**.

This strongly suggested an interoperability problem involving PHY negotiation. It does not establish which device's Bluetooth stack is at fault.

## Workaround

In the existing, configured Keychron ZMK build workspace, add the included file `no-auto-phy.conf`:

```ini
CONFIG_BT_AUTO_PHY_UPDATE=n
```

From the workspace root (with the correct Keychron `rtl8762g` source, Zephyr SDK and `west` environment already configured):

```bash
west build -p always \
  -s app -d app/build \
  -b rtl8762gtu_kb \
  -- \
  -DSHIELD=keychron_k1_ultra_ansi \
  -DEXTRA_CONF_FILE="$PWD/no-auto-phy.conf"
```

**Important build notes:**

- This example assumes a working Keychron-specific west workspace; it is **not** a complete clean-machine setup guide.
- In the tested source checkout, an additional local Kconfig declaration for `RTK_DFU` was required for the build to succeed. Do not assume the upstream branch builds unmodified in every environment. The missing declaration added to `app/Kconfig` was:

  ```kconfig
  config RTK_DFU
      bool "Enable Realtek DFU"
      default y
  ```

  Confirm the symbol is actually missing before editing; this is a build-specific workaround, not an upstream recommendation.
- The Keychron build's post-processing step used a fixed `app/build/zephyr/zmk.bin` path, so the tested baseline and modified images were built **sequentially into `app/build`**. Using an alternate build directory without adapting that post-processing step can produce a misleading output file.
- Compare the generated `app/build/zephyr/.config` against a baseline. In the successful experiment, the **only** Kconfig difference was:

```diff
-CONFIG_BT_AUTO_PHY_UPDATE=y
+# CONFIG_BT_AUTO_PHY_UPDATE is not set
```

The experiment's post-processed `zmk.bin` was **305,628 bytes**, with SHA-256:

```text
8525e8b6570478b1618d04b58501432d0e8e022b79cb864de8938aae9490fd54
```

This checksum is provided to identify the **tested output**, not as an instruction to flash someone else's image.

## Flashing and outcome

The working firmware was flashed with **`Realtek_HID_flasher.exe`**, the Windows Realtek SC-FWU-over-HID utility from the [community flasher project](https://github.com/komax20000-cloud/KeychronK1Ultra8K/tree/main/tools/flasher). This was **not** an official Keychron firmware updater. Its source should be reviewed before use.

### Reproducing the tested USB HID flashing workflow

1. Confirm you have a firmware image **built for K1 Ultra 8K ANSI (`KCZKK18K`)**, not an image intended for a different keyboard or layout. The image used in the successful experiment is identified by its SHA-256 above; it is **not distributed** here.
2. Connect the keyboard over USB. Close Keychron Launcher and any other application that might access the keyboard's HID interface.
3. From Windows PowerShell, in the folder containing `Realtek_HID_flasher.exe`, check the DFU HID interface and identify the keyboard:

   ```powershell
   .\Realtek_HID_flasher.exe handshake
   ```

   The known-good **post-flash** handshake from this experiment was:

   ```text
   >> GET_MODEL_INFO (0x60)...
      model = "KCZKK18K"   fw = "1.0.1-r"
      dfu_version = 0x0001 enc_mode = 0
      identity VALID (Keychron K1 Ultra 8K - ANSI)
   ```

   **Note:** The captured handshake was after flashing; it does not establish the precise pre-flash sequence or prove the keyboard is in a separate hardware bootloader mode.
4. Only if the model and firmware image are both confirmed compatible, run the flasher with the locally built firmware path:

   ```powershell
   .\Realtek_HID_flasher.exe flash .\no-auto-phy.bin
   ```

   `no-auto-phy.bin` is an **example filename** for your own verified build, not a downloadable file from this repository. The tool requests confirmation before uploading.
5. Do not disconnect the keyboard during the upload, device-side **CRC verification**, or `IMAGE_SWITCH`. The successfully tested run reported `CRC verified OK` and sent `IMAGE_SWITCH` before the keyboard restarted.
6. After restart, test Bluetooth pairing. In the original experiment, pairing succeeded and the keyboard also worked over Bluetooth on multiple devices.

**DFU entry and the spacebar reset method:** The flashing tool uses the keyboard's Realtek DFU **HID interface** (usage page `0x008C`, model `KCZKK18K`). No use of the physical reset button beneath the spacebar is recorded in the experiment's available command log; the exact button/switch sequence used to expose the DFU interface before flashing cannot be independently reconstructed from that log. Do **not** present the spacebar method as a tested prerequisite. First try the read-only `handshake` while connected over USB; if the interface cannot be found, stop rather than experimenting with an unverified bootloader or image.

**Risk and validation:** A valid model handshake does **not** establish `.bin` compatibility, and a failed update may render the keyboard unusable. The owner reported Keychron support recommending recovery with Keychron Launcher, but recovery when USB/DFU initialization is broken has **not** been tested. This workflow is confirmed on one K1 Ultra 8K ANSI only; independent user confirmations are not yet available.

## What this does — and does not — prove

`CONFIG_BT_AUTO_PHY_UPDATE=n` disables **automatic host-initiated PHY update requests** in the firmware. It does **not** necessarily disable the Realtek controller's support for LE 2M or prevent a central device from requesting a PHY change.

No new HCI capture was collected after the successful flash to establish the active PHY. The result is a **confirmed workaround for this hardware combination**, not proof that all Realtek adapters or Keychron Ultra models share the same root cause.

## References

- [Keychron's `rtl8762g` ZMK source](https://github.com/Keychron/zmk/tree/rtl8762g)
- [ZMK Bluetooth connection troubleshooting](https://zmk.dev/docs/troubleshooting/connection-issues)
- [Community Realtek HID flasher source](https://github.com/komax20000-cloud/KeychronK1Ultra8K/tree/main/tools/flasher) (third-party; review before use)
