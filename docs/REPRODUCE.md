# Step-by-step: diagnose, build, flash and verify the BLE workaround

This guide reproduces the successful **Keychron K1 Ultra 8K ANSI (`KCZKK18K`)** test. It separates **required steps to apply the workaround** from the **optional PHY experiment** that led to it. Read [the investigation journal](DIAGNOSTIC-JOURNAL.md) for the evidence and uncertainties.

> **Firmware flashing risk:** An incompatible or interrupted flash may render your keyboard unusable. Neither the image nor the flashing executable is distributed in this repository. The steps were validated on one keyboard; hardware recovery from a non-booting image was **not** validated. Do not flash if you have not independently verified your hardware model, the tool and a usable recovery path.

## A. Identify your hardware (Windows; read-only)

1. Confirm **Keychron K1 Ultra 8K ANSI with knob**. The tested DFU model string is `KCZKK18K`; images from other K1/Ultra/ISO/JIS variants must not be substituted.
2. Check the Windows Bluetooth controller:

   ```powershell
   Get-PnpDevice -Class Bluetooth | Select-Object FriendlyName, InstanceId, Status
   Get-CimInstance Win32_PnPSignedDriver |
     Where-Object { $_.DeviceID -like 'USB\VID_0BDA&PID_4852*' } |
     Select-Object DeviceName, DriverVersion, InfName
   ```

3. The tested host used Realtek `USB\VID_0BDA&PID_4852` in a Lenovo Legion 5 15ACH6H (82JU). Other Bluetooth adapters or keyboard variants are **untested**.

## B. Prepare the firmware build environment (Ubuntu / WSL2)

**What was tested:** an existing Keychron-specific `west`/Zephyr workspace rooted at `~/keychron-workspace/zmk`, with the ARM Zephyr toolchain already installed. The commands below reconstruct the source checkout and build; setting up a brand-new SDK installation end-to-end has **not** been independently validated.

