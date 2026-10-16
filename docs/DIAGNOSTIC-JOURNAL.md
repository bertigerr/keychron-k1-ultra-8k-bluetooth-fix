# Investigation journal: K1 Ultra 8K × Realtek LE PHY pairing failure

**Experiment:** 15–16 October 2026. **Status:** workaround confirmed on one K1 Ultra 8K ANSI; root cause not isolated to either vendor's firmware.

This is an evidence-based reconstruction of how the workaround was found. It distinguishes observed behaviour, inference, and remaining uncertainties. For reproduction instructions, see [Step-by-step guide](REPRODUCE.md).

## Test environment

| Item | Tested configuration |
| --- | --- |
| Keyboard | Keychron K1 Ultra 8K ANSI with knob; USB `3434:1610`; DFU-reported `KCZKK18K` |
| Computer | Lenovo Legion 5 15ACH6H (82JU), Windows 11 |
| Built-in Bluetooth | Realtek USB `0BDA:4852`, RTL8852A family |
| Windows drivers examined | Realtek `1.9.1046.3002` (INF `oem21.inf` in the captured check) and `1.1071.2406.2401` |
| Diagnostic environment | Ubuntu running under WSL2 with USB passthrough via `usbipd`; working Bluetooth/Realtek modules in its Linux kernel |
| Firmware source | `Keychron/zmk`, `rtl8762g`, commit `7f8d16732a3782e237915bff30a1f67de09268b7` |

## Investigation sequence

### 1. Windows pairing fails

**Observed:** Windows could discover the keyboard, but pairing repeatedly failed. Switching between examined Realtek driver versions did not fix it. The same keyboard worked with other hosts. This narrowed the issue to a host/keyboard interoperability problem, but did not identify the faulty component.

**Eliminated workaround:** Windows Device Manager and registry inspection of the Realtek adapter exposed no documented, usable toggle to force LE 1M only. Scanning `oem21.inf` for `PHY`, `LE2M`, `2MPhy`, `DisableLE`, and `EnableLE` returned no matches. This does **not** prove the vendor driver lacks internal controls.

### 2. Reuse the *same* Realtek controller under Linux

Rather than testing with a different Bluetooth dongle, the built-in Realtek USB controller was detached from Windows and attached to Ubuntu in WSL2 using `usbipd`. This kept the physical Bluetooth hardware constant while changing the host Bluetooth stack and exposing BlueZ PHY controls.

**Scope:** Stock WSL2 installations may lack the necessary Bluetooth USB kernel drivers; working passthrough/modules were established before this test. The Windows adapter must be detached from WSL2 and returned to Windows after testing.

### 3. Restrict allowed PHY to LE 1M

```bash
sudo btmgmt --index 0 phy BR1M1SLOT LE1MTX LE1MRX
```

**Observed:** Keychron paired and connected (`Bonded: yes`, `Connected: yes`). This demonstrated that the adapter, host and keyboard could complete a BLE connection under the LE 1M restriction.

### 4. Allow LE 2M on the same hardware

```bash
sudo btmgmt --index 0 phy BR1M1SLOT LE1MTX LE1MRX LE2MTX LE2MRX
```

**Observed:** Pairing failed again. HCI tracing showed a transition to LE 2M followed roughly **0.48 s** later by disconnect reason **`0x08` (Connection Timeout)**. The failure was reproduced in repeated tests.

**Inference:** PHY change/negotiation was the strongest trigger. The observation did **not** identify whether the initiating host, Realtek controller firmware, or Keychron's controller/firmware violated an expected behaviour. The HCI trace was not committed to this repository; the figures above come from the recorded experiment.

### 5. Change the smallest possible firmware setting

Since no validated Windows setting forced LE 1M on this adapter, we investigated the firmware's automatic PHY switching behaviour in the Keychron Realtek-enabled ZMK fork. The selected intervention was:

```ini
CONFIG_BT_AUTO_PHY_UPDATE=n
```

The tested build was compared against a baseline. **The only reported Kconfig difference** in `app/build/zephyr/.config` was:

