# Device Tree & Overlay — Complete Interview Questions Guide

## Difficulty Levels

```
┌─────────────────────────────────────────────────────────────┐
│  🟢 BEGINNER      - Fresher / 0-1 years                     │
│  🟡 INTERMEDIATE  - 1-3 years experience                    │
│  🔴 ADVANCED      - 3+ years / Senior Engineer              │
│  🟣 EXPERT        - Lead / Architect level                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Section 1: Basic Concept Questions 🟢

### Q1. What is a Device Tree? Why do we need it?

Device Tree is a **data structure** that describes the hardware components of a system to the OS kernel.

**PROBLEM (Before Device Tree):**
```
┌────────────────────────────────────────────┐
│  Each board needed its OWN kernel          │
│  Hardware info was HARDCODED in kernel     │
│  10 boards = 10 different kernels          │
│  Any hardware change = recompile kernel    │
└────────────────────────────────────────────┘
```

**SOLUTION (With Device Tree):**
```
┌────────────────────────────────────────────┐
│  ONE generic kernel                        │
│  Hardware described in SEPARATE .dtb file  │
│  10 boards = 1 kernel + 10 .dtb files     │
│  Hardware change = only change .dtb        │
└────────────────────────────────────────────┘
```

**Key points to mention:**
- Separates hardware description from kernel code
- One kernel can boot multiple hardware variants
- `.dts` (source) compiled to `.dtb` (binary)
- Bootloader passes DTB to kernel at boot
- Kernel reads DTB to know what hardware exists

---

### Q2. What is the difference between `.dts`, `.dtsi`, and `.dtb`?

```
┌──────────┬──────────────────────────────────────────────────┐
│ .dts     │ Device Tree SOURCE                               │
│          │ • Human-readable text file                       │
│          │ • Top-level file for a specific board            │
│          │ • Contains #include of .dtsi files              │
│          │ • Example: board-myphone.dts                     │
├──────────┼──────────────────────────────────────────────────┤
│ .dtsi    │ Device Tree SOURCE INCLUDE                       │
│          │ • Like a header file in C                        │
│          │ • Contains reusable SoC/common definitions        │
│          │ • Included by multiple .dts files                │
│          │ • Example: soc-snapdragon.dtsi                   │
├──────────┼──────────────────────────────────────────────────┤
│ .dtb     │ Device Tree BINARY (BLOB)                        │
│          │ • Compiled binary of .dts                        │
│          │ • What bootloader loads and passes to kernel     │
│          │ • Not human-readable                             │
│          │ • Example: board-myphone.dtb                     │
└──────────┴──────────────────────────────────────────────────┘
```

**Analogy:** `.dtsi` = header file (`.h`), `.dts` = C source (`.c`), `.dtb` = compiled binary

---

### Q3. What is DTC? What command do you use to compile a DT?

DTC = **Device Tree Compiler** — converts `.dts` (text) to `.dtb` (binary) and vice versa.

```bash
# Compile DTS → DTB
dtc -I dts -O dtb -o output.dtb input.dts

# Decompile DTB → DTS (for debugging)
dtc -I dtb -O dts -o output.dts input.dtb

# Compile with symbol preservation (REQUIRED for overlays)
dtc -@ -I dts -O dtb -o output.dtb input.dts
```

**Flags:**
- `-I` = Input format (`dts` or `dtb`)
- `-O` = Output format (`dts` or `dtb`)
- `-o` = Output file
- `-@` = Generate `__symbols__` node (critical for overlays — without it, overlay cannot find nodes to modify)

---

### Q4. What is a Device Tree Node? What are Properties?

**NODE:** Represents a hardware device or logical component.

```dts
uart0: serial@7e201000 {   // Node
//  │      │      │
//  │      │      └── unit-address (= first reg value)
//  │      └───────── node-name
//  └──────────────── label (for referencing: &uart0)

    compatible = "arm,pl011";    // String property
    reg = <0x7e201000 0x200>;    // Integer property
    interrupts = <2 25>;         // Integer array
    status = "okay";             // String property
    read-only;                   // Boolean (presence = true)
};
```

**Property Types:**
```
┌──────────────┬──────────────────────────────────────┐
│ String       │ compatible = "vendor,device";        │
├──────────────┼──────────────────────────────────────┤
│ String list  │ compatible = "v2", "v1";             │
├──────────────┼──────────────────────────────────────┤
│ Integer(u32) │ reg = <0x1000 0x100>;                │
├──────────────┼──────────────────────────────────────┤
│ Integer(u64) │ size = /bits/ 64 <0x100000000>;      │
├──────────────┼──────────────────────────────────────┤
│ Boolean      │ read-only;  (just presence)          │
├──────────────┼──────────────────────────────────────┤
│ Byte array   │ mac-addr = [00 11 22 33 44 55];      │
├──────────────┼──────────────────────────────────────┤
│ Phandle ref  │ interrupt-parent = <&gic>;           │
└──────────────┴──────────────────────────────────────┘
```

---

### Q5. What is the `compatible` property? How does driver matching work?

`compatible` is the **most important** DT property — it tells the kernel which driver should handle this device.

**Format:** `"vendor,device-name"` (vendor = manufacturer lowercase, device = chip name)

```dts
sensor@68 {
    compatible = "invensense,mpu6050",   // Most specific first
                 "invensense,mpu6000";   // Fallback
};
```

**Driver matching flow:**
```
DT Node: compatible = "invensense,mpu6050"
                │
                ▼