1. Install or verify Git, CMake, Ninja, Python, `west` and an ARM-capable Zephyr SDK. For a new Linux environment, consult the [official Zephyr setup guide](https://docs.zephyrproject.org/latest/develop/getting_started/index.html). Do not assume generic upstream ZMK or a normal upstream Zephyr checkout supports the Realtek board.
2. Create a Keychron manifest workspace and fetch dependencies:

   ```bash
   mkdir -p ~/keychron-workspace/zmk
   cd ~/keychron-workspace/zmk
   west init -m https://github.com/Keychron/zmk --mr rtl8762g .
   west update
   ```

3. Pin the **source revisions observed in the successful experiment**. These pins must be applied *after* `west update` because the manifests also reference mutable branch names:

   ```bash
   git -C app fetch origin 7f8d16732a3782e237915bff30a1f67de09268b7
   git -C app checkout --detach 7f8d16732a3782e237915bff30a1f67de09268b7
   git -C zephyr fetch origin 2c11634850f7d6b04e55a253d4506a6e7c8bd673
   git -C zephyr checkout --detach 2c11634850f7d6b04e55a253d4506a6e7c8bd673
   git -C modules/hal/realtek fetch origin 3c8e0bf787d80bc98c0fe4041f660ee1bf091206
   git -C modules/hal/realtek checkout --detach 3c8e0bf787d80bc98c0fe4041f660ee1bf091206
   ```

4. Verify the source revisions:

   ```bash
   git -C app rev-parse HEAD
   git -C zephyr rev-parse HEAD
   git -C modules/hal/realtek rev-parse HEAD
   west --version
   ```

   The first three values should match the commit IDs above. **Do not run `west update` again after pinning** unless you re-checkout these revisions.

5. Install the Keychron fork's Python requirements in the already-configured build environment if they are not installed:

   ```bash
   python3 -m pip install -r zephyr/scripts/requirements.txt -r app/scripts/requirements.txt
   ```

   If system Python blocks direct `pip` installs, use a Python virtual environment. Make sure a compatible Zephyr ARM compiler/toolchain is active before building.

## C. Establish a baseline, then build the one-setting fix

1. From the workspace root (`~/keychron-workspace/zmk`), build the **baseline** to verify the toolchain and board configuration:

   ```bash
   west build -p always -s app -d app/build -b rtl8762gtu_kb -- -DSHIELD=keychron_k1_ultra_ansi
   cp app/build/zephyr/.config ~/keychron-workspace/k1-baseline.config
   ```

2. **Build-specific Kconfig issue:** In the original tested checkout, building initially required a missing declaration for `RTK_DFU` in `app/Kconfig`:

   ```kconfig
   config RTK_DFU
       bool "Enable Realtek DFU"
       default y
   ```

   Apply this **only when the build reports the symbol as missing** and you have verified the checkout; it was an experimental local workaround, **not** an upstream change that everyone should apply. Re-run the baseline before continuing if you had to change Kconfig.

3. Download this repo's **only functional change** to the workspace root:

   ```bash
   curl -fsSL https://raw.githubusercontent.com/bertigerr/keychron-k1-ultra-8k-bluetooth-fix/main/no-auto-phy.conf -o no-auto-phy.conf
   cat no-auto-phy.conf
   ```

   Expected content:

   ```ini
   CONFIG_BT_AUTO_PHY_UPDATE=n
   ```

4. Build the modified firmware in the **same build directory** as the baseline:

   ```bash
   west build -p always -s app -d app/build -b rtl8762gtu_kb -- \
     -DSHIELD=keychron_k1_ultra_ansi \
     -DEXTRA_CONF_FILE="$PWD/no-auto-phy.conf"
   ```

   The original Keychron post-processing step reads a fixed `app/build/zephyr/zmk.bin` path. Build baseline and modified firmware **sequentially**; using a different directory without adapting post-processing can produce a misleading artifact.

5. Check the resulting Kconfig and generated image:

   ```bash
   grep -E '^(CONFIG_BT_AUTO_PHY_UPDATE=|# CONFIG_BT_AUTO_PHY_UPDATE is not set)' app/build/zephyr/.config
   diff -u ~/keychron-workspace/k1-baseline.config app/build/zephyr/.config
   stat -c '%s bytes' app/build/zephyr/zmk.bin
   sha256sum app/build/zephyr/zmk.bin
   ```

   In the **recorded** run, the *only Kconfig difference* was:

   ```diff
   -CONFIG_BT_AUTO_PHY_UPDATE=y
   +# CONFIG_BT_AUTO_PHY_UPDATE is not set
   ```

   The recorded processed image was **305,628 bytes**, SHA-256 `8525e8b6570478b1618d04b58501432d0e8e022b79cb864de8938aae9490fd54`. A different toolchain or unpinned dependency can change binary hashes; do not treat size or SHA alone as proof of firmware safety.

6. Transfer **your own** built `app/build/zephyr/zmk.bin` to a Windows-accessible location and name it `no-auto-phy.bin`. This repository intentionally does not host a prebuilt `.bin`.

## D. Flash on Windows (Realtek HID interface)

1. Review the source of [the third-party Realtek HID flasher](https://github.com/komax20000-cloud/KeychronK1Ultra8K/tree/main/tools/flasher). The successful experiment used its `Realtek_HID_flasher.exe`, **not** the official Keychron Launcher firmware updater.
2. Connect the keyboard to Windows over USB-C. Close Keychron Launcher and other HID tools. The exact pre-flash sequence for exposing the DFU interface was **not preserved** in the test log; do not assume that pressing a reset button beneath the spacebar is required.
3. Open PowerShell in the directory containing the flasher and use the **read-only** handshake:

   ```powershell
   .\Realtek_HID_flasher.exe handshake
   ```

   Stop unless the interface is detected and the model reports `KCZKK18K`. **The recorded handshake below occurred after flashing**, not before:

   ```text
   >> GET_MODEL_INFO (0x60)...
      model = "KCZKK18K"   fw = "1.0.1-r"
      dfu_version = 0x0001 enc_mode = 0
      identity VALID (Keychron K1 Ultra 8K - ANSI)
   ```

4. **Only after confirming hardware/image compatibility and recovery options**, invoke flashing with the compiled file placed in the current directory:

   ```powershell
   .\Realtek_HID_flasher.exe flash .\no-auto-phy.bin
   ```

5. Confirm the image in the flasher prompt. Keep the USB cable connected through transfer, device-side `CRC verified OK`, `IMAGE_SWITCH`, and reboot. A model handshake alone does **not** verify that a `.bin` matches the keyboard.

## E. Validate that the fix worked

1. Return to Windows Bluetooth settings, remove any failed/stale keyboard pairing if necessary, and try a new Bluetooth pairing.
2. Confirm that the keyboard connects and types normally; disconnect/reconnect it and repeat on any additional hosts you can test.
3. **Original result:** The previously failing Realtek/Windows pairing succeeded after the modified firmware, and Bluetooth worked on multiple hosts.
4. **Do not claim LE 2M support is disabled.** The fix disables *automatic PHY update requests*; no post-flash HCI capture was obtained to prove the negotiated PHY.

## F. Optional: reproduce the LE 1M / LE 2M failure in Ubuntu

**This is diagnostic-only and not required to build or flash.** It was tested using WSL2 with a Bluetooth-capable kernel and `usbipd` USB passthrough. A default WSL2 kernel may lack usable Realtek/Bluetooth modules.

1. In an Administrator PowerShell, identify the Realtek USB BUSID using `usbipd list` (the tested machine reported **`2-4`**, but the identifier may differ). With that BUSID, run:

   ```powershell
   usbipd bind --busid 2-4
   usbipd attach --wsl --busid 2-4
   ```

2. In Ubuntu, verify the controller is visible with `bluetoothctl list` and identify the correct adapter index (the tested one used **`0`**). Start an HCI trace in a separate terminal with `sudo btmon` before each test.
3. Test pairing with 1M allowed and 2M excluded:

   ```bash
   sudo btmgmt --index 0 phy BR1M1SLOT LE1MTX LE1MRX
   ```

   Try pairing from `bluetoothctl`; the original test reached `Bonded: yes`, `Connected: yes`.
4. Enable both 1M and 2M PHY options and try again:

   ```bash
   sudo btmgmt --index 0 phy BR1M1SLOT LE1MTX LE1MRX LE2MTX LE2MRX
   ```

   In the original pre-fix trace, LE 2M was negotiated, followed about **0.48 seconds** later by disconnect `0x08` (`Connection Timeout`). Record your own HCI trace to verify whether the same failure occurs.
5. Once finished, return USB ownership to Windows:

   ```powershell
   usbipd detach --busid 2-4
   ```

   If your BUSID differs, replace `2-4` in all three commands. Do not leave the Bluetooth controller attached to WSL2 while troubleshooting Windows pairing.

## Troubleshooting and scope

- **`west build` fails:** Confirm correct Keychron `rtl8762g` fork, Zephyr/HAL pins, toolchain, `RTK_DFU` definition and fixed build path. A clean-machine build was not validated end-to-end.
- **Handshake cannot find the keyboard:** Close HID-using software, verify USB connection and model. Stop if the DFU HID interface is unavailable; the spacebar reset procedure was not validated.
- **Connected but still unstable:** Capture Windows/BlueZ HCI traffic, compare active PHY and disconnect codes; do not infer from the outcome alone that an adapter is defective.
- **Recovery:** Keychron Launcher was mentioned by Keychron support, but recovery from a firmware image that prevents USB/DFU initialization remains **untested**.
- **Independent confirmation:** None reported at publication time.