```diff
-CONFIG_BT_AUTO_PHY_UPDATE=y
+# CONFIG_BT_AUTO_PHY_UPDATE is not set
```

**Important distinction:** This disables ZMK/Zephyr's automatic PHY update behaviour; it is **not** equivalent to removing LE 2M PHY support from the Bluetooth controller. No proof exists that post-fix connections are always on LE 1M.

### 6. Build and flash the modified firmware

The image was built for board `rtl8762gtu_kb` / shield `keychron_k1_ultra_ansi` from the exact source revision above, in the Keychron-specific west workspace. The fork's fixed-path post-processing required using `app/build` sequentially for baseline and modified builds. In that source checkout, a local declaration for missing Kconfig symbol `RTK_DFU` was needed; that is a build-specific adjustment rather than a general recommendation.

| Tested image characteristic | Value |
| --- | --- |
| Post-processed file | `app/build/zephyr/zmk.bin` |
| Image size | **305,628 bytes** |
| SHA-256 | `8525e8b6570478b1618d04b58501432d0e8e022b79cb864de8938aae9490fd54` |

The image was flashed on Windows with `Realtek_HID_flasher.exe` over the Realtek DFU HID interface (usage page `0x008C`). Recorded success indicators included device-side **CRC verified OK** and `IMAGE_SWITCH` being sent. A **post-flash** read-only handshake returned:

```text
>> GET_MODEL_INFO (0x60)...
   model = "KCZKK18K"   fw = "1.0.1-r"
   dfu_version = 0x0001 enc_mode = 0
   identity VALID (Keychron K1 Ultra 8K - ANSI)
```

**Unknown:** The complete sequence used to expose the HID interface *before* flashing was not preserved. There is no recorded use of a physical reset button under the spacebar, but its non-use cannot be proven from the available command log. Do not present a spacebar-button sequence as a verified prerequisite.

### 7. Verify outcome

**Observed:** After the new firmware started, Bluetooth pairing succeeded with the previously failing Windows/Realtek combination, and the keyboard subsequently worked over Bluetooth with multiple hosts. This is a successful single-unit intervention, not independent validation.

## Evidence versus hypothesis

| Statement | Assessment |
| --- | --- |
| LE 1M restriction on the same Realtek adapter allowed pairing under Ubuntu | **Observed** |
| Allowing LE 2M was followed by a PHY transition and connection timeout `0x08` | **Observed** |
| Firmware setting `CONFIG_BT_AUTO_PHY_UPDATE=n` resolved Windows pairing in this test | **Observed** |
| LE 2M operation/negotiation is implicated | **Strong inference** |
| The Realtek driver alone is defective | **Not established** |
| Keychron permanently operates at LE 1M after flashing | **Not established** (no post-fix PHY capture) |
| Every K1 Ultra 8K revision will behave the same | **Not established** |

## Open verification questions

- Obtain a post-fix HCI trace to see whether connections remain LE 1M or whether LE 2M still occurs successfully.
- Repeat the test with other Bluetooth chipsets and Keychron hardware revisions before claiming a universal fix.
- Preserve before/after `btmon` logs and exact toolchain versions if repeating the experiment.
- Establish a tested recovery procedure before recommending custom flashing to less-experienced owners.

## Sources and reproducibility anchors

- [Step-by-step reproduction guide](REPRODUCE.md)
- [Pinned Keychron ZMK commit](https://github.com/Keychron/zmk/commit/7f8d16732a3782e237915bff30a1f67de09268b7)
- [Keychron Zephyr commit observed in the workspace](https://github.com/Keychron/zephyr/commit/2c11634850f7d6b04e55a253d4506a6e7c8bd673)
- [Keychron Realtek HAL commit observed in the workspace](https://github.com/Keychron/hal_realtek/commit/3c8e0bf787d80bc98c0fe4041f660ee1bf091206)
- [Third-party Realtek HID flasher source](https://github.com/komax20000-cloud/KeychronK1Ultra8K/tree/main/tools/flasher)