Kernel searches all registered drivers
                │
                ▼
Driver's of_match_table:
{ .compatible = "invensense,mpu6050" }  ← MATCH!
                │
                ▼
driver->probe() called
```

```c
static const struct of_device_id mpu6050_ids[] = {
    { .compatible = "invensense,mpu6050" },
    { .compatible = "invensense,mpu6000" },  // fallback
    { }  // sentinel
};
```

> **Key point:** Multiple compatible strings = fallback chain. Kernel tries first string; if no driver found, tries second. Most specific version first, generic last.

---

### Q6. What is `status` property? What values can it have?

```
┌────────────┬────────────────────────────────────────────────┐
│ "okay"     │ Device is enabled, driver will probe it        │
├────────────┼────────────────────────────────────────────────┤
│ "disabled" │ Device exists but is disabled                  │
│            │ Driver will NOT probe it                       │
├────────────┼────────────────────────────────────────────────┤
│ "reserved" │ Device reserved for other use (e.g. firmware)  │
├────────────┼────────────────────────────────────────────────┤
│ "fail"     │ Device detected serious error                  │
└────────────┴────────────────────────────────────────────────┘
```

**Common pattern:**

```dts
// soc.dtsi — disabled by default
i2c0: i2c@15000000 {
    compatible = "example,i2c";
    reg = <0x15000000 0x1000>;
    status = "disabled";    // OFF in SoC template
};

// board.dts — enable for this board
&i2c0 {
    status = "okay";        // Enable on this specific board
};
```

SoC has many peripherals; not all boards use all of them. Default disabled = safe, board enables what it needs.

---

## Section 2: Device Tree Overlay Questions 🟡

### Q7. What is a Device Tree Overlay (DTBO)? Why is it needed?

DTBO = **Device Tree Binary Overlay** — a binary patch that modifies the base DTB without changing its source.

**Supply chain model:**
```
SoC Vendor (Qualcomm/MediaTek)
    └── Provides: base.dtb  (SoC-level hardware)
                    │
                    ▼
ODM (Board Manufacturer)
    └── Adds: odm.dtbo  (camera, display, touch)
              WITHOUT modifying SoC DTB source
                    │
                    ▼
OEM (Phone Brand)
    └── Adds: oem.dtbo  (LEDs, buttons, NFC)
              WITHOUT modifying ODM or SoC sources
                    │
                    ▼
Bootloader merges: base.dtb + odm.dtbo + oem.dtbo
= Final merged DTB passed to kernel
```

**Benefits:** No source sharing between layers, each layer independently updatable, supports Project Treble separation, OTA updates can update overlays independently.

```dts
/dts-v1/;
/plugin/;   // ← This line makes it an overlay!
```

---

### Q8. What is the difference between `target` and `target-path` in overlays?

```
┌────────────────┬─────────────────────────────────────────────┐
│  target        │ References node by PHANDLE (label)          │
│                │ Resolved at compile time                    │
│                │ Requires -@ flag when compiling base DTB    │
│                │ Faster — no string search needed            │
├────────────────┼─────────────────────────────────────────────┤
│  target-path   │ References node by PATH STRING              │
│                │ Resolved at runtime                         │
│                │ Works without -@ flag on base DTB           │
│                │ More portable but slower                    │
└────────────────┴─────────────────────────────────────────────┘
```

```dts
// Using target (phandle) — preferred
fragment@0 {
    target = <&uart0>;
    __overlay__ { status = "okay"; };
};

// Using target-path (string path)
fragment@0 {
    target-path = "/soc/serial@7e201000";
    __overlay__ { status = "okay"; };
};

