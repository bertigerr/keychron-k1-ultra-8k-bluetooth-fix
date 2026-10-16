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

The custom image was transferred over the keyboard's Realtek HID DFU interface (HID Usage Page `0x008C`, model `KCZKK18K`). The tool reported successful transfer, **CRC verified OK**, and transmission of the `IMAGE_SWITCH` command. After reboot, the keyboard responded to DFU handshake and Bluetooth pairing succeeded. The keyboard subsequently worked over Bluetooth on multiple devices.

This README deliberately does not include a one-click flash command: firmware flashing should only be attempted after verifying hardware compatibility and an appropriate recovery procedure. The owner reported that Keychron support recommended recovery through Keychron Launcher, but recovery from a firmware image that prevents USB/DFU initialization has **not** been tested.

## What this does — and does not — prove

`CONFIG_BT_AUTO_PHY_UPDATE=n` disables **automatic host-initiated PHY update requests** in the firmware. It does **not** necessarily disable the Realtek controller's support for LE 2M or prevent a central device from requesting a PHY change.

No new HCI capture was collected after the successful flash to establish the active PHY. The result is a **confirmed workaround for this hardware combination**, not proof that all Realtek adapters or Keychron Ultra models share the same root cause.

## References

- [Keychron's `rtl8762g` ZMK source](https://github.com/Keychron/zmk/tree/rtl8762g)
- [ZMK Bluetooth connection troubleshooting](https://zmk.dev/docs/troubleshooting/connection-issues)
- [Community Realtek HID flasher source](https://github.com/komax20000-cloud/KeychronK1Ultra8K/tree/main/tools/flasher) (third-party; review before use)
