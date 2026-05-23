# Building U-Boot for the VAR-SOM-MX93: A Complete Deep-Dive Guide

**Target hardware:** Variscite VAR-SOM-MX93 (NXP i.MX93 SoC, ARMv8-A)  
**Release:** Yocto Scarthgap 6.6.y_2.2.2-v1.0  
**Guide URL:** https://dev.variscite.com/var-som-mx93/mx93-yocto-scarthgap-6.6.y_2.2.2-v1.0/yocto-build-u-boot/

#Author: Firas Abd El Gani
---

## Table of Contents

1. [What is U-Boot? The Boot Sequence from Power-On to Linux](#1-what-is-u-boot)
2. [In-Yocto vs. Out-of-Yocto Builds](#2-in-yocto-vs-out-of-yocto-builds)
3. [Cross-Compilation Toolchains](#3-cross-compilation-toolchains)
4. [make mrproper, clean, and distclean](#4-make-mrproper-clean-and-distclean)
5. [defconfig Files](#5-defconfig-files)
6. [make -j$(nproc)](#6-make--jnproc)
7. [The i.MX93 Boot Image Components](#7-the-imx93-boot-image-components)
8. [imx-mkimage and soc.mak](#8-imx-mkimage-and-socmak)
9. [The dd Command and SD Card Layout](#9-the-dd-command-and-sd-card-layout)
10. [git pull and When to Update](#10-git-pull-and-when-to-update)
11. [Step-by-Step Walkthrough](#11-step-by-step-walkthrough)
12. [Visual Boot Sequence Diagram](#12-visual-boot-sequence-diagram)

---

## 1. What is U-Boot?

### What a Bootloader Actually Is

When you power on a computer — any computer, from a server to a microwave's control board — the processor doesn't magically know how to run Linux. The CPU has no inherent concept of a filesystem, a kernel image, RAM that needs initializing, or a display to print to. It only knows how to execute machine instructions starting from a specific, hardwired memory address.

A bootloader is the software that bridges this gap. It is the first real software the processor runs, and its job is to wake up the hardware just enough that a full operating system can take over.

U-Boot (Das U-Boot, "Universal Bootloader") is the most widely used open-source bootloader in the embedded Linux world. It supports hundreds of ARM, RISC-V, MIPS, and other architectures, and it is what your VAR-SOM-MX93 uses.

### The i.MX93 Boot Sequence: Step by Step from Power-On

Understanding this sequence is foundational to everything else in this guide.

**Step 1 — Power-On Reset**

The instant the PMIC (Power Management IC) asserts the reset line, the i.MX93's Boot ROM takes control. The Boot ROM is a small (typically 96–256 KB) read-only program permanently burned into the SoC silicon at the factory — it cannot be modified or erased. Its address is mapped such that the ARM core's reset vector points directly to it.

The Boot ROM is minimal by design. It knows how to talk to a fixed set of storage controllers (eMMC, SD card, SPI NOR flash, USB) and nothing else. It does NOT initialize DDR RAM — there is only the SoC's internal SRAM (OCRAM, roughly 256 KB on i.MX93) available at this point.

**Step 2 — Boot Mode Detection**

The Boot ROM reads boot mode pins and/or on-chip fuses (e-fuses are one-time-programmable bits) to decide where to look for the next piece of software. On your VAR-SOM-MX93 development kit, this is typically SD card or eMMC. The Boot ROM knows that for i.MX93, the first-stage bootloader is located at a specific byte offset on that storage medium (more on why in Section 9).

**Step 3 — Loading and Running the SPL**

The Boot ROM reads the image at the expected offset (32 KB into the SD card/eMMC), verifies its header (an NXP container header — more in Section 7), and copies it into OCRAM. This image is the SPL (Secondary Program Loader), also called the first-stage bootloader. Because OCRAM is small, SPL must be compact — typically under 200 KB.

The Boot ROM then jumps to the SPL's entry point. At this moment, the Boot ROM is no longer running — control has been fully transferred.

**Step 4 — SPL Initializes Hardware**

The SPL now runs from OCRAM and performs critical hardware initialization that the Boot ROM was too simple to do:

- **Clock tree setup:** The SoC has a complex network of PLLs (phase-locked loops) and clock dividers. The CPU, memory bus, and peripheral clocks all need to be configured at their correct frequencies.
- **DDR DRAM initialization:** This is the biggest job. The external LPDDR4/LPDDR4X DRAM chips on your module require a complex initialization sequence: power-up, mode register writes, and a calibration procedure called "DDR training" that tunes signal timing to compensate for PCB trace lengths and electrical characteristics. The SPL uses the Synopsys DDR PHY training firmware blob (from NXP) to perform this training. Without successful DDR init, there is nowhere to load U-Boot proper.
- **Basic console output:** SPL typically initializes one UART for debug output so you can see what's happening.

**Step 5 — SPL Loads the Second Stage**

With DDR now available (hundreds of megabytes of RAM), the SPL can load the much larger full U-Boot (u-boot.bin), ATF BL31, and related components from storage into DDR. It then sets up execution so that ATF BL31 runs first.

**Step 6 — ATF BL31 Establishes the Secure World**

ARM Trusted Firmware BL31 runs at EL3 — the highest and most privileged exception level in ARMv8-A. It sets up the Secure Monitor, initializes the TrustZone security partitioning, and registers services (called SMCs, Secure Monitor Calls) that software running at lower privilege levels can invoke. Once it has done its setup, BL31 remains resident in memory as a permanent EL3 handler — it never fully exits, it just drops execution down to U-Boot at EL2.

**Step 7 — U-Boot Proper Runs**

U-Boot now runs at EL2 (hypervisor exception level). This is the "full" bootloader with a rich feature set: a command-line shell, network stack (TFTP, NFS), USB support, filesystem drivers (FAT, ext4), environment variable storage, and crucially the ability to load a Linux kernel.

U-Boot's jobs include:
- Detecting or configuring the boot medium and boot partition
- Loading the Linux kernel image (Image or zImage) from storage or network into DDR
- Loading the correct Device Tree Blob (.dtb) for the hardware configuration
- Optionally loading an initial RAM disk (initrd/initramfs)
- Setting up kernel boot arguments (the `bootargs` environment variable)
- Performing any last hardware configuration the kernel needs
- Jumping to the kernel entry point (via the `booti` or `bootm` command), transferring control to Linux at EL1

**Step 8 — Linux Kernel Takes Over**

The kernel starts at EL1, decompresses itself if needed, reads the DTB to understand the hardware, initializes drivers, mounts the root filesystem, and starts the init process (systemd or BusyBox init). U-Boot is now completely done — it stays in memory but is never executed again (and its memory space gets reused by the kernel).


**What the Linux Kernel Actually Does — A Deep Dive**
The Linux kernel is the permanent, privileged core of the operating system — the layer of software that owns the hardware and acts as the sole mediator between all user applications and the physical world beneath them. When U-Boot hands off execution via booti, the kernel entry point receives two things in CPU registers: the address of the kernel image itself and the address of the Device Tree Blob (DTB) U-Boot placed in DDR. From that single jump, the kernel must bootstrap an entire operating environment from scratch. The first thing it does is self-relocation and decompression — if the kernel was stored as a compressed image (Image.gz or similar), the stub decompressor runs first, inflating the kernel into its final in-memory form before the real kernel code starts executing. Next comes MMU (Memory Management Unit) initialization: the processor is initially running with a flat, physical address space (what hardware sees), but the kernel immediately sets up page tables and enables the MMU, switching to a virtual address space. From this point on, every memory access — including the kernel's own code fetching instructions — goes through hardware address translation. This is foundational to everything that follows: it enforces memory isolation between processes, allows the kernel to map the same physical RAM at different virtual addresses for different purposes, and makes kernel memory invisible and inaccessible to userspace.
With the MMU active, the kernel parses the Device Tree Blob passed by U-Boot. This is the kernel's hardware manifest — a structured binary description of every peripheral, bus, interrupt controller, clock domain, and pin configuration on the board. The kernel has no other way to know about the hardware; there is no BIOS, no PCI auto-discovery, no ACPI table on an embedded ARM platform. The DTB tells it: "there is a UART at physical address 0x44380000, connected to interrupt 19, driven by clock LPUART1_GATE." The kernel reads each node, matches the compatible string (e.g., "fsl,imx93-uart") against its driver registry, and instantiates a driver for each matched device. This binding process is what makes the kernel's single binary work across dozens of different i.MX93-based boards — the hardware description is external and swappable, the driver logic is compiled in.
Simultaneously, the kernel initializes its core subsystems in a carefully sequenced order: the interrupt controller (GIC, Generic Interrupt Controller) is set up so that hardware events can signal the CPU; the scheduler is initialized so that multiple threads of execution can share the CPU fairly; the memory allocator (SLAB/SLUB) is brought up so that dynamic kernel memory allocation (kmalloc) becomes possible; the virtual filesystem layer (VFS) is initialized as an abstraction over all storage backends; and the process subsystem is prepared to create and manage tasks. Each of these subsystems has deep interdependencies — the scheduler needs the memory allocator, drivers need the interrupt controller, the VFS needs drivers — so the kernel's start_kernel() function calls them in a precise order honed over decades of kernel development.
Once subsystems are live, the kernel begins driver initialization for all the devices the DTB described. Each driver's probe() function runs: it maps the device's physical registers into the kernel's virtual address space (ioremap), requests its interrupt lines, sets up DMA buffers if needed, and registers itself with the appropriate kernel subsystem (a UART driver registers with the tty layer, an eMMC driver registers with the block layer, an Ethernet driver registers with the net layer). This is when the hardware truly "wakes up" under kernel control — the eMMC controller is initialized, clocks are configured, and the kernel can now read and write blocks of storage. Critically, all this happens through the kernel's driver model (sometimes called the device model or devtmpfs model), which manages power, binding, and the /sys filesystem entries that expose hardware state to userspace.
With block devices available, the kernel can mount the root filesystem. U-Boot passed the root= argument in bootargs (e.g., root=/dev/mmcblk0p2 rootfstype=ext4), telling the kernel which partition to use and which filesystem driver to invoke. The VFS calls the ext4 driver, which reads the superblock, validates the filesystem, and makes the directory tree accessible. At this point the kernel has a complete filesystem hierarchy. It then searches for and executes PID 1 — typically /sbin/init (systemd) or /bin/sh (BusyBox init). This is the single most privileged userspace process, the ancestor of every other process that will ever run, and its startup marks the moment the system transitions from "the kernel bootstrapping itself" to "a running operating system serving applications." From here on, the kernel's role shifts from initialization to its permanent runtime function: arbitrating access to hardware (via system calls like read(), write(), ioctl()), scheduling CPU time across all running processes, enforcing memory isolation so no process can corrupt another's memory or the kernel's own, managing power states (calling down to ATF BL31 via SMC instructions for PSCI operations like CPU suspend or system reboot), and handling faults — translating hardware exceptions like page faults, bus errors, and undefined instructions into signals or process kills rather than silent corruption. The kernel never fully "completes" its work — it is a permanent, event-driven executive that runs for as long as the system is powered on, invisibly managing every hardware interaction that every piece of software, from a shell script to a real-time control loop, will ever make.

### What Happens If U-Boot Is Missing or Corrupted?

If the boot image at offset 32 KB is absent or has a bad NXP container header, the Boot ROM will fail to find a valid image. Depending on fuse settings, it will either:
- Try the next boot source (USB download mode, for example) — this is how you can recover a bricked board using NXP's UUU tool
- Hang with no output (most common during development when you flash a bad image)

You will see nothing on the UART console, the board will appear dead. This is why having a working recovery SD card (a second SD card with a known-good image) is essential during U-Boot development.

If U-Boot is present but its environment is corrupted (the env partition on eMMC has garbage data), U-Boot will detect the CRC error on the saved environment, print a warning like `*** Warning - bad CRC, using default environment`, and fall back to the compiled-in defaults. This is usually recoverable.

---

## 2. In-Yocto vs. Out-of-Yocto Builds

### What the Yocto Build System Does

Yocto is a massive meta-build system. When you do a full Yocto build, it fetches source code for hundreds of packages, cross-compiles them all, and assembles a complete bootable Linux image. U-Boot is one package among many. Yocto fetches the U-Boot source, applies patches, configures it with the correct defconfig, sets all the right environment variables, and builds it — but this all happens deep inside Yocto's own pipeline.

Building U-Boot via Yocto has advantages: it's reproducible, it correctly tracks dependencies, and the output is integrated into the final image automatically. But it also means:

- A full Yocto build takes 4–12+ hours on first run (it compiles everything from scratch, including GCC itself)
- Rebuilding U-Boot inside Yocto still invokes Yocto's task scheduler overhead
- You cannot easily step through or debug the build process
- Making a small U-Boot change and testing it requires triggering a Yocto rebuild that may rebuild more than you want

### Why Build Out of Yocto Tree?

Building U-Boot out-of-tree means working directly with the U-Boot source code and the cross-compiler toolchain, bypassing Yocto entirely. This is the professional workflow for engineers who are actively developing or debugging U-Boot. The reasons are:

**Speed:** Once the toolchain is installed, a U-Boot rebuild from scratch takes 1–3 minutes. An incremental rebuild (after changing one file) takes seconds. Compare this to triggering a Yocto task rebuild.

**Transparency:** Every `make` command you type is directly building U-Boot. There is no abstraction layer. If something goes wrong, you see the exact compiler error immediately.

**Iteration:** You can edit a file, rebuild, flash an SD card, and test in under 5 minutes. This tight loop is essential for debugging boot issues.

**Toolchain reuse:** Yocto builds and installs a standalone SDK/toolchain that you can use forever without re-running the full Yocto build. Once installed to `/opt/fsl-imx-xwayland/...`, it is self-contained.

**The tradeoff:** You are responsible for keeping your out-of-tree build consistent with the versions of supporting firmware blobs (ATF, Sentinel, DDR firmware) that Yocto would have used. The Variscite guide handles this for you by specifying exact git branch tags and download URLs.

---

## 3. Cross-Compilation Toolchains

### Why You Cannot Use Your Host's gcc

Your development machine runs Ubuntu (or similar) on an x86_64 processor. The VAR-SOM-MX93 runs on an NXP i.MX93, which is an ARMv8-A (AArch64) processor. These are completely different instruction set architectures (ISAs).

The `gcc` on your Ubuntu machine produces x86_64 machine code — instructions that the i.MX93's ARM cores cannot execute at all. If you ran `make` on U-Boot with your system gcc, you would produce a binary that might look valid but would crash instantly when the ARM processor tried to run it (or the NXP container validation would reject it during boot).

A **cross-compiler** is a compiler that runs on your host machine (x86_64) but produces machine code for a different target architecture (AArch64). The cross-compiler executable itself is an x86_64 program — it runs on your PC — but the binaries it produces contain ARM instructions for the i.MX93.

Cross-compilers are identified by a **target triplet** (or quadruplet) prefix that describes the target: `aarch64-poky-linux-`. In this context:
- `aarch64` — 64-bit ARM architecture (ARMv8-A)
- `poky` — the vendor/OS name (Poky is Yocto's reference distro)
- `linux` — Linux ABI (Application Binary Interface)

Every cross-tool carries this prefix: `aarch64-poky-linux-gcc`, `aarch64-poky-linux-ld`, `aarch64-poky-linux-objcopy`, etc.

### What the Yocto Toolchain Is

Yocto (via its `bitbake -c populate_sdk` task or a pre-built SDK installer) produces a standalone toolchain installer — a self-extracting shell script that installs everything needed to cross-compile for the target into a single directory. For your release, it installs to:

```
/opt/fsl-imx-xwayland/6.6-scarthgap/
```

Inside this directory you'll find:
- `sysroots/x86_64-pokysdk-linux/` — the host tools (cross-gcc, cross-binutils, etc.)
- `sysroots/armv8a-poky-linux/` — the target sysroot (target headers, libraries)
- `environment-setup-armv8a-poky-linux` — the shell script you source

### What `source environment-setup-armv8a-poky-linux` Does

The `source` command (equivalent to `.`) runs a shell script in the **current shell's context**, meaning all the environment variable assignments in the script stick after it finishes. If you ran it with `bash environment-setup-...` instead, those variables would be set inside a child shell that immediately exits — they would be lost.

This environment script sets dozens of variables. The critical ones for U-Boot are:

```bash
export ARCH=arm64
# Tells the U-Boot Makefile to use the arm64 architecture config.
# Without this, make would try to build for your host architecture (x86_64).

export CROSS_COMPILE=aarch64-poky-linux-
# This prefix is prepended to every tool invocation.
# When U-Boot's Makefile calls $(CC), it expands to $(CROSS_COMPILE)gcc
# = aarch64-poky-linux-gcc.
# Same for $(LD), $(AR), $(OBJCOPY), etc.

export CC="aarch64-poky-linux-gcc  --sysroot=/opt/fsl-imx-xwayland/6.6-scarthgap/sysroots/armv8a-poky-linux"
# The full C compiler command with the target sysroot path so the compiler
# finds the correct headers and libraries for the target.

export CXX="aarch64-poky-linux-g++  --sysroot=..."
export LD="aarch64-poky-linux-ld  --sysroot=..."
export AR=aarch64-poky-linux-ar
export AS=aarch64-poky-linux-as
export RANLIB=aarch64-poky-linux-ranlib
export STRIP=aarch64-poky-linux-strip
export NM=aarch64-poky-linux-nm
export OBJCOPY=aarch64-poky-linux-objcopy
export OBJDUMP=aarch64-poky-linux-objdump

export LDFLAGS="--sysroot=..."
# Linker flags. NOTE: This BREAKS the ATF build, which is why
# you must 'unset LDFLAGS' before building ATF (covered in the walkthrough).

export PATH="/opt/fsl-imx-xwayland/6.6-scarthgap/sysroots/x86_64-pokysdk-linux/usr/bin/aarch64-poky-linux:$PATH"
# Prepends the cross-compiler's bin directory to PATH, so when the Makefile
# calls 'gcc' it finds aarch64-poky-linux-gcc from this path.

export PKG_CONFIG_PATH="..."
export SDKTARGETSYSROOT="/opt/fsl-imx-xwayland/6.6-scarthgap/sysroots/armv8a-poky-linux"
export OECORE_NATIVE_SYSROOT="..."
```

After sourcing this script, your shell is "poisoned" for cross-compilation — every call to `gcc` invokes the cross-compiler. This is exactly what you want. Do not open a new terminal and source the file again for each component; source it once per shell session.

### Verifying the Toolchain Is Active

After sourcing, run:
```bash
echo $CROSS_COMPILE
# Should print: aarch64-poky-linux-

$CC --version
# Should print something like:
# aarch64-poky-linux-gcc (GCC) 13.3.0
# This confirms you have the cross-compiler active.

$CC -print-sysroot
# Should print the sysroot path
```

---

## 4. make mrproper, clean, and distclean

The U-Boot (and Linux kernel) build system produces many intermediate files during compilation: object files (.o), dependency files (.d), generated C headers, the final `.config`, and more. Before switching to a completely different configuration (e.g., switching from one board's defconfig to another, or after a major source update), you need to clean out stale files that could cause subtle build errors.

The three clean targets are:

### `make clean`

Removes compiled object files, module files, and most generated binaries. It does NOT remove the `.config` file or generated header files (like `include/autoconf.mk` and `include/config/`). Use `make clean` when you have changed source files and want a clean rebuild of binaries, but want to keep your current board configuration.

### `make mrproper`

This goes further. In addition to everything `make clean` removes, it also removes:
- The `.config` file
- `.config.old`
- All generated configuration headers (`include/autoconf.mk`, `include/config/`, `include/generated/`)
- Editor and patch artifacts

The name comes from a German cleaning product brand — it means "making things proper" (thorough cleaning). This is the correct choice before applying a new `defconfig`. You are wiping out all previous configuration choices and starting fresh.

**Use this whenever you want to switch board configurations or after fetching new source code that significantly changed the build system.**

### `make distclean`

A superset of `mrproper`. Also removes editor backup files (`*.orig`, `*.rej`), sparse output files, and other files that are not normally tracked. Used mainly to prepare a source tree for redistribution. In practice, `mrproper` is sufficient for day-to-day development.

**Rule of thumb:** Before building for a new board or after a `git pull` that changed Kconfig files, always run `make mrproper`.

---

## 5. defconfig Files

### What Is Kconfig?

U-Boot (like the Linux kernel) uses the Kconfig system for build configuration. Kconfig defines thousands of boolean, tristate, integer, and string options (called `CONFIG_` symbols) that control which features, drivers, and behaviors are compiled in. For example:

```
CONFIG_CMD_BOOTM=y         # Include the bootm command
CONFIG_NET=y               # Include network stack
CONFIG_DM_I2C=y            # Include driver model I2C support
CONFIG_SYS_TEXT_BASE=0x80200000  # Where in RAM U-Boot proper is linked
```

### Where Does defconfig Live?

Defconfig files live in `configs/` within the U-Boot source tree. You will find:

```
uboot-imx/
└── configs/
    ├── imx93_var_som_defconfig    ← your board's config
    ├── imx93_11x11_evk_defconfig
    ├── ...hundreds of other board configs...
```

### What Does a defconfig Contain?

A defconfig file is **not** a complete `.config`. It contains only the configuration symbols whose values differ from the Kconfig-defined defaults, plus any symbols that have no default. This keeps the file small and readable — typically a few dozen to a few hundred lines instead of thousands.

For example, `imx93_var_som_defconfig` might contain entries like:

```
CONFIG_TARGET_IMX93_VAR_SOM=y
CONFIG_SPL=y
CONFIG_IMX93=y
CONFIG_AHAB_BOOT=y
CONFIG_SYS_MALLOC_F_LEN=0x10000
CONFIG_ENV_IS_IN_MMC=y
...
```

### What Does `make imx93_var_som_defconfig` Do?

This target invokes the `scripts/kconfig/conf` utility with the `--defconfig` option pointing at `configs/imx93_var_som_defconfig`. Kconfig reads all the `Kconfig` definition files throughout the source tree (which define every possible symbol, its type, its default value, and its dependencies). It then processes the defconfig values and fills in all the blanks with defaults, resolving dependencies (e.g., if `CONFIG_NET=y` requires `CONFIG_DM=y`, DM is automatically enabled). The result is a fully complete `.config` file with every symbol assigned a value.

You can then run `make menuconfig` to interactively browse and modify the configuration before building. This generates a new `.config` but does not update the `defconfig` — to update the defconfig with your changes, you would run `make savedefconfig` and then copy the output.

### The Resulting .config

After running the defconfig target, you will find a `.config` file at the top of the source tree. It is typically 4,000–7,000 lines long. This is what the build system actually reads during compilation. Never edit `.config` directly if you plan to use `menuconfig` later, because it will be regenerated.

---

## 6. make -j$(nproc)

### What -j Means

The `-j` flag to `make` stands for "jobs" — it controls the maximum number of parallel compilation processes. Without `-j`, make runs one process at a time: compile file A, wait for it to finish, compile file B, wait, etc. On a modern multi-core CPU, this leaves most cores idle.

With `-j8` (for example), make starts up to 8 compilation jobs simultaneously, using all 8 cores. U-Boot has hundreds of source files, so parallelism dramatically reduces build time.

### What `$(nproc)` Is

`$(...)` is shell command substitution. `nproc` is a Linux utility that prints the number of available processing units (cores/threads). So `$(nproc)` evaluates to the number of CPUs on your machine at the time make is invoked. On a 4-core machine it gives `make -j4`; on a 16-core machine, `make -j16`.

This is idiomatic: you don't hardcode `-j8` because you might run this command on different machines. `$(nproc)` always gives the optimal value.

### Build Time Expectations

On a modern 8-core development machine, a clean U-Boot build takes approximately 60–120 seconds. An incremental rebuild after changing one file takes 2–10 seconds.

---

## 7. The i.MX93 Boot Image Components

The i.MX93's boot image is not a single monolithic binary. It is a carefully structured container that packages multiple firmware blobs together, each serving a distinct purpose. Understanding what each one does is essential to understanding why the build process is so involved.

### 7.1 The NXP Image Container Format

NXP i.MX9 chips use a firmware packaging format called an **Image Container**. The Boot ROM understands this format. A container has a header that describes the firmware blobs inside it (their load addresses, entry points, and cryptographic signatures for secure boot). The final `flash.bin` / `imx-boot` is actually one or more such containers concatenated.

### 7.2 SPL — u-boot-spl.bin

The SPL (Secondary Program Loader) is the first piece of software that U-Boot contributes to the boot sequence. It is compiled from the same U-Boot source tree as U-Boot proper, but configured to be tiny (it must fit in OCRAM, typically under 200 KB on i.MX93).

SPL is built with a separate Kconfig subset controlled by `CONFIG_SPL=y` and related `CONFIG_SPL_*` symbols. Its job is narrow: initialize the minimum hardware (clocks, DDR via the Synopsys PHY firmware), then load U-Boot proper, ATF BL31, and related firmware into DDR, and hand off.

The output file `spl/u-boot-spl.bin` is a raw binary image of the SPL.

### 7.3 U-Boot Proper — u-boot.bin

This is the full-featured bootloader. It runs after SPL has initialized DDR and jumps to it. `u-boot.bin` is a raw binary (as opposed to `u-boot` which is an ELF file with debug info, and `u-boot.img` which adds a legacy U-Boot header). The `u-boot.bin` is what gets packaged into the boot container.

### 7.4 ATF BL31 — bl31.bin (ARM Trusted Firmware)

#### What ARMv8-A Exception Levels Are

ARMv8-A defines four numbered Exception Levels (ELs), from least to most privileged:

```
EL0  —  Userspace applications (your app, shell scripts)
EL1  —  Operating system kernel (Linux)
EL2  —  Hypervisor (KVM, Xen) or bootloader (U-Boot uses EL2)
EL3  —  Secure Monitor (ATF BL31) — most privileged
```

Higher EL = more hardware access, more trust. Code at EL3 can do things code at EL1 cannot — like configure TrustZone security, access secure registers, and manage the secure/non-secure world split.

#### What TrustZone Is

ARM TrustZone is a hardware security extension built into the ARM cores. It partitions the processor into two "worlds":

- **Secure World:** Can access all memory and peripherals, including those marked as "secure only." Runs at EL3 (Secure Monitor) and EL1-S (Secure OS like OP-TEE).
- **Normal World:** Cannot access Secure World memory/peripherals. Runs at EL2 and EL1-NS (Linux, U-Boot).

The boundary between worlds is enforced by hardware — even the Linux kernel cannot reach into Secure World memory. This is what protects things like cryptographic keys, DRM content decryption, and secure payment processing.

#### Why ATF BL31 Is Needed

On ARMv8-A systems, *something* must run at EL3 to set up this environment. ATF (ARM Trusted Firmware, now called TF-A) is the open-source implementation. BL31 specifically is the "EL3 Runtime Firmware" — it initializes TrustZone configuration, registers Power State Coordination Interface (PSCI) handlers (for CPU hotplug, system suspend/resume), and remains resident at EL3 for the lifetime of the system.

Without ATF BL31, there is no EL3 handler. Linux (at EL1) cannot call PSCI to park cores or suspend to RAM. Some NXP-specific secure services would be unavailable. The system would not boot correctly on i.MX93.

Variscite maintains their own fork of ATF at `github.com/varigit/imx-atf` with i.MX93-specific patches on top of NXP's fork (which is itself on top of upstream TF-A).

#### Why `unset LDFLAGS` Before Building ATF

The Yocto toolchain script sets `LDFLAGS` with a `--sysroot=...` path pointing to the Yocto target sysroot. This is correct for U-Boot, which links against the Yocto-provided libc headers. But ATF BL31 is freestanding firmware — it does not link against any libc at all, and it has its own linker scripts. If `LDFLAGS` contains sysroot flags, the ATF Makefile's linker invocations will fail with path errors. Unsetting `LDFLAGS` before building ATF prevents this.

### 7.5 Sentinel Firmware — mx93a1-ahab-container.img

#### What the EdgeLock Secure Enclave (Sentinel) Is

The i.MX93 contains, in addition to its two main Cortex-A55 application cores, a dedicated security processor: a separate Cortex-M33-based subsystem NXP calls the **EdgeLock Secure Enclave** (sometimes called Sentinel or ELE). This is a completely independent processor within the SoC with its own ROM, RAM, and cryptographic hardware accelerators.

The Sentinel runs NXP-signed firmware (which you cannot modify — it is signed with NXP's private key and validated by hardware). It handles:
- Cryptographic key derivation and storage
- Random number generation (TRNG)
- Secure boot image authentication (AHAB)
- Lifecycle management (transitioning the chip from development to production security)
- Runtime cryptographic services (used by OP-TEE, for example)

#### What AHAB Is

AHAB (Advanced High Assurance Boot) is NXP's secure boot framework for i.MX9 chips. The Boot ROM validates each firmware component in the boot container using the Sentinel. In non-secure-boot mode (OEM open, the default for development), the Sentinel firmware is still required to initialize the enclave hardware and provide its services — the signature verification steps are skipped, but the Sentinel itself still needs to boot.

`mx93a1-ahab-container.img` is the signed NXP container that carries the Sentinel firmware. It must be present in the boot image for the system to boot correctly. The `a1` in the name refers to silicon revision A1 of the i.MX93.

#### How You Get It

NXP distributes this as a self-extracting binary under their Proprietary License Agreement. The `wget` step downloads it from NXP's servers, `chmod +x` makes it executable, and running it extracts the actual firmware file. You must accept NXP's EULA during extraction.

### 7.6 DDR PHY Firmware — Synopsys DDR Training Firmware

#### Why DDR Initialization Requires Firmware

Modern LPDDR4/LPDDR4X DRAM initialization is not as simple as writing a few registers. It requires a process called **DDR training**, where the DDR controller's PHY (Physical Interface layer) — the analog circuitry that drives the data bus — calibrates timing parameters specific to your PCB layout.

Every PCB has slightly different trace lengths, impedances, and crosstalk characteristics. DDR training measures these at runtime and programs compensation values into the PHY registers to achieve reliable signal timing. The training process involves sending known test patterns and measuring the results — it's an iterative optimization loop.

The Synopsys DWC DDRPHY (which NXP uses in i.MX93) implements this training as firmware that runs on a tiny internal processor within the DDR PHY block itself. This firmware blob is not open source — it is proprietary Synopsys intellectual property licensed to NXP, then distributed as part of NXP's BSP.

The firmware files you copy are typically:
- `lpddr4_imem_1d_*.bin` — instruction memory for 1D training phase
- `lpddr4_dmem_1d_*.bin` — data memory for 1D training phase
- `lpddr4_imem_2d_*.bin` — instruction memory for 2D training phase
- `lpddr4_dmem_2d_*.bin` — data memory for 2D training phase

These blobs are loaded by SPL into the DDR PHY's internal memory at runtime during the DDR initialization sequence.

### 7.7 Device Tree Blobs — .dtb Files

#### What a Device Tree Is

The Linux kernel is designed to run on many different hardware configurations. But how does a single kernel binary know that this particular board has an I2C sensor at address 0x48 on bus 2, while another board has it at address 0x50 on bus 1? How does it know which UART is the console? Which pins are GPIO and which are SPI?

On x86 PCs, hardware is discoverable via PCI and ACPI. Embedded hardware is not discoverable — the kernel must be told about it. Device Trees solve this. A Device Tree Source (DTS) file is a human-readable description of the hardware:

```c
uart0: serial@44380000 {
    compatible = "fsl,imx93-uart";
    reg = <0x44380000 0x10000>;
    interrupts = <GIC_SPI 19 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clk IMX93_CLK_LPUART1_GATE>;
    status = "okay";
};
```

The DTS file is compiled by the Device Tree Compiler (`dtc`) into a compact binary format called a Device Tree Blob (DTB, `.dtb` extension). U-Boot receives the correct DTB as part of its build, and when it boots Linux, it passes the DTB's address to the kernel in a CPU register. The kernel reads it to discover all hardware.

#### The Two DTB Files in This Build

The VAR-SOM-MX93 module can be paired with different carrier boards. These carrier boards have different peripherals, connectors, and pin assignments:

- **`imx93-var-som-symphony.dtb`** — for the Variscite Symphony carrier board, Variscite's full-featured development and evaluation board with audio, display connectors, PCIe, USB hub, etc.
- **`imx93-var-dart-dt8mcustomboard.dtb`** — for the DT8MCustomBoard, another Variscite carrier board variant.

Both DTBs are compiled from DTS files in `arch/arm/dts/` within the U-Boot source tree. The SOM-specific hardware (CPU, DDR, eMMC on the module) is usually in a shared `.dtsi` include file, while carrier-board-specific peripherals are in the main `.dts` file.

The boot image packages both DTBs. U-Boot uses environment variables and board detection logic to choose the correct DTB at runtime.

---

## 8. imx-mkimage and soc.mak

### What imx-mkimage Is

`imx-mkimage` is an NXP tool that assembles all the individual firmware blobs into a single bootable image in NXP's container format. It understands the container header structure that the i.MX93 Boot ROM can parse, knows the correct load addresses for each component, and knows how to lay them out in memory and in the output binary.

The tool is structured around per-SoC `soc.mak` Makefiles in subdirectories: `iMX8M/soc.mak`, `iMX93/soc.mak`, etc. Each `soc.mak` knows the specifics of that SoC's container format, component names, and assembly steps.

### Building imx-mkimage for iMX93

```bash
make soc=iMX93
```

This builds the `mkimage_imx8` binary (despite the name, it's used for both i.MX8 and i.MX9 families — NXP reused the binary with updated logic) and populates `iMX93/soc.mak` with the correct rules. You then copy both out:

```bash
cp iMX93/soc.mak mkimage_imx8 ..
```

### What `flash_singleboot` Does

The `flash_singleboot` target in `soc.mak` instructs `mkimage_imx8` to:

1. Take all the input files you've assembled in `imx-boot-tools/`: SPL, U-Boot proper, ATF BL31, Sentinel container, DDR firmware blobs, DTBs
2. Create an NXP Image Container with appropriate headers, specifying load addresses for each blob
3. Pack everything into a single binary output file called `flash.bin`

The resulting `flash.bin` (renamed to `imx-boot`) is a single file you can write to an SD card or eMMC at offset 32 KB. The Boot ROM will parse the container headers and know exactly where to copy each component and where to jump.

### The Full `make -f soc.mak` Command Explained

```bash
make -f soc.mak \
    SOC=iMX93 \
    dtbs="imx93-var-dart-dt8mcustomboard.dtb imx93-var-som-symphony.dtb" \
    MKIMG=./imx-mkimage/mkimage_imx8 \
    PAD_IMAGE=./pad_image.sh \
    CC=gcc \
    flash_singleboot
```

- `-f soc.mak` — use `soc.mak` as the Makefile (not the default `Makefile`)
- `SOC=iMX93` — variable passed into soc.mak selecting the i.MX93 rules
- `dtbs="..."` — space-separated list of DTB files to include in the image
- `MKIMG=./imx-mkimage/mkimage_imx8` — path to the mkimage tool binary
- `PAD_IMAGE=./pad_image.sh` — a helper script that pads binary files to alignment boundaries
- `CC=gcc` — use the host's gcc (not the cross-compiler) for any soc.mak build-time utilities that run on the host. Note this overrides the `CC` set by the Yocto toolchain env. This is correct because imx-mkimage itself runs on the host PC, not on the target.
- `flash_singleboot` — the make target to build

---

## 9. The dd Command and SD Card Layout

### Why the Boot Image Is at Offset 32 KB

An SD card (or eMMC) begins at byte offset 0. The absolute first sector (512 bytes) is traditionally the MBR (Master Boot Record) on MBR-partitioned disks, or the first part of a GPT (GUID Partition Table) header. Either way, the partition table data lives in the first ~34 sectors (17 KB for GPT). This must be preserved — if you overwrite it with boot firmware, the OS cannot find the partitions.

NXP's Boot ROM for i.MX93 is designed to look for the boot image at offset 32 KB (64 sectors × 512 bytes/sector). This 32 KB gap is large enough to hold any partition table format safely. Sector 64 (0-indexed) is the first sector the Boot ROM reads when looking for an NXP image container.

This is a fixed hardware/firmware contract defined in NXP's i.MX93 Reference Manual and Boot ROM implementation. You cannot change it without custom Boot ROM fuse settings.

So the SD card layout looks like:

```
Offset 0 KB     — MBR or GPT protective MBR (first 512 bytes)
Offset 0.5 KB   — GPT header (if GPT-partitioned)
Offset 1–32 KB  — GPT partition entries, reserved space
Offset 32 KB    — imx-boot (SPL + ATF + U-Boot + Sentinel + DDR FW)
                  This overwrites whatever was here, but it's in the gap
                  between the partition table and the first partition.
                  The first partition typically starts at offset 1 MB or later.
Offset 1 MB+    — FAT boot partition (with kernel, DTB, etc.)
                  rootfs partition (ext4)
                  etc.
```

### The dd Command: Every Argument

```bash
sudo dd if=imx-boot of=/dev/sdX bs=1K seek=32 conv=fsync
```

**`sudo`** — `dd` writes directly to a block device, bypassing the filesystem. This requires root privileges.

**`if=imx-boot`** — "input file." This is the source data. `dd` reads from this file sequentially.

**`of=/dev/sdX`** — "output file." This is the destination. `/dev/sdX` is a raw block device (e.g., `/dev/sdb` for a USB SD card reader). Writing to `/dev/sdX` (not `/dev/sdX1`) writes to the raw device, starting from absolute offset 0 of the storage medium.

**`bs=1K`** — "block size." This sets both the read and write block size to 1024 bytes. `dd` reads 1 KB at a time and writes 1 KB at a time. This affects performance (larger block sizes = fewer I/O calls = faster), but for correctness, the key is that `seek` is measured in units of `bs`.

**`seek=32`** — Skips 32 blocks at the output before writing. With `bs=1K`, each block is 1024 bytes, so `seek=32` means start writing at byte offset 32 × 1024 = 32,768 bytes = 32 KB. This is how we land the boot image exactly at offset 32 KB without overwriting the partition table.

**`conv=fsync`** — After all data is written, call `fsync()` on the output file descriptor before exiting. This forces the kernel to flush all write-back cache to the actual storage device. Without this, `dd` might return "success" while data is still sitting in kernel write buffers — and if you remove the SD card immediately, the write might be incomplete. `conv=fsync` ensures the data is physically on the SD card when dd exits.

### What Happens If You Write to the Wrong /dev/sdX

`dd` is sometimes called "disk destroyer" in sysadmin culture for good reason. If you accidentally specify your system's primary SSD (`/dev/sda`) instead of your SD card (`/dev/sdb`):
- The MBR/GPT of your main drive is overwritten with the U-Boot image
- Your machine will fail to boot
- All your data is not necessarily gone (it is still on the disk), but the partition table is gone, making recovery complex and potentially destructive

**Always verify the correct device** with `lsblk` or `dmesg | tail -30` after inserting the SD card before running `dd`. The correct device typically shows as something like `/dev/sdb` with partitions `/dev/sdb1`, `/dev/sdb2`. Its size should match your SD card capacity.

```bash
# Safe workflow:
lsblk                      # See all block devices and sizes
# Identify your SD card, e.g. /dev/sdb (8GB card = ~7.4G shown)
sudo dd if=imx-boot of=/dev/sdb bs=1K seek=32 conv=fsync
```

---

## 10. git pull and When to Update

`git pull` is shorthand for `git fetch` (download new commits from the remote repository) followed by `git merge` (integrate those commits into your current branch). In the context of `uboot-imx` on the `lf_v2024.04_6.6.52-2.2.2_var01` branch, running `git pull` would fetch any new commits Variscite has pushed to that branch since you last cloned or pulled.

**When to run `git pull`:**
- When Variscite releases a patch or bug fix for that branch
- When you want to pick up upstream U-Boot fixes that have been backported
- When Variscite's release notes indicate a specific commit is needed for a fix you need

**Risks to be aware of:**
- New commits might change `Kconfig` symbols, breaking your custom configuration
- New commits might change `imx93_var_som_defconfig`, meaning a `make mrproper` + defconfig re-application is needed
- If you have local uncommitted changes, a pull might create merge conflicts you must resolve
- The new commits might be incompatible with the current versions of ATF, Sentinel firmware, or DDR firmware — always check if the branch version string has changed and update the supporting firmware accordingly

**Best practice after `git pull`:**
```bash
git pull
make mrproper
make imx93_var_som_defconfig
make -j$(nproc)
```

Then rebuild the full boot image with `imx-mkimage`.

---

## 11. Step-by-Step Walkthrough

This section walks through every command in the guide. We will work in a fresh working directory — let's call it `~/var-mx93/`.

```bash
mkdir ~/var-mx93
cd ~/var-mx93
```

---

### Step 1: Install the Yocto Toolchain

Before anything else, you need the cross-compilation toolchain. Follow Variscite's [Toolchain installation guide](https://dev.variscite.com/var-som-mx93/mx93-yocto-scarthgap-6.6.y_2.2.2-v1.0/yocto-toolchain-installation/). In summary, you download an SDK installer script from Variscite or generate one from Yocto, run it, and it installs to `/opt/fsl-imx-xwayland/6.6-scarthgap/`.

**Verify the toolchain exists:**
```bash
ls /opt/fsl-imx-xwayland/6.6-scarthgap/
# You should see: environment-setup-armv8a-poky-linux  sysroots/  ...
```

---

### Step 2: Clone the U-Boot Source

**What we are doing:** Obtaining Variscite's fork of NXP's `uboot-imx` repository, on the exact branch that matches the BSP release.

```bash
git clone https://github.com/varigit/uboot-imx.git -b lf_v2024.04_6.6.52-2.2.2_var01
cd uboot-imx
```

**Arguments explained:**
- `git clone <url>` — downloads the entire repository (all commits, branches, tags) from GitHub into a new local directory named `uboot-imx`
- `-b lf_v2024.04_6.6.52-2.2.2_var01` — immediately checks out this specific branch after cloning. Without `-b`, git would check out the default branch (usually `main` or `master`), which might not match the Variscite BSP version you are using.

The branch name `lf_v2024.04_6.6.52-2.2.2_var01` decodes as:
- `lf` — Linux Foundation (NXP's BSP is released under LF governance)
- `v2024.04` — U-Boot upstream version 2024.04
- `6.6.52` — Linux kernel version this BSP targets
- `2.2.2` — NXP BSP release number
- `var01` — Variscite patch revision 1 on top of NXP's release

**Successful output looks like:**
```
Cloning into 'uboot-imx'...
remote: Enumerating objects: 850000, done.
remote: Counting objects: 100% (...)...
...
Resolving deltas: 100% (...), done.
```

**Errors to watch for:**
- `fatal: repository 'https://...' not found` — check your internet connection and the URL
- `error: pathspec '...' did not match any file(s) known to git` — the branch name is wrong or doesn't exist yet; double-check the exact branch name in the guide

After cloning, the `uboot-imx/` directory contains the full U-Boot source. Key subdirectories:
- `arch/arm/` — ARM architecture code, including device trees in `arch/arm/dts/`
- `board/variscite/` — Variscite board-specific C code
- `configs/` — defconfig files
- `drivers/` — device drivers
- `spl/` — SPL-specific source
- `tools/` — host-side tools including `mkimage`

---

### Step 3: Source the Yocto Toolchain Environment

**What we are doing:** Configuring the current shell session to use the cross-compiler.

```bash
source /opt/fsl-imx-xwayland/6.6-scarthgap/environment-setup-armv8a-poky-linux
```

You must be inside the `uboot-imx/` directory (or any directory where you plan to build) before doing this, though technically the source path doesn't matter — the environment is set globally in your shell.

**Successful output:** Typically no output at all, or a brief message like:
```
SDK environment now set up; additionally you may now run devtool to perform development tasks.
```

**Verify it worked:**
```bash
echo $CROSS_COMPILE
# aarch64-poky-linux-

echo $ARCH
# arm64

${CROSS_COMPILE}gcc --version
# aarch64-poky-linux-gcc (GCC) 13.3.0 20240311 (... poky ...)
```

**Critical pitfall:** This sourcing affects only the **current terminal session**. If you close the terminal and open a new one, you must source the file again before running any make commands. A very common beginner mistake is opening a new terminal, trying to build, and getting errors because the environment is not set.

**Another pitfall:** Do NOT source the environment and then switch to a different directory that has its own `Makefile` with a different meaning for `CC` or `ARCH`. The environment is global to the shell.

---

### Step 4: Clean the Source Tree

**What we are doing:** Removing any previous build artifacts so we start fresh.

```bash
make mrproper
```

If this is a brand-new clone, there is nothing to clean, and `mrproper` will run quickly and make no visible changes. The benefit is that running it is a safe habit — if you later re-use this source tree for a second build, `mrproper` ensures you are starting clean.

**Successful output:**
```
  CLEAN   scripts/basic
  CLEAN   scripts/kconfig
  CLEAN   include/config include/generated
  CLEAN   .config .config.old
```
(Or just finishes silently with a new clone.)

**Error to watch for:** If `make` is not found, your system is missing build tools:
```bash
sudo apt-get install make gcc build-essential bison flex libssl-dev libncurses-dev python3 python3-setuptools swig
```

---

### Step 5: Apply the Board Configuration

**What we are doing:** Generating the `.config` file from the board's defconfig.

```bash
make imx93_var_som_defconfig
```

**Successful output:**
```
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  HOSTCC  scripts/kconfig/confdata.o
  ...
  HOSTLD  scripts/kconfig/conf
#
# configuration written to .config
#
```

After this completes, a `.config` file exists at the top of the source tree. You can inspect it with `less .config` — it will be thousands of lines.

**Optional: Inspect or customize the configuration:**
```bash
make menuconfig
# Opens a text-based GUI for browsing/modifying config options
# Press / to search for a CONFIG_ symbol
# Press Q then Y to save and exit
```

**Error to watch for:** `make[1]: *** No rule to make target 'imx93_var_som_defconfig'. Stop.` — this means the defconfig file doesn't exist at `configs/imx93_var_som_defconfig`. Verify you are on the correct branch and the file is present:
```bash
ls configs/imx93_var_som*
```

---

### Step 6: Build U-Boot

**What we are doing:** Compiling U-Boot, producing the SPL binary and U-Boot proper binary.

```bash
make -j$(nproc)
```

This is the longest step. The build system will compile hundreds of C files, link them, and generate the final binaries.

**Watching the build:** You will see lines like:
```
  CC      arch/arm/cpu/armv8/cache_v8.o
  CC      arch/arm/cpu/armv8/cpu.o
  CC      board/variscite/imx93_var_som/imx93_var_som.o
  ...
  LD      u-boot
  OBJCOPY u-boot.bin
  ...
  LD      spl/u-boot-spl
  OBJCOPY spl/u-boot-spl.bin
  ...
  DTC     arch/arm/dts/imx93-var-som-symphony.dtb
  DTC     arch/arm/dts/imx93-var-dart-dt8mcustomboard.dtb
```

**Successful output ends with something like:**
```
  MKIMAGE u-boot-dtb.imx
  OBJCOPY spl/u-boot-spl.bin
```
And a `u-boot.bin` file should appear in the source root.

**Verify the key outputs exist:**
```bash
ls -lh u-boot.bin spl/u-boot-spl.bin tools/mkimage
ls -lh arch/arm/dts/imx93-var-som-symphony.dtb arch/arm/dts/imx93-var-dart-dt8mcustomboard.dtb
```
Expected sizes: `u-boot.bin` ~1–2 MB, `spl/u-boot-spl.bin` ~100–200 KB.

**Common errors:**

`cc1: error: unrecognized command-line option '-mno-unaligned-access'` — wrong compiler architecture. The toolchain environment was not sourced, or `ARCH` is not set to `arm64`. Re-source the environment.

`/usr/bin/ld: cannot find -lgcc` — you are accidentally using the host linker instead of the cross-linker. `CROSS_COMPILE` is not set. Re-source the environment.

`error: 'CONFIG_...' undeclared` — a required Kconfig symbol is missing. Try `make mrproper` then re-apply the defconfig.

---

### Step 7: Create the imx-boot-tools Directory

**What we are doing:** Creating a working directory where all firmware blobs will be assembled before running imx-mkimage.

```bash
cd ~/var-mx93
mkdir imx-boot-tools
cd imx-boot-tools
```

This directory will become the assembly area. All subsequent steps download into this directory, and `soc.mak` expects all its inputs to be present here.

---

### Step 8: Download and Extract the Sentinel Firmware

**What we are doing:** Obtaining the NXP EdgeLock Sentinel firmware container for i.MX93 revision A1.

```bash
wget https://www.nxp.com/lgfiles/NMG/MAD/YOCTO/firmware-ele-imx-2.0.3.1-52f7740
chmod +x firmware-ele-imx-2.0.3.1-52f7740
./firmware-ele-imx-2.0.3.1-52f7740
cp firmware-ele-imx-2.0.3.1-52f7740/mx93a1-ahab-container.img .
```

**Line by line:**

`wget <url>` — Downloads the file from NXP's server. The filename `firmware-ele-imx-2.0.3.1-52f7740` means: firmware for ELE (EdgeLock Enclave), version 2.0.3.1, short git hash `52f7740`.

`chmod +x firmware-ele-imx-2.0.3.1-52f7740` — Makes the downloaded file executable. By default, downloaded files don't have the execute bit set. This is a self-extracting archive (a shell script with appended binary data) — you need to execute it to extract the firmware.

`./firmware-ele-imx-2.0.3.1-52f7740` — Runs the self-extractor. **You will be prompted to accept NXP's Proprietary Software License Agreement.** Type `y` or `yes` and press Enter. The firmware is extracted into a new directory named `firmware-ele-imx-2.0.3.1-52f7740/`.

`cp firmware-ele-imx-2.0.3.1-52f7740/mx93a1-ahab-container.img .` — Copies the actual firmware blob to the current directory (`imx-boot-tools/`). The `.` at the end means "copy here."

**Verify:**
```bash
ls -lh mx93a1-ahab-container.img
# Should be ~50-200 KB
file mx93a1-ahab-container.img
# Should say: data (it's a binary container, not a regular ELF or script)
```

**Common errors:**
- `wget: ... 403 Forbidden` — NXP occasionally restricts direct downloads. Try opening the URL in a browser, accepting the EULA there, and downloading manually.
- Extraction hangs asking for license acceptance — read the prompt, type `y`, press Enter.

---

### Step 9: Download and Extract the DDR Firmware

**What we are doing:** Obtaining the Synopsys DDR PHY training firmware blobs for i.MX93.

```bash
wget https://www.nxp.com/lgfiles/NMG/MAD/YOCTO/firmware-imx-8.26.1-410be01.bin
chmod +x firmware-imx-8.26.1-410be01.bin
./firmware-imx-8.26.1-410be01.bin
cp firmware-imx-8.26.1-410be01/firmware/ddr/synopsys/* .
```

The same pattern as the Sentinel firmware: download a self-extracting binary, accept the EULA, and copy out the relevant files. The wildcard `*` in the last command copies all files from `firmware/ddr/synopsys/` into `imx-boot-tools/`.

**Verify the DDR firmware files are present:**
```bash
ls *.bin | grep -i lpddr
# You should see files like:
# lpddr4_imem_1d_*.bin
# lpddr4_dmem_1d_*.bin
# lpddr4_imem_2d_*.bin
# lpddr4_dmem_2d_*.bin
```

---

### Step 10: Clone and Build imx-mkimage

**What we are doing:** Obtaining NXP's image packaging tool, building it, and extracting the files we need.

```bash
git clone https://github.com/nxp-imx/imx-mkimage -b lf-6.6.52-2.2.2
cd imx-mkimage
make soc=iMX93
cp iMX93/soc.mak mkimage_imx8 ..
cd ..
```

**`git clone ... -b lf-6.6.52-2.2.2`** — Note this is the NXP upstream `imx-mkimage` repository (not the Variscite fork), on the matching BSP branch `lf-6.6.52-2.2.2`. The branch naming here has no `var01` suffix because imx-mkimage is a host-side packaging tool — Variscite doesn't typically fork it.

**`make soc=iMX93`** — Builds the `mkimage_imx8` binary and processes `iMX93/soc.mak`. The `soc=iMX93` variable tells the imx-mkimage Makefile to use i.MX93-specific configuration. This `make` runs with your **host compiler** (since the tool runs on your PC, not on ARM), but since you have the Yocto toolchain environment sourced, `CC` is set to the cross-compiler. The imx-mkimage Makefile handles this by using `HOSTCC` for the tool itself. If you see errors about missing host gcc, temporarily do `export CC=gcc` just for this build (the guide does this later in the `soc.mak` final build step by passing `CC=gcc`).

**`cp iMX93/soc.mak mkimage_imx8 ..`** — Copies two things up one level into `imx-boot-tools/`:
- `IiMX93/soc.mak` — the i.MX93-specific make rules for assembling the final image
- `mkimage_imx8` — the actual packaging tool binary

**Verify:**
```bash
ls -lh ../soc.mak ../mkimage_imx8
file ../mkimage_imx8
# Should say: ELF 64-bit LSB executable, x86-64  ← it's a host tool, not ARM
```

---

### Step 11: Clone Variscite's ATF Fork and Build BL31

**What we are doing:** Building ARM Trusted Firmware for i.MX93, producing `bl31.bin`.

```bash
cd ~/var-mx93/imx-boot-tools
git clone https://github.com/varigit/imx-atf -b lf_v2.10_6.6.52-2.2.2_var01
cd imx-atf
source /opt/fsl-imx-xwayland/6.6-scarthgap/environment-setup-armv8a-poky-linux
unset LDFLAGS
make PLAT=imx93 bl31
cp build/imx93/release/bl31.bin ..
cd ..
```

**`git clone ... -b lf_v2.10_6.6.52-2.2.2_var01`** — Clones Variscite's fork of NXP's ATF fork. The `v2.10` refers to TF-A (Trusted Firmware-A) upstream version 2.10.

**Re-source the toolchain** — You are in a new directory context and the guide recommends re-sourcing. It does not hurt to do this — it just re-exports the same variables.

**`unset LDFLAGS`** — As explained in Section 7.4, the Yocto toolchain sets `LDFLAGS` with `--sysroot` pointing to the Yocto target sysroot. ATF is freestanding firmware with its own linker scripts and does not use a sysroot. If you don't unset `LDFLAGS`, you will see linker errors like:
```
aarch64-poky-linux-ld: cannot find crti.o: No such file or directory
```
or similar. Unsetting it before the ATF build removes this interference.

**`make PLAT=imx93 bl31`** — Builds only the BL31 target for the `imx93` platform. The ATF Makefile uses `PLAT` to select platform-specific code from `plat/imx/imx93/`. The output is placed in `build/imx93/release/`.

**`cp build/imx93/release/bl31.bin ..`** — Copies the BL31 binary up into `imx-boot-tools/`.

**Verify:**
```bash
ls -lh ../bl31.bin
# Typical size: 30-70 KB
file ../bl31.bin
# ELF 64-bit or just "data" (it should be an AArch64 binary)
${CROSS_COMPILE}readelf -h ../bl31.bin 2>/dev/null | grep Machine
# Should say: AArch64  ← confirms it's an ARM binary, not x86
```

**Common error:** If you forget `unset LDFLAGS`:
```
/opt/fsl-imx-xwayland/.../aarch64-poky-linux-ld: cannot find crt*.o
```
Solution: `unset LDFLAGS` and run `make PLAT=imx93 bl31` again.

---

### Step 12: Copy All U-Boot Binaries into imx-boot-tools

**What we are doing:** Gathering the U-Boot outputs (built in Step 6) into the `imx-boot-tools/` assembly directory.

```bash
cd ~/var-mx93/imx-boot-tools
cp ../uboot-imx/tools/mkimage mkimage_uboot
cp ../uboot-imx/u-boot.bin .
cp ../uboot-imx/spl/u-boot-spl.bin \
   ../uboot-imx/arch/arm/dts/imx93-var-som-symphony.dtb \
   ../uboot-imx/arch/arm/dts/imx93-var-dart-dt8mcustomboard.dtb \
   .
```

**What each file is:**
- `mkimage_uboot` — U-Boot's own `mkimage` tool (renamed from `tools/mkimage` to avoid confusion with `mkimage_imx8`). This host tool creates U-Boot-format images (FIT images, legacy U-Boot headers). `soc.mak` uses it internally.
- `u-boot.bin` — U-Boot proper binary
- `u-boot-spl.bin` — SPL binary
- `imx93-var-som-symphony.dtb` — Device Tree Blob for Symphony carrier board
- `imx93-var-dart-dt8mcustomboard.dtb` — Device Tree Blob for DT8MCustomBoard

**Verify all required files are present:**
```bash
ls imx-boot-tools/
# Should contain (at minimum):
# bl31.bin
# mx93a1-ahab-container.img
# lpddr4_imem_1d_*.bin, lpddr4_dmem_1d_*.bin (DDR firmware)
# lpddr4_imem_2d_*.bin, lpddr4_dmem_2d_*.bin
# soc.mak
# mkimage_imx8
# mkimage_uboot
# u-boot.bin
# u-boot-spl.bin
# imx93-var-som-symphony.dtb
# imx93-var-dart-dt8mcustomboard.dtb
```

---

### Step 13: Build the Final Boot Image

**What we are doing:** Running imx-mkimage to assemble all components into a single bootable image.

```bash
cd ~/var-mx93/imx-boot-tools
make -f soc.mak clean
make -f soc.mak \
    SOC=iMX93 \
    dtbs="imx93-var-dart-dt8mcustomboard.dtb imx93-var-som-symphony.dtb" \
    MKIMG=./mkimage_imx8 \
    PAD_IMAGE=./pad_image.sh \
    CC=gcc \
    flash_singleboot
mv flash.bin imx-boot
```

**`make -f soc.mak clean`** — Cleans previous build artifacts from soc.mak. Note this `clean` is on the imx-mkimage packaging step, not the U-Boot source clean.

**Note on MKIMG path:** The guide originally shows `MKIMG=./imx-mkimage/mkimage_imx8` — this assumes you still have the `imx-mkimage/` subdirectory inside `imx-boot-tools/`. Since we copied `mkimage_imx8` directly to `imx-boot-tools/`, use `MKIMG=./mkimage_imx8`.

**`PAD_IMAGE=./pad_image.sh`** — This shell script is expected by `soc.mak` to pad binary images to alignment boundaries. It should have been included in the imx-mkimage clone. Find it:
```bash
ls ~/var-mx93/imx-boot-tools/imx-mkimage/pad_image.sh
cp ~/var-mx93/imx-boot-tools/imx-mkimage/pad_image.sh .
chmod +x pad_image.sh
```

**`CC=gcc`** — Override the cross-compiler CC with the host gcc for this step. The imx-mkimage tool and soc.mak build host-side utilities; they should use the host compiler.

**`flash_singleboot`** — The target. `soc.mak` will invoke `mkimage_imx8` multiple times with different arguments to create the image containers, pad them, and concatenate them into `flash.bin`.

**Watching the output:** You will see commands like:
```
./mkimage_imx8 -soc iMX93 -append mx93a1-ahab-container.img ...
./mkimage_imx8 -soc iMX93 -c -ap u-boot-spl.bin ...
...
```

**`mv flash.bin imx-boot`** — Renames the output to the conventional filename for the i.MX9 boot image.

**Verify:**
```bash
ls -lh imx-boot
# Typical size: 1.5 – 3.5 MB
```

**Common errors:**

`No rule to make target 'flash_singleboot'` — `soc.mak` is not in the current directory or was not copied correctly.

`./mkimage_imx8: Permission denied` — the binary isn't executable: `chmod +x mkimage_imx8`

`pad_image.sh: not found` — copy it from the imx-mkimage directory as described above.

`error: ... DDR firmware not found` — a DDR firmware blob is missing. Check that all `lpddr4_*.bin` files are in the current directory.

---

### Step 14: Write the Image to an SD Card

**What we are doing:** Flashing the boot image to the SD card so the i.MX93 can boot from it.

**First, identify your SD card device:**
```bash
lsblk
# Look for a device of the correct size (e.g., 8G, 16G, 32G)
# It will show as /dev/sdb or /dev/sdc etc.
# Also check: ls /dev/sd* before and after inserting the SD card to see which appears

# Or check dmesg:
dmesg | tail -20
# Look for lines like: [12345.678] sd 6:0:0:0: [sdb] 15126528 512-byte logical blocks
```

**Flash the image:**
```bash
sudo dd if=imx-boot of=/dev/sdb bs=1K seek=32 conv=fsync
```
(Replace `/dev/sdb` with your actual device.)

**Monitor progress:** On large images, `dd` can appear to hang. You can check progress in another terminal:
```bash
# Linux 4.0+ with dd progress:
sudo dd if=imx-boot of=/dev/sdb bs=1K seek=32 conv=fsync status=progress
```

**After dd completes:**
```bash
sync     # Extra safety flush
# Now safely eject the SD card
```

**Boot the board:** Insert the SD card into the VAR-SOM-MX93 carrier board with UART console connected (115200 8N1). Power on. You should immediately see SPL output:
```
U-Boot SPL 2024.04 (...)
Normal Boot
Trying to boot from MMC1
...
ATF BL31: ...
U-Boot 2024.04 (...)
```

If you see nothing at all, the SD card is not being found or the image at offset 32KB is not valid. Double-check the `of=/dev/sdX` path and that the image size is correct.

---

### Step 15: Optional — Copy to Recovery SD Card

If you use Variscite's recovery SD card (a pre-built SD card that can flash eMMC via a recovery script), you can copy your built `imx-boot` to it:

```bash
sudo cp imx-boot /media/rootfs/opt/images/...
```

The `...` path is specific to Variscite's recovery image layout. Check Variscite's [Yocto Recovery SD Card](https://dev.variscite.com/var-som-mx93/mx93-yocto-scarthgap-6.6.y_2.2.2-v1.0/yocto-recovery-sd-card/) guide for the exact path.

After copying, safely eject the SD card. When the recovery SD boots on the target board, it will use your custom-built U-Boot to flash to eMMC.

**Note on old U-Boot environments:** The guide mentions:
> If you manually upgrade an existing U-Boot, and you have an old environment saved, it is a good idea to reset your environment to the new default.

Do this from the U-Boot console (UART):
```
env default -a
saveenv
```
This discards any saved environment (in eMMC/NAND) and replaces it with the compiled-in defaults from your new U-Boot build.

---

## 12. Visual Boot Sequence Diagram

### i.MX93 Full Boot Sequence: Power-On to Linux

```
╔═══════════════════════════════════════════════════════════════════════╗
║              i.MX93 BOOT SEQUENCE — VAR-SOM-MX93                     ║
╚═══════════════════════════════════════════════════════════════════════╝

  POWER-ON RESET
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 0: Boot ROM (on-chip ROM, cannot be modified)                │
│                                                                     │
│  Privilege: EL3 (Secure World)                                      │
│  Runs from: Internal ROM (no external RAM needed)                   │
│  Purpose:                                                           │
│    • Read boot mode pins / fuses                                    │
│    • Initialize SD/eMMC controller (bare minimum)                  │
│    • Read image container header at offset 32KB of boot device      │
│    • Validate container header (AHAB, via Sentinel)                 │
│    • Load SPL into OCRAM (~256KB on-chip SRAM)                      │
│    • Transfer execution to SPL entry point                          │
│                                                                     │
│  Involved firmware: Boot ROM (silicon), Sentinel (ELE) firmware    │
│    for header validation                                            │
└─────────────────────────────────────────────────────────────────────┘
       │  (jumps to SPL in OCRAM)
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 1: SPL — Secondary Program Loader  (u-boot-spl.bin)         │
│                                                                     │
│  Privilege: EL3 → drops to EL2 for U-Boot                         │
│  Runs from: OCRAM (on-chip SRAM, ~256KB)                           │
│  Purpose:                                                           │
│    • Initialize PLL clock tree to full operating frequencies        │
│    • Load Synopsys DDR PHY firmware blobs into PHY internal memory  │
│    • Execute DDR training (calibrate signal timing for this PCB)    │
│    • Initialize LPDDR4/LPDDR4X DRAM (now hundreds of MB available) │
│    • Load U-Boot proper (u-boot.bin) into DDR                      │
│    • Load ATF BL31 (bl31.bin) into DDR (secure memory region)      │
│    • Jump to ATF BL31                                               │
│                                                                     │
│  Involved firmware: u-boot-spl.bin                                  │
│                     lpddr4_imem_1d/2d.bin (DDR training firmware)   │
│                     lpddr4_dmem_1d/2d.bin (DDR training firmware)   │
└─────────────────────────────────────────────────────────────────────┘
       │  (jumps to BL31 in DDR, EL3)
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 2: ATF BL31 — ARM Trusted Firmware                          │
│           (bl31.bin — Varigit fork of NXP fork of TF-A)            │
│                                                                     │
│  Privilege: EL3 (Secure World, highest privilege, stays resident)  │
│  Runs from: DDR (secure memory region, protected by TrustZone)     │
│  Purpose:                                                           │
│    • Set up TrustZone hardware security partitioning               │
│    • Configure which memory regions are Secure vs Non-Secure        │
│    • Register PSCI handlers (CPU hotplug, suspend/resume)           │
│    • Register SMC (Secure Monitor Call) dispatch table              │
│    • BL31 never truly "exits" — it stays resident at EL3           │
│    • Drops execution to U-Boot at EL2 via ERET instruction          │
│                                                                     │
│  Involved firmware: bl31.bin                                        │
│  Also concurrent: Sentinel (ELE) firmware continues running         │
│                   on its separate Cortex-M33, providing             │
│                   crypto services to the rest of the system         │
└─────────────────────────────────────────────────────────────────────┘
       │  (ERET to U-Boot at EL2)
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 3: U-Boot Proper  (u-boot.bin)                              │
│                                                                     │
│  Privilege: EL2 (Non-Secure World, hypervisor level)               │
│  Runs from: DDR (non-secure memory)                                 │
│  Purpose:                                                           │
│    • Full hardware initialization (USB, display, network, etc.)     │
│    • Initialize environment (from eMMC env partition or defaults)   │
│    • Select boot device and partition                               │
│    • Load Linux kernel (Image) into DDR                            │
│    • Load Device Tree Blob (.dtb) into DDR                         │
│    • Optionally load initramfs into DDR                             │
│    • Set up kernel boot arguments (bootargs env variable)           │
│    • Transfer to Linux kernel via 'booti' command                   │
│    • Can be interrupted at console for manual commands              │
│                                                                     │
│  Involved files: u-boot.bin                                         │
│                  imx93-var-som-symphony.dtb (or dt8m variant)       │
└─────────────────────────────────────────────────────────────────────┘
       │  (booti: jumps to kernel entry at EL1, passing DTB address)
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 4: Linux Kernel                                              │
│                                                                     │
│  Privilege: EL1 (Non-Secure World, OS kernel level)                │
│  Runs from: DDR (non-secure memory)                                 │
│  Purpose:                                                           │
│    • Decompress kernel if needed                                    │
│    • Parse Device Tree to discover hardware                         │
│    • Initialize all kernel subsystems and drivers                   │
│    • Mount root filesystem                                          │
│    • Start PID 1 (systemd or BusyBox init)                         │
│    • U-Boot is now dead — its memory gets reclaimed                │
│                                                                     │
│  Can call back to EL3 via SMC instructions to ATF BL31              │
│  for PSCI (power management) operations                             │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAGE 5: Userspace                                                 │
│                                                                     │
│  Privilege: EL0 (Non-Secure World, least privileged)               │
│  Runs from: DDR (user virtual address space, MMU active)           │
│  Examples: systemd, shell, your application code                   │
└─────────────────────────────────────────────────────────────────────┘


╔═══════════════════════════════════════════════════════════════════════╗
║  PRIVILEGE LEVEL SUMMARY                                             ║
╠═══════════════╦══════════════════════════╦═══════════════════════════╣
║  Level        ║  Component               ║  Notes                   ║
╠═══════════════╬══════════════════════════╬═══════════════════════════╣
║  EL3 Secure   ║  Boot ROM                ║  Silicon, immutable       ║
║  EL3 Secure   ║  ATF BL31 (resident)     ║  Stays in memory forever  ║
║  EL3 Secure   ║  SPL (briefly)           ║  Transitions to EL2       ║
║  EL3 Secure   ║  Sentinel (own CPU)      ║  Parallel, separate core  ║
╠═══════════════╬══════════════════════════╬═══════════════════════════╣
║  EL2 Non-Sec  ║  U-Boot Proper           ║  Hypervisor-level         ║
╠═══════════════╬══════════════════════════╬═══════════════════════════╣
║  EL1 Non-Sec  ║  Linux Kernel            ║  OS kernel level          ║
╠═══════════════╬══════════════════════════╬═══════════════════════════╣
║  EL0 Non-Sec  ║  Userspace Apps          ║  Least privileged         ║
╚═══════════════╩══════════════════════════╩═══════════════════════════╝


╔═══════════════════════════════════════════════════════════════════════╗
║  PHYSICAL MEMORY MAP DURING U-BOOT (after DDR init)                 ║
╠═══════════════════════════════════════════════════════════════════════╣
║  0x00000000  ─ 0x0007FFFF  OCRAM (512KB, SPL ran here)              ║
║  0x20000000  ─ 0x2001FFFF  ATF BL31 (secure SRAM, EL3 code)        ║
║  0x80000000  ─ ...          DDR start                                ║
║  0x80200000               U-Boot proper (CONFIG_SYS_TEXT_BASE)      ║
║  0x83000000               Linux kernel load address (typical)       ║
║  0x83000000+              DTB placed by U-Boot just above kernel    ║
║                                                                      ║
║  (exact addresses vary; see .config for CONFIG_SYS_TEXT_BASE        ║
║   and board-specific memory map)                                     ║
╚═══════════════════════════════════════════════════════════════════════╝


╔═══════════════════════════════════════════════════════════════════════╗
║  SD CARD / eMMC LAYOUT                                              ║
╠═══════════════════════════════════════════════════════════════════════╣
║  Offset 0 KB     MBR (512B) + GPT headers (~16KB)                  ║
║  Offset 0–32KB   Reserved (partition table area, NOT overwritten)   ║
║  Offset 32KB     ◄── imx-boot written here by dd seek=32 ──►       ║
║                  Contains (packed by imx-mkimage):                  ║
║                    • SPL container                                   ║
║                    • Sentinel AHAB container                        ║
║                    • ATF BL31                                       ║
║                    • DDR PHY firmware blobs                         ║
║                    • U-Boot proper                                  ║
║                    • Device Tree Blobs                              ║
║  Offset 1MB+     FAT partition: kernel Image, .dtb, etc.            ║
║  Further out     ext4 rootfs partition                               ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## Quick Reference: Files Produced and Where They Come From

```
imx-boot-tools/
├── bl31.bin                           ← from: varigit/imx-atf (build)
├── mx93a1-ahab-container.img          ← from: NXP ELE firmware download
├── lpddr4_imem_1d_*.bin               ← from: NXP DDR firmware download
├── lpddr4_dmem_1d_*.bin               ← from: NXP DDR firmware download
├── lpddr4_imem_2d_*.bin               ← from: NXP DDR firmware download
├── lpddr4_dmem_2d_*.bin               ← from: NXP DDR firmware download
├── soc.mak                            ← from: nxp-imx/imx-mkimage (copy)
├── mkimage_imx8                       ← from: nxp-imx/imx-mkimage (built)
├── mkimage_uboot                      ← from: varigit/uboot-imx tools/mkimage
├── u-boot.bin                         ← from: varigit/uboot-imx (built)
├── u-boot-spl.bin                     ← from: varigit/uboot-imx (built)
├── imx93-var-som-symphony.dtb         ← from: varigit/uboot-imx (built)
├── imx93-var-dart-dt8mcustomboard.dtb ← from: varigit/uboot-imx (built)
└── imx-boot                           ← FINAL OUTPUT (flash this to SD/eMMC)
```

---

*Document prepared for Variscite VAR-SOM-MX93, BSP Yocto Scarthgap 6.6.y_2.2.2-v1.0*
*Guide source: https://dev.variscite.com/var-som-mx93/mx93-yocto-scarthgap-6.6.y_2.2.2-v1.0/yocto-build-u-boot/*