// Modern shorthand (equivalent to target phandle)
&uart0 {
    status = "okay";
};
```

> **⚠️ Tricky:** `target` requires base DTB compiled with `-@`. If base DTB has no symbols, use `target-path`.

---

### Q9. Explain the DTBO partition format and how bootloader selects overlays.

```
┌──────────────────────────────────────────────────┐
│  DTBO Header                                     │
│  ├── magic: 0xd7b7ab1e                           │
│  ├── total_size                                  │
│  ├── dt_entry_count                              │
│  └── dt_entry_size                               │
├──────────────────────────────────────────────────┤
│  Entry Table                                     │
│  ├── Entry[0]: { size, offset, id, rev,          │
│  │              custom[0..3] }                   │
│  └── Entry[n]: ...                               │
├──────────────────────────────────────────────────┤
│  DTBO Data                                       │
│  ├── overlay_0.dtbo (raw binary)                 │
│  └── overlay_n.dtbo                              │
└──────────────────────────────────────────────────┘
```

**Bootloader selection logic:**
1. Read `board_id` from hardware (GPIO strapping, eFuse, SMEM, I2C EEPROM)
2. Read `board_revision` from hardware
3. Scan DTBO entry table — apply all entries where `entry.id == board_id && entry.rev == board_rev`

```bash
mkdtboimg.py create dtbo.img \
    camera-v1.dtbo \
    camera-v2.dtbo \
    display.dtbo \
    audio.dtbo
```

---

### Q10. How does an overlay MODIFY an existing property vs ADD a new node?

```dts
// MODIFY: redefine the property — new value REPLACES old
&i2c0 {
    clock-frequency = <400000>;  // Overrides old value (100000)
    status = "okay";
};

// ADD new node to existing parent
&i2c0 {
    #address-cells = <1>;
    #size-cells = <0>;

    sensor@68 {                  // NEW node added to i2c0
        compatible = "invensense,mpu6050";
        reg = <0x68>;
    };
};

// ADD new node to root
/ {
    fragment@0 {
        target-path = "/";
        __overlay__ {
            my-new-device {
                compatible = "vendor,mydevice";
                status = "okay";
            };
        };
    };
};

