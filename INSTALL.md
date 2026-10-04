# LineageOS NFC Fix Patches

## SO-01J (Xperia XZ3 Docomo) FeliCa NFC Fix for LineageOS 22.2

### Problem

LineageOS 22.2 on Sony Xperia XZ3 SO-01J shows:
- NFC kernel recognition but I/O errors
- Missing NFC toggle in settings
- NFC functionality disabled

### Root Cause

SO-01J (Docomo variant) uses **Sony CXD224x FeliCa** NFC chip, but LineageOS was built with generic akatsuki config targeting **NXP PN553**.

**Hardware differences:**

| Component | Docomo (SO-01J) | Generic |
|-----------|-----------------|----------|
| NFC Chip | Sony CXD224x | NXP PN553 |
| I2C Address | 0x29 | 0x28 |
| GPIO 63 | Pull-up | Pull-down |
| LDO | Rohm BD7602 | N/A |
| DTV Tuner | MN88553 | N/A |
| Config | CONFIG_MACH_SONY_AKATSUKI_DCM=y | CONFIG_MACH_SONY_AKATSUKI=y |

### Solution

Use `tama_akatsuki_dcm_defconfig` kernel configuration which includes:
- Proper CXD224x FeliCa device tree
- GPIO 63 pull-up for FeliCa interrupt
- Rohm BD7602 LDO support
- DTV tuner configuration

### Application

#### Prerequisites

1. LineageOS 22.2 source (lineage-22.2 branch)
2. Kernel: `android_kernel_sony_sdm845` with lineage-22.2 branch
3. Device: `android_device_sony_akatsuki` with lineage-22.2 branch

#### Step 1: Apply Patch

```bash
cd device/sony/akatsuki
git apply path/to/0001-akatsuki-Fix-NFC-FeliCa-support-for-SO-01J-DCM.patch
```

Or manually edit `BoardConfig.mk`:

```makefile
# Before
TARGET_KERNEL_CONFIG := tama_akatsuki_defconfig

# After
TARGET_KERNEL_CONFIG := tama_akatsuki_dcm_defconfig
```

#### Step 2: Verify Kernel Support

Confirm that `android_kernel_sony_sdm845` has DCM config support:

```bash
ls -la android_kernel_sony_sdm845/arch/arm64/configs/diffconfig/akatsuki_dcm_diffconfig
ls -la android_kernel_sony_sdm845/arch/arm64/boot/dts/somc/sdm845-tama-akatsuki_dcm*
```

If these files don't exist, update kernel to lineage-22.2 branch:

```bash
cd android_kernel_sony_sdm845
git checkout lineage-22.2
git pull
```

#### Step 3: Build

```bash
cd lineage
. build/envsetup.sh
lunch lineage_akatsuki-userdebug
mka bootimage
mka systemimage
```

#### Step 4: Flash

```bash
adb reboot bootloader
fastboot flash boot boot.img
fastboot flash system system.img
fastboot reboot
```

### Verification

After applying patch and rebuilding:

```bash
# Check device tree loaded
adb shell getprop ro.board.platform
# Should show: somc,akatsuki-dcm

# Check kernel config
adb shell cat /proc/config.gz | gunzip | grep MACH_SONY_AKATSUKI
# Should show: CONFIG_MACH_SONY_AKATSUKI_DCM=y

# Check FeliCa driver
adb shell dmesg | grep -i "cxd224\|felica"

# Check NFC HAL
adb shell lshal | grep nfc

# Check NFC logs
adb shell logcat | grep -i nfc
```

Expected result:
- NFC toggle appears in Settings > Connected devices > NFC
- No I/O errors in dmesg
- FeliCa can be used for payments (Suica, etc.)

### Technical Details

**Device Tree Configuration (somc-tama-nfc_carillon.dtsi):**

```dts
&qupv3_se10_i2c {
    felica_ldo@1e {
        compatible = "rohm,bd7602";
        reg = <0x1e>;
    };
    felica@29 {
        compatible = "sony,cxd224x-i2c";
        reg = <0x29>;
        interrupt-parent = <&tlmm>;
        interrupts = <63 0x2002>;
        sony,nfc_int = <&tlmm 63 0>;
        sony,nfc_wake = <&tlmm 62 0>;
    };
};
```

**GPIO Configuration (sdm845-tama-akatsuki_jp-common.dtsi):**

```dts
/* GPIO_63: FELICA_INT_N */
&sdm_gpio_63 {
    mux { pins = "gpio63"; function = "gpio"; };
    config {
        pins = "gpio63";
        drive-strength = <2>;
        bias-pull-up;      /* Critical for FeliCa */
        input-enable;
    };
};
```

### Troubleshooting

1. **NFC toggle still missing**
   - Verify kernel was rebuilt with new config
   - Check `adb shell cat /proc/config.gz | gunzip | grep DCM`
   - Ensure `android_kernel_sony_sdm845` lineage-22.2 branch used

2. **I/O errors persist**
   - Check dmesg for missing FeliCa driver
   - Verify device tree overlay loaded: `adb shell cat /proc/device-tree/qup-se10-i2c/felica@29/compatible`
   - Check GPIO 62/63 configuration

3. **FeliCa not working**
   - Verify Rohm BD7602 LDO loaded
   - Check interrupt configuration
   - Ensure NFC HAL has CXD224x support

### References

- LineageOS Kernel: https://github.com/LineageOS/android_kernel_sony_sdm845
- Device Tree: arch/arm64/boot/dts/somc/sdm845-tama-akatsuki_dcm.dtsi
- Config: arch/arm64/configs/diffconfig/akatsuki_dcm_diffconfig
- NFC Config: device/sony/akatsuki/nfc/libnfc-nxp.conf

### License

Apache License 2.0 (matching LineageOS)