// DELETE a property
&uart0 {
    /delete-property/ current-speed;
};
```

**Important merge rules:**
- **Properties → OVERWRITE** (new value wins)
- **Sub-nodes → MERGE** (overlay adds to existing)
- **New nodes → APPEND** to parent

---

## Section 3: Boot Process Questions 🟡

### Q11. Walk me through how Device Tree is used during Android boot.

**Stage 1 — Bootloader:**
1. Load base DTB from `dtb`/`vendor_boot` partition
2. Load `dtbo.img` from `dtbo` partition
3. AVB verify signatures
4. Read hardware `board_id` (GPIO/eFuse)
5. Apply matching overlays using `libfdt`
6. Dynamically modify DT: RAM size, bootargs, initrd location, KASLR seed
7. Place merged DTB in DRAM
8. Jump to kernel, pass DTB address in `x3` register

**Stage 2 — Kernel:**
1. `early_init_dt_scan()` — read `/chosen/bootargs` + `/memory`
2. `unflatten_device_tree()` — flat binary → linked tree in memory
3. `irqchip_init()` — find GIC node, init interrupt controller
4. `of_clk_init()` — build clock tree from DT
5. `of_platform_populate()` — walk DT, create `platform_device` for each node
6. Driver matching & probing — match `compatible` → call `probe()`
7. Expose DT via `/proc/device-tree/`

**Stage 3 — Init Process:**
1. Read `/proc/device-tree/chosen/bootargs` → extract `androidboot.*` → set `ro.boot.*` properties
2. Read `/proc/device-tree/firmware/android/fstab` → mount partitions
3. Start services based on hardware properties

**Stage 4 — HAL Layer:**
- Uses `/dev/*` nodes created by DT-probed drivers
- May directly read `/proc/device-tree/` for extra config

---

### Q12. How does bootloader pass Device Tree to the kernel?

**ARM64 kernel entry registers:**
```
┌─────────┬────────────────────────────────────────────┐
│  x0     │ = 0 (reserved)                             │
│  x1     │ = 0 (reserved)                             │
│  x2     │ = 0 (reserved)                             │
│  x3     │ = PHYSICAL ADDRESS of DTB in RAM  ← KEY   │
└─────────┴────────────────────────────────────────────┘
```

```
Memory layout:
┌──────────────────────────────────────┐
│  0x80000000  Kernel Image            │
│  ...                                 │
│  0x81F00000  DTB  ← bootloader here │
│  0x82000000  Ramdisk                 │
└──────────────────────────────────────┘
```

Kernel validates DTB: `magic = 0xd00dfeed`. On ARM32 the register is `r2`, not `x3`.

---

### Q13. What properties does the bootloader typically ADD or MODIFY in DT?

**`/chosen` node:**
```
┌─────────────────────────────────────────────────────────────┐
│  bootargs              │ Serial#, verified boot state, slot  │
│  linux,initrd-start    │ Ramdisk DRAM location               │
│  linux,initrd-end      │                                     │
│  kaslr-seed            │ Random seed — must change each boot │
│  android,serialno      │ Device serial number                │
│  android,verifiedboot  │ green / yellow / orange / red       │
└─────────────────────────────────────────────────────────────┘
```

**`/memory` node:** actual detected DRAM size (3GB device ≠ 6GB device).

**`/reserved-memory`:** framebuffer address, TrustZone region, ramoops buffer location.

**Example bootargs added by bootloader:**
```
console=ttyMSM0,115200n8
androidboot.hardware=myphone
androidboot.serialno=HT7251A01234
androidboot.verifiedbootstate=green
androidboot.slot_suffix=_a
mem=6144M
```

---

## Section 4: Kernel & Driver Questions 🔴

### Q14. How does the Linux kernel match a DT node to a driver?

```c
// Driver registration
static const struct of_device_id my_ids[] = {
    { .compatible = "vendor,chip-v2", .data = &chip_v2_config },
    { .compatible = "vendor,chip-v1", .data = &chip_v1_config },
    { }  // MUST end with empty entry
};
MODULE_DEVICE_TABLE(of, my_ids);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name           = "my-driver",
        .of_match_table = my_ids,
    },
};
module_platform_driver(my_driver);

// In probe function
static int my_probe(struct platform_device *pdev) {
    struct device_node *np = pdev->dev.of_node;

    // Which compatible string matched?
    const struct of_device_id *match = of_match_device(my_ids, &pdev->dev);
    struct chip_config *cfg = match->data;

    // Read register base
    struct resource *res = platform_get_resource(pdev, IORESOURCE_MEM, 0);

    // Read interrupt
    int irq = platform_get_irq(pdev, 0);

    // Read custom property
    u32 speed;
    of_property_read_u32(np, "clock-frequency", &speed);

    return 0;
}
```

> **Important:** `status = "disabled"` nodes are **SKIPPED** — `of_device_is_available()` returns false, so no `platform_device` is created.

---

### Q15. What is deferred probing in Linux kernel? How does DT relate to it?

**Problem:** Display driver needs clock, regulator, and GPIO — all of which must be probed first. What if display probes before the clock driver?

**Solution:** Return `-EPROBE_DEFER` from `probe()`. Kernel marks device as deferred and retries after each successful probe.

```c
static int display_probe(struct platform_device *pdev) {
    struct clk *clk = devm_clk_get(&pdev->dev, "pixel_clk");
    if (IS_ERR(clk))
        return -EPROBE_DEFER;  // Clock not ready yet
    // ...
}
```

**DT relationship:** DT defines the references (phandles) between devices — these create the dependency chain.

```dts
display@18000000 {
    clocks     = <&clock_ctrl 40>;    // Dependency on clock_ctrl
    vdd-supply = <&vcc_1v8>;          // Dependency on regulator
    reset-gpios = <&gpio0 17 0>;      // Dependency on GPIO
};
```

**Android 14+:** Lazy probing defers non-critical devices to late boot → faster boot time.

---

### Q16. Explain `#address-cells` and `#size-cells` in Device Tree.

These tell how many 32-bit cells represent **address** and **size** in child nodes' `reg` property.

```dts
// 32-bit addressing (most SoC peripherals)
soc {
    #address-cells = <1>;  // 1 cell = 32-bit address
    #size-cells = <1>;     // 1 cell = 32-bit size

    uart@12000 {
        reg = <0x12000 0x100>;
        //     │       └── size (0x100 bytes)
        //     └────────── address (0x12000)
    };
};

// 64-bit addressing (root node, modern SoCs)
/ {
    #address-cells = <2>;  // 2 cells = 64-bit address
    #size-cells = <2>;     // 2 cells = 64-bit size

    memory@80000000 {
        reg = <0x0 0x80000000  0x0 0x40000000>;
        // address: 0x0_80000000 (2GB mark)
        // size:    0x0_40000000 (1GB)
    };
};

// I2C bus — address only, no size
i2c0 {
    #address-cells = <1>;
    #size-cells = <0>;    // No size for I2C devices

    sensor@68 {
        reg = <0x68>;     // Just I2C address
    };
};
```

> **Tricky follow-up:** `#size-cells = <0>` means no size field in `reg` — used for I2C devices and SPI chip-selects.

---

### Q17. What is a phandle? How are device references made in DT?

A **phandle** is a unique integer auto-assigned to each DT node, used to reference one node from another.

```dts
// Provider declares its cell count
clock_ctrl: clock-controller@11000000 {
    compatible = "example,clk";
    #clock-cells = <1>;   // Reference needs 1 extra arg
};

// Consumer references by label (& syntax)
uart0: serial@14000000 {
    clocks = <&clock_ctrl 10>;
    //        │            └── clock ID (the 1 extra cell)
    //        └────────────── phandle to clock_ctrl
};
```

**Common reference patterns:**
```
┌─────────────────┬──────────────────────────────────────────┐
│  interrupts     │ interrupt-parent = <&gic>;               │
│                 │ interrupts = <0 33 4>;                   │
├─────────────────┼──────────────────────────────────────────┤
│  clocks         │ clocks = <&clk 10>;                      │
│                 │ clock-names = "uart_clk";                │
├─────────────────┼──────────────────────────────────────────┤
│  regulators     │ vdd-supply = <&vcc_3v3>;                 │
├─────────────────┼──────────────────────────────────────────┤
│  GPIOs          │ reset-gpios = <&gpio0 17 0>;             │
├─────────────────┼──────────────────────────────────────────┤
│  pinctrl        │ pinctrl-0 = <&uart_pins>;                │
└─────────────────┴──────────────────────────────────────────┘
```

**Cells convention:** `#clock-cells = <1>` → `<&clk clock_id>`, `#gpio-cells = <2>` → `<&gpio pin flags>`

---

## Section 5: AOSP-Specific Questions 🔴

### Q18. Where is the DTB located in Android 11 vs Android 12? Why did it change?

**Android 11 and before:** Separate `dtb` partition.
```
boot (kernel) | dtb (base DTB) | dtbo (overlays)
```

**Android 12 and after:** DTB embedded inside `vendor_boot`.
```
boot (kernel) | vendor_boot (ramdisk + BASE DTB) | dtbo (overlays)
```

```makefile
# OLD
BOARD_PREBUILT_DTBIMAGE_DIR := device/vendor/dtb

# NEW
BOARD_MOVE_DTB_TO_VENDOR_BOOT := true
BOARD_BOOT_HEADER_VERSION     := 4
```

**Why the change:**
1. **Treble compliance** — DTB is vendor-specific data; it belongs in the vendor partition
2. **Atomic updates** — `vendor_boot` and DTB update together; prevents version mismatch
3. **Partition simplification** — removes separate `dtb` partition
4. **AVB** — single signature covers both ramdisk and DTB

---

### Q19. What is the relationship between DT and Android Treble?

```
┌─────────────────────────────────┐
│     AOSP / Google               │
│     system, product partitions  │  ← No DT access
└─────────────────────────────────┘
               ▲ VINTF interface
┌─────────────────────────────────┐
│     Vendor                      │
│     HALs, vendor modules        │  ← Uses DT-probed drivers
│     vendor_boot (DTB inside)    │  ← DT lives here
└─────────────────────────────────┘
┌─────────────────────────────────┐
│     ODM                         │
│     Board-specific HALs         │  ← DTBO lives here
│     dtbo (overlays)             │
└─────────────────────────────────┘
```

**How DT enables Treble:** Each layer (SoC vendor, ODM, OEM) provides its own DT artifact independently, without sharing source code. All layers are independently OTA-updateable.

**VTS mandatory DT nodes:**
```dts
/firmware/android { compatible = "android,firmware"; }
/firmware/android/fstab { ... }
/chosen { android boot properties }
```

---

### Q20. What is VTS DT Verification? What does it test?

VTS = **Vendor Test Suite** — Google's Treble compliance tests that must pass before a device can ship with Google apps.

```
VtsDtboVerification      — DTBO magic, header format, entry validity
VtsKernelDtTest          — /proc/device-tree/ accessible, required nodes present
VtsVendorBootDtbTest     — DTB in vendor_boot, header version correct (Android 12+)
VtsGkiCompliance         — GKI compliance, no non-upstream nodes (Android 12+)
VtsDtSchemaValidation    — YAML schema conformance (Android 13+)
VtsDtLazyProbeTest       — Lazy probe behavior (Android 14+)
VtsAcfValidation         — ACF validation (Android 15)
```

```bash
vts-tradefed run vts -m VtsDtboVerification
vts-tradefed run vts -m VtsKernelDtTest
```

**Common VTS failures:** missing `/firmware/android` node, wrong DTBO magic, required property absent, DTB not in `vendor_boot` (Android 12+).

---

## Section 6: Advanced / Tricky Questions 🟣

### Q21. What happens if two overlays modify the SAME property? Which wins?

**Rule: LAST APPLIED OVERLAY WINS.**

```
Base DTB:      current-speed = <115200>
Overlay 1:     current-speed = <921600>   ← applied first
Overlay 2:     current-speed = <460800>   ← applied second → WINS

RESULT:        current-speed = <460800>
```

**For nodes vs properties:**
```
Properties → OVERWRITE  (last value replaces previous)
Nodes      → MERGE      (overlay adds to/merges with existing node)
```

> **⚠️ Tricky edge case:** If an overlay adds a node that already exists in the base, it **merges** with the existing node — it does NOT create a duplicate.

---

### Q22. How do you debug a device that is not probing? What DT steps do you take?

```bash
# STEP 1: Verify DT node exists
adb shell ls /proc/device-tree/soc/

# STEP 2: Check if node is enabled
adb shell hexdump -C /proc/device-tree/soc/uart@14000000/status
# "6f6b6179 00" = "okay\0" = enabled

# STEP 3: Check driver is registered
adb shell cat /sys/bus/platform/drivers/

# STEP 4: Check device created on bus
adb shell ls /sys/bus/platform/devices/

# STEP 5: Search dmesg for errors
adb shell dmesg | grep -E "probe|defer|error"

# STEP 6: Verify compatible string exactly
adb shell cat /proc/device-tree/soc/uart@14000000/compatible | tr '\0' '\n'

# STEP 7: Decompile live DT
adb shell cat /sys/firmware/fdt > fdt.bin
dtc -I dtb -O dts -o live.dts fdt.bin
grep -A 20 "uart@14000000" live.dts

# STEP 8: Verify overlay applied correctly
fdtoverlay -i base.dtb overlay.dtbo -o merged.dtb
```

**Common root causes:**
```
┌────────────────────────────────────────────────────────────┐
│  status = "disabled"        │ DTBO not applied?            │
│  Wrong compatible string    │ Typo in .dts or driver       │
│  Missing required property  │ Driver returns -EINVAL       │
│  Dependency not ready       │ Returns -EPROBE_DEFER        │
│  Wrong reg address          │ ioremap fails                │
│  Wrong interrupt number     │ request_irq fails            │
│  Clock not available        │ devm_clk_get fails           │
│  GPIO conflict              │ gpio_request fails           │
└────────────────────────────────────────────────────────────┘
```

---

### Q23. What is the difference between `interrupts` and `interrupt-parent`?

- **`interrupt-parent`** — WHICH interrupt controller handles this device (phandle)
- **`interrupts`** — WHICH specific IRQ line within that controller

```dts
// GIC interrupt controller
gic: interrupt-controller@10000000 {
    interrupt-controller;
    #interrupt-cells = <3>;      // Need 3 values per interrupt
};

// GPIO (also an interrupt controller)
gpio0: gpio@13000000 {
    interrupt-controller;
    #interrupt-cells = <2>;      // Need 2 values per interrupt
    interrupt-parent = <&gic>;
    interrupts = <0 32 4>;       // GPIO's own IRQ to GIC
};

// Sensor uses GPIO as interrupt controller
sensor@68 {
    interrupt-parent = <&gpio0>;
    interrupts = <5 2>;          // GPIO pin 5, falling edge
    //           │  └── trigger type
    //           └───── GPIO pin number
};
```

**Inheritance:** `interrupt-parent` set at a parent node is inherited by all children unless overridden.

---

### Q24. What is `of_property_read_u32` vs `of_property_read_u32_array`?

```c
// DT:
// clock-frequency = <400000>;
// fifo-sizes = <64 128 256>;
// device-name = "my-sensor";
// dual-channel;

static int my_probe(struct platform_device *pdev) {
    struct device_node *np = pdev->dev.of_node;

    // Single u32
    u32 clk_freq;
    of_property_read_u32(np, "clock-frequency", &clk_freq);

    // u32 array
    u32 fifo_sizes[3];
    of_property_read_u32_array(np, "fifo-sizes", fifo_sizes, 3);
    // fifo_sizes[0]=64, [1]=128, [2]=256

    // String
    const char *name;
    of_property_read_string(np, "device-name", &name);

    // Boolean (presence check)
    bool dual = of_property_read_bool(np, "dual-channel");

    // GPIO
    int gpio = of_get_named_gpio(np, "power-gpios", 0);

    // u64
    u64 big_value;
    of_property_read_u64(np, "big-reg", &big_value);

    return 0;
}
```

**Return values:** `0` = success, `-EINVAL` = property not found, `-EOVERFLOW` = array too small, `-ENODATA` = property exists but empty.

---

### Q25. How does pinctrl work with Device Tree?

**PINCTRL** manages physical GPIO pin multiplexing — one pin can be UART TX, GPIO, or I2C SDA; only one function active at a time.

```dts
// 1. Pin controller (in SoC dtsi)
pinctrl: pinctrl@12000000 {
    compatible = "example,soc-pinctrl";

    uart0_pins: uart0-pins {
        pins = "gpio0", "gpio1";
        function = "uart0";        // Mux to UART
        drive-strength = <8>;
        bias-pull-up;
    };

    uart0_sleep_pins: uart0-sleep-pins {
        pins = "gpio0", "gpio1";
        function = "gpio";         // Low-power sleep mode
        bias-pull-down;
    };
};

// 2. Peripheral references pinctrl groups
uart0: serial@14000000 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart0_pins>;       // Active state
    pinctrl-1 = <&uart0_sleep_pins>; // Sleep state
};
```

When UART driver probes, pinctrl subsystem configures pins 0 and 1 for UART function. On system suspend, pins switch to sleep state for lower power.

---

### Q26. What happens to DT nodes in `reserved-memory`? How does it work?

`reserved-memory` marks regions the kernel must **not** use for general allocation.

```dts
/ {
    reserved-memory {
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;

        // no-map: kernel cannot access at all
        secure_memory: memory@80000000 {
            reg = <0x0 0x80000000 0x0 0x200000>;
            no-map;
        };

        // Shared DMA pool
        dma_pool: memory@90000000 {
            compatible = "shared-dma-pool";
            reg = <0x0 0x90000000 0x0 0x4000000>;
            reusable;
            linux,cma-default;
        };

        // Crash log — survives reboot
        ramoops: memory@A0000000 {
            compatible = "ramoops";
            reg = <0x0 0xA0000000 0x0 0x100000>;
            record-size  = <0x40000>;
            console-size = <0x40000>;
            pmsg-size    = <0x20000>;
        };
    };
};

// Device references its pool
gpu@19000000   { memory-region = <&dma_pool>;   };
display@18000000 { memory-region = <&framebuffer>; };
```

Kernel parses `reserved-memory` during early boot and marks those regions in the memblock allocator — general allocators skip them entirely.

---

## Section 7: Practical / Hands-On Questions 🔴

### Q27. Write a DT node for an I2C sensor with interrupt and power supply.

BME280 pressure/temp sensor: I2C at `0x76`, interrupt on GPIO 25, 3.3V power, 200ms interval.

```dts
// Regulator
vdd_sensor: regulator-sensor {
    compatible = "regulator-fixed";
    regulator-name = "sensor-vdd";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    gpio = <&gpio0 10 0>;
    enable-active-high;
    startup-delay-us = <2000>;
};

// Enable I2C bus
&i2c1 {
    status = "okay";
    clock-frequency = <400000>;   // Fast mode
    #address-cells = <1>;
    #size-cells = <0>;

    bme280: pressure-sensor@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;

        interrupt-parent = <&gpio0>;
        interrupts = <25 IRQ_TYPE_EDGE_FALLING>;

        vddd-supply  = <&vdd_sensor>;
        vddio-supply = <&vcc_1v8>;

        bosch,sampling-frequency = <200>;   // 200ms

        // Rotation matrix — sensor mounted upside-down
        mount-matrix = "0", "-1", "0",
                       "-1", "0", "0",
                        "0", "0", "-1";
    };
};
```

**As a DTBO overlay:**

```dts
/dts-v1/;
/plugin/;

&i2c1 {
    #address-cells = <1>;
    #size-cells = <0>;

    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
        interrupt-parent = <&gpio0>;
        interrupts = <25 2>;           // 2 = falling edge
        vddd-supply = <&vdd_sensor>;
    };
};
```

---

### Q28. How do you verify if a DTBO was correctly applied to a device?

```bash
# Method 1: Check live DT
adb shell cat /sys/firmware/fdt > live_fdt.bin
dtc -I dtb -O dts -o live.dts live_fdt.bin
grep -A 20 "bme280@76" live.dts

# Or check single property
adb shell cat /proc/device-tree/soc/i2c@15001000/bme280@76/compatible

# Method 2: dmesg for driver probe
adb shell dmesg | grep bme280
# Should see: bme280 1-0076: BME280 probed successfully

# Method 3: sysfs device
adb shell ls /sys/bus/i2c/devices/
# Should list: 1-0076

# Method 4: Diff before and after overlay
dtc -I dtb -O dts -o before.dts base.dtb
fdtoverlay -i base.dtb -o merged.dtb sensor-overlay.dtbo
dtc -I dtb -O dts -o after.dts merged.dtb
diff before.dts after.dts

# Method 5: Inspect DTBO image
mkdtboimg.py dump dtbo.img
mkdtboimg.py dump dtbo.img -b extracted_   # Extract individual .dtbo files
dtc -I dtb -O dts extracted_0.dtbo        # View overlay content
```

---

## Section 8: Scenario-Based Questions 🟣

### Q29. Your camera is not working. Logs show "no such device". How do you use DT to diagnose?

```bash
# 1. Check DT node exists
adb shell ls /proc/device-tree/soc/i2c@15001000/
# Is camera@1a listed? If MISSING → DTBO not applied or wrong DT

# 2. Check node is enabled
adb shell hexdump -C /proc/device-tree/soc/i2c@15001000/camera@1a/status
# "6f6b6179 00" = "okay" = enabled

# 3. Check I2C bus enabled
adb shell hexdump -C /proc/device-tree/soc/i2c@15001000/status

# 4. Verify compatible string (exact match, case sensitive)
adb shell cat /proc/device-tree/soc/i2c@15001000/camera@1a/compatible | xxd

# 5. Check driver loaded
adb shell dmesg | grep -i "imx586\|sony\|camera"
# "probe failed" = DT found but probe error
# No output = driver not loaded or DT node missing

# 6. Check MCLK
adb shell dmesg | grep mclk
# "failed to enable mclk" = wrong clock ID in DT

# 7. Check GPIO conflicts
adb shell cat /sys/kernel/debug/gpio
```

**"no such device" usually means:** wrong I2C address in DT, wrong I2C bus, bus not enabled, or camera power not configured.

---

### Q30. How would you add support for a new sensor to AOSP using DT?

**Complete workflow — adding BMP390 pressure sensor:**

**Step 1: Write DT Binding Schema (Android 13+ required)**

```yaml
# kernel/Documentation/devicetree/bindings/iio/pressure/bmp390.yaml
%YAML 1.2
---
$id: http://devicetree.org/schemas/iio/pressure/bmp390.yaml#
title: Bosch BMP390 Pressure Sensor

properties:
  compatible:
    const: bosch,bmp390
  reg:
    maxItems: 1
  vdd-supply: true
  interrupts:
    maxItems: 1
  bosch,odr-hz:
    $ref: /schemas/types.yaml#/definitions/uint32
    default: 50

required: [compatible, reg]
```

**Step 2: Add DTBO overlay**

```dts
/dts-v1/;
/plugin/;

&i2c1 {
    #address-cells = <1>;
    #size-cells = <0>;

    bmp390: pressure@77 {
        compatible = "bosch,bmp390";
        reg = <0x77>;
        interrupt-parent = <&gpio0>;
        interrupts = <28 IRQ_TYPE_EDGE_RISING>;
        vdd-supply   = <&vcc_3v3>;
        vddio-supply = <&vcc_1v8>;
        bosch,odr-hz = <50>;
    };
};
```

**Step 3: Write kernel driver**

```c
static const struct of_device_id bmp390_of_match[] = {
    { .compatible = "bosch,bmp390" },
    { }
};

static int bmp390_probe(struct i2c_client *client) {
    struct device_node *np = client->dev.of_node;

    u32 odr;
    of_property_read_u32(np, "bosch,odr-hz", &odr);

    struct regulator *vdd = devm_regulator_get(&client->dev, "vdd");
    regulator_enable(vdd);

    u8 chip_id = i2c_smbus_read_byte_data(client, 0x00);
    if (chip_id != 0x60) return -ENODEV;

    // Register IIO device ...
    return 0;
}
```

**Step 4: Add to `BoardConfig.mk`**

```makefile
BOARD_DTBO_SRC += \
    device/myvendor/myboard/dt/overlays/bmp390-overlay.dts
```

**Step 5: Build, flash, verify**

```bash
make dtboimage
fastboot flash dtbo dtbo.img
adb shell dmesg | grep bmp390
adb shell cat /sys/bus/iio/devices/iio:device0/name
# → bmp390
```

---

## Quick Reference: Most Frequently Asked

```
┌─────────────────────────────────────────────────────────────────────┐
│  FRESHER LEVEL:                                                     │
│  Q1:  What is Device Tree?                                          │
│  Q2:  .dts vs .dtsi vs .dtb                                        │
│  Q5:  What is compatible property?                                  │
│  Q6:  What does status = "okay" mean?                               │
│                                                                     │
│  INTERMEDIATE:                                                      │
│  Q7:  What is DTBO and why needed?                                  │
│  Q11: How bootloader processes DT?                                  │
│  Q14: How kernel probes drivers using DT?                           │
│  Q15: What is deferred probing?                                     │
│  Q18: DTB location change in Android 12?                            │
│                                                                     │
│  ADVANCED:                                                          │
│  Q21: Two overlays, same property — which wins?                     │
│  Q22: Debug device not probing?                                     │
│  Q25: What is pinctrl in DT?                                        │
│  Q19: DT and Treble relationship?                                   │
│                                                                     │
│  PRACTICAL:                                                         │
│  Q27: Write DT node for sensor                                      │
│  Q28: How to verify DTBO applied?                                   │
│  Q29/Q30: Diagnose hardware not working                             │
└─────────────────────────────────────────────────────────────────────┘
```

## ⚠️ Trick Questions to Watch

- **PCI/USB devices** do NOT use DT for device detection — they use their own discovery protocols
- The **`-@` flag** is required when compiling base DTB for overlay support
- **Last overlay wins** when two overlays modify the same property
- **Nodes MERGE, properties OVERWRITE** in overlays
- `status = "disabled"` nodes are **NOT probed** — no `platform_device` created
- **Android 12:** DTB moved from separate partition into `vendor_boot`
