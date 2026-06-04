# Yocto Project — Interview Crash Course
> *Zero to Interview-Ready in a Few Hours*
> *Tailored for embedded systems engineers with Linux & driver background*

---

## SECTION 1 — What Is Yocto? (Simple Mental Model)

Yocto is **not a Linux distribution**. It is a **build system framework** that generates a custom embedded Linux distribution tailored to your hardware.

Think of it like this:

```
Your Hardware (e.g., STM32MP1, Qualcomm SA8155)
        +
Your Software Requirements (drivers, packages, init system)
        +
Yocto Build System
        =
A complete, minimal, production-ready Linux image
```

**Why not just use Ubuntu/Debian?**
- Too bloated for constrained embedded targets
- No control over what's included
- Not reproducible across teams
- No cross-compilation workflow built in

**Yocto gives you:**
- Reproducible builds
- Full control over every package in the rootfs
- Cross-compilation toolchain generation
- BSP abstraction for different SoCs
- SDK generation for application developers

### The Poky Reference Distribution

**Poky** = Reference implementation of a Yocto-based distribution. It includes:
- BitBake (the build engine)
- OpenEmbedded-Core (OE-Core) layer with base recipes
- A set of scripts and configuration files

Think of Poky as the "template project" you clone to get started.

---

## SECTION 2 — Core Terminology (Quick Reference)

| Term | What It Is | Analogy |
|---|---|---|
| **BitBake** | The build engine/task scheduler | GNU Make on steroids |
| **Recipe (.bb)** | Instructions to build one software package | A Makefile for one package |
| **Layer** | A collection of recipes, organized by purpose | A Git repo / plugin |
| **BSP Layer** | Board Support Package — hardware-specific recipes | Platform HAL |
| **Metadata** | All recipes, config files, classes — everything BitBake processes | Source code for the build |
| **Image** | The final rootfs + kernel assembled for flashing | Output binary |
| **Distro** | A policy layer — defines init system, libc, features | A Linux distro config |
| **Machine** | Hardware target definition (CPU arch, flash layout) | A board config |
| **SDK** | Toolchain + sysroot for app development off-target | NDK / cross-toolchain |
| **Sysroot** | Headers + libraries for cross-compilation | `/usr` for your target |
| **Class (.bbclass)** | Reusable build logic inherited by recipes | A base class / mixin |
| **Configuration (.conf)** | Variables that control builds globally | Environment variables |
| **Append File (.bbappend)** | Modifies an existing recipe without touching it | Monkey-patching |
| **TMPDIR** | Where build artifacts are stored | `build/` directory |
| **Workdir** | Per-recipe working directory inside TMPDIR | Per-package build dir |

---

## SECTION 3 — Yocto Architecture & Build Workflow

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Developer Input                       │
│  local.conf (MACHINE, DISTRO)  bblayers.conf (layers)  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                     BitBake Engine                       │
│  - Parses all metadata (recipes, classes, configs)       │
│  - Resolves dependencies (build a DAG)                   │
│  - Executes tasks in order (fetch→unpack→patch→         │
│    configure→compile→install→package→image)              │
└────────────────────────┬────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     ┌─────────┐   ┌──────────┐  ┌──────────────┐
     │  Layer  │   │  Layer   │  │  BSP Layer   │
     │ meta    │   │meta-oe   │  │meta-qualcomm │
     │(OE-Core)│   │          │  │              │
     └─────────┘   └──────────┘  └──────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    Build Outputs                          │
│  - Root Filesystem (rootfs)                              │
│  - Kernel Image (zImage / Image.gz)                      │
│  - Device Tree Blobs (.dtb)                              │
│  - Bootloader (u-boot)                                   │
│  - SDK / Toolchain                                       │
│  - Package feeds (.rpm / .deb / .ipk)                    │
└─────────────────────────────────────────────────────────┘
```

### Build Task Pipeline (per recipe)

```
do_fetch → do_unpack → do_patch → do_configure → do_compile
    → do_install → do_package → do_package_write → do_rootfs
```

Each `do_*` step is a **task** that BitBake can run, cache, and re-run independently.

### The Two Key Config Files

**`conf/local.conf`** — Your personal build settings:
```
MACHINE = "qemux86-64"          # Target hardware
DISTRO = "poky"                 # Linux distro policy
BB_NUMBER_THREADS = "8"        # Parallel tasks
PARALLEL_MAKE = "-j8"          # Make parallelism
DL_DIR = "/downloads"          # Shared download cache
SSTATE_DIR = "/sstate-cache"   # Shared build cache
```

**`conf/bblayers.conf`** — Which layers are active:
```
BBLAYERS ?= " \
  /home/user/poky/meta \
  /home/user/poky/meta-poky \
  /home/user/meta-openembedded/meta-oe \
  /home/user/meta-myboard \
"
```

---

## SECTION 4 — Important Components Deep Dive

### 4.1 BitBake

BitBake is the task execution engine. Key things to know:

- Reads all `.bb`, `.bbappend`, `.conf`, `.bbclass` files
- Builds a **dependency graph** of all tasks
- Executes tasks in parallel where possible
- Uses a **shared state cache (sstate)** to avoid rebuilding unchanged tasks

**BitBake variable expansion:**
```
SRC_URI = "https://example.com/${PN}-${PV}.tar.gz"
# PN = Package Name, PV = Package Version (auto-set from recipe filename)
```

### 4.2 Recipes (.bb files)

A recipe tells BitBake how to build one software component.

**Naming convention:** `<name>_<version>.bb`
Example: `busybox_1.35.0.bb`

**Minimal recipe structure:**
```bitbake
SUMMARY = "My custom application"
DESCRIPTION = "Does something useful on the target"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://LICENSE;md5=abc123"

SRC_URI = "git://github.com/user/myapp.git;branch=main"
SRCREV = "deadbeefdeadbeef1234"

S = "${WORKDIR}/git"          # Source directory

inherit cmake                  # Use CMake class

EXTRA_OECMAKE = "-DENABLE_FEATURE=ON"

do_install() {
    install -d ${D}${bindir}
    install -m 0755 myapp ${D}${bindir}/
}
```

**Key variables:**
| Variable | Meaning |
|---|---|
| `PN` | Package name (auto from filename) |
| `PV` | Package version (auto from filename) |
| `S` | Source directory |
| `D` | Destination directory (staging rootfs) |
| `WORKDIR` | Recipe working directory |
| `SRC_URI` | Where to fetch source from |
| `DEPENDS` | Build-time dependencies |
| `RDEPENDS` | Runtime dependencies |
| `FILES` | What files go into the package |
| `inherit` | Include a bbclass |

### 4.3 Layers

A layer is a structured directory containing related metadata.

**Standard layer structure:**
```
meta-mylayer/
├── conf/
│   └── layer.conf          ← Layer registration file (required)
├── recipes-core/
│   └── myapp/
│       └── myapp_1.0.bb
├── recipes-bsp/
│   └── u-boot/
│       └── u-boot_%.bbappend
├── classes/
│   └── myclass.bbclass
└── README
```

**layer.conf (minimum required):**
```
BBPATH .= ":${LAYERDIR}"
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"
BBFILE_COLLECTIONS += "mylayer"
BBFILE_PATTERN_mylayer = "^${LAYERDIR}/"
LAYERPRIORITY_mylayer = "6"
```

**Layer priority** determines which recipe wins when multiple layers define the same recipe. Higher number = higher priority.

### 4.4 BSP Layer

A BSP (Board Support Package) layer contains everything hardware-specific:

- Machine configuration (`.conf` in `conf/machine/`)
- Kernel recipe or append (patches, defconfig, device trees)
- Bootloader recipe (u-boot customization)
- Firmware blobs (DSP firmware, GPU firmware)

**Machine config example (`conf/machine/myboard.conf`):**
```
TARGET_ARCH = "arm"
DEFAULTTUNE = "cortexa53"
SOC_FAMILY = "qcom"

KERNEL_IMAGETYPE = "Image.gz"
KERNEL_DEVICETREE = "qcom/myboard.dtb"

SERIAL_CONSOLE = "115200;ttyMSM0"

IMAGE_FSTYPES = "ext4 wic"

UBOOT_MACHINE = "myboard_defconfig"
```

**For Qualcomm/ARM:** You'd look for `meta-qcom` or vendor-specific BSP layers that define things like:
- Hexagon DSP firmware integration
- Secure boot chain (ABL/XBL)
- QSPI/eMMC flash layout in the `.wks` file

### 4.5 Classes (.bbclass)

Reusable build logic. Common ones you'll encounter:

| Class | Purpose |
|---|---|
| `autotools` | For packages using ./configure + make |
| `cmake` | For CMake-based packages |
| `meson` | For Meson build system |
| `kernel` | For Linux kernel recipes |
| `module` | For out-of-tree kernel modules |
| `systemd` | For adding systemd service units |
| `image` | For image recipes |
| `sdk` | For SDK generation |
| `pkgconfig` | pkg-config integration |
| `useradd` | Add users/groups to rootfs |

**Using a class:**
```bitbake
inherit autotools pkgconfig systemd
```

### 4.6 Images

An image recipe assembles the final rootfs. Common built-in images:

| Image | Contents |
|---|---|
| `core-image-minimal` | Bare minimum — busybox shell, init |
| `core-image-base` | Minimal + some utilities |
| `core-image-full-cmdline` | Full command line tools |
| `core-image-sato` | Full GUI (GNOME/GTK) |
| `core-image-weston` | Wayland/Weston desktop |

**Custom image recipe:**
```bitbake
require recipes-core/images/core-image-minimal.bb

SUMMARY = "My product image"

IMAGE_INSTALL += " \
    myapp \
    openssh \
    python3 \
    can-utils \
    mosquitto \
"

IMAGE_FEATURES += "ssh-server-openssh debug-tweaks"
```

**IMAGE_FEATURES** — high-level switches:
- `debug-tweaks` — empty root password, allow root login
- `ssh-server-openssh` — install and enable OpenSSH
- `package-management` — include package manager in image
- `read-only-rootfs` — mount rootfs as read-only

### 4.7 SDK

The Yocto SDK provides a cross-compilation environment for application developers.

**Two types:**
1. **Standard SDK** — Cross toolchain + target sysroot
2. **Extensible SDK (eSDK)** — Includes BitBake, can build new recipes

**Generate SDK:**
```bash
bitbake core-image-minimal -c populate_sdk
```

**Install and use SDK:**
```bash
# Install
./tmp/deploy/sdk/poky-glibc-x86_64-core-image-minimal-aarch64-toolchain-3.4.sh

# Source the environment
source /opt/poky/3.4/environment-setup-aarch64-poky-linux

# Now cross-compile
$CC -o myapp myapp.c
aarch64-poky-linux-gcc --sysroot=$SDKTARGETSYSROOT -o myapp myapp.c
```

### 4.8 Package Management

Yocto supports three package formats:
- **IPK** (default for embedded) — lightweight, for opkg
- **RPM** — used in fedora-based systems
- **DEB** — Debian-compatible

Set in `local.conf`:
```
PACKAGE_CLASSES = "package_ipk"
```

---

## SECTION 5 — Common Commands

### Initial Setup
```bash
# Clone Poky
git clone -b kirkstone https://git.yoctoproject.org/poky
cd poky

# Initialize the build environment
source oe-init-build-env [build-dir]
# This creates/enters build/ directory and sets up PATH
```

### Building
```bash
# Build an image
bitbake core-image-minimal

# Build a specific recipe
bitbake busybox

# Build only a specific task of a recipe
bitbake busybox -c compile

# Force a task to re-run (even if cached)
bitbake busybox -c compile -f

# Clean a recipe
bitbake busybox -c clean

# Clean + remove shared state for a recipe
bitbake busybox -c cleansstate

# Clean everything (nuclear option)
bitbake -c cleanall core-image-minimal
```

### Inspecting & Debugging
```bash
# Show all variables for a recipe (very useful in interviews!)
bitbake -e busybox | grep ^SRC_URI

# Show the task dependency graph
bitbake -g core-image-minimal
# Produces task-depends.dot — visualize with graphviz

# Devshell — interactive shell inside recipe's build environment
bitbake busybox -c devshell

# List all available recipes
bitbake -s

# Show recipe info
bitbake -e myrecipe | less

# List tasks for a recipe
bitbake myrecipe -c listtasks

# Search for which recipe provides a package
bitbake -s | grep busybox
```

### Layer Management
```bash
# List active layers
bitbake-layers show-layers

# Add a layer
bitbake-layers add-layer ../meta-mylayer

# Show which recipe is being used (and from which layer)
bitbake-layers show-recipes | grep myapp

# Find which layer overrides a recipe
bitbake-layers show-overrides
```

### Workspace / devtool (eSDK workflow)
```bash
# Create a new recipe for an existing source
devtool add myapp /path/to/source

# Modify an existing recipe (extracts to workspace)
devtool modify busybox

# Build with devtool
devtool build myapp

# Deploy directly to a running target over SSH
devtool deploy-target myapp root@192.168.1.100

# Finish — move recipe back to your layer
devtool finish myapp ../meta-mylayer

# Update a recipe to a new upstream version
devtool upgrade myapp
```

### Output Locations
```bash
tmp/deploy/images/<MACHINE>/     # Kernel, rootfs, DTB, bootloader
tmp/deploy/sdk/                  # SDK installers
tmp/deploy/ipk/                  # Individual packages
tmp/work/<arch>/<recipe>/<ver>/  # Per-recipe workdir (for debugging)
```

---

## SECTION 6 — Common Interview Questions & Answers

### Q1: What is Yocto and how is it different from Buildroot?

**Answer:** Yocto is a build framework that generates a custom embedded Linux distribution. Unlike Buildroot, which is simpler and generates a single rootfs with no package management, Yocto:
- Supports package management (ipk/rpm/deb) in the final image
- Has a more sophisticated layer-based architecture for reusability
- Generates proper SDKs for application development
- Is more complex but scales better for large commercial products
- Has a huge ecosystem of vendor BSP layers (TI, NXP, Qualcomm, etc.)

### Q2: What is BitBake and what does it do?

**Answer:** BitBake is the task scheduler and build engine of Yocto. It reads all the metadata (recipes, classes, config files), resolves dependencies by building a directed acyclic graph (DAG), and executes tasks in the correct order with maximum parallelism. It also implements the sstate (shared state) cache mechanism to avoid redundant rebuilds.

### Q3: What is a recipe and what are its key components?

**Answer:** A recipe (`.bb` file) describes how to build a single software package. Key components:
- `SRC_URI` — where to fetch the source (git, http, local)
- `SRCREV` or checksum — pinning the exact version
- `DEPENDS` / `RDEPENDS` — build-time and runtime dependencies
- `inherit` — which build classes to use (autotools, cmake, etc.)
- `do_compile`, `do_install` — task overrides if needed
- `FILES` — what files get packaged

### Q4: What is a layer and why do we use them?

**Answer:** A layer is a structured directory containing related recipes, configuration, and classes. Layers enable modular, reusable builds. For example, a BSP layer from a silicon vendor contains all the hardware-specific recipes. Your application layer contains your product-specific code. You mix layers from multiple sources without modifying them, using `.bbappend` files to customize. This keeps upstream layers clean and upgradeable.

### Q5: What is the difference between DEPENDS and RDEPENDS?

**Answer:** 
- `DEPENDS` — build-time dependency. The listed recipe must be built *before* this recipe starts compiling. Example: a recipe that links against `libssl` needs `DEPENDS = "openssl"`.
- `RDEPENDS` — runtime dependency. The package must be *present on the target* at runtime. Example: a Python script package has `RDEPENDS_${PN} = "python3"`.

### Q6: What is the shared state (sstate) cache?

**Answer:** The sstate cache is Yocto's binary cache of task outputs. When a task completes successfully, its output is stored as a `.tgz` in the sstate directory, keyed by a hash of all inputs (source code, recipe variables, toolchain). On the next build, if the hash matches, the task output is restored from cache instead of re-executing. This drastically speeds up incremental builds and enables sharing the cache across a team (via a shared NFS or HTTP sstate mirror).

### Q7: How do you add a package to an image?

**Answer:** Two main ways:
1. Add it to `IMAGE_INSTALL` in the image recipe or `local.conf`:
   ```
   IMAGE_INSTALL:append = " mypackage openssh"
   ```
2. Add it via `IMAGE_FEATURES` if there's a feature that includes it.

Note the `:append` operator — this is the correct Honister/Kirkstone syntax (older releases use `_append`).

### Q8: What is a .bbappend file and when do you use it?

**Answer:** A `.bbappend` file modifies an existing recipe from another layer without editing the original. This is critical for maintainability — you can customize a vendor recipe (like `u-boot` or `linux-yocto`) without forking it. The filename must match the original recipe name. Use `%` as a wildcard for version matching:

```
# u-boot_%.bbappend — matches any u-boot version
SRC_URI:append = " file://my-board.patch"
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
```

### Q9: How does Yocto handle the Linux kernel?

**Answer:** The kernel is built by a kernel recipe (e.g., `linux-yocto`, or a vendor recipe like `linux-qcom`). You typically don't edit the kernel recipe directly — instead you:
- Use a `.bbappend` to add patches via `SRC_URI:append`
- Provide a `defconfig` or use `KBUILD_DEFCONFIG`
- Use `KERNEL_DEVICETREE` in the machine config to specify which DTBs to build
- Apply configuration fragments (`.cfg` files) that get merged with the main defconfig

```bitbake
# linux-myvendor_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI:append = " file://enable-can.cfg \
                   file://0001-my-driver.patch"
```

### Q10: What is MACHINE and DISTRO?

**Answer:**
- `MACHINE` defines the hardware target — CPU architecture, kernel image type, device trees, serial console, flash layout. Set in `local.conf` or passed as an environment variable.
- `DISTRO` defines the distribution policy — which init system (systemd vs sysvinit), which C library (glibc vs musl), compiler flags, enabled features. `poky` is the default distro. Companies create their own distro layer for product-specific policies.

### Q11: Explain IMAGE_FEATURES vs IMAGE_INSTALL.

**Answer:**
- `IMAGE_INSTALL` directly lists packages to include in the rootfs.
- `IMAGE_FEATURES` are high-level switches that expand into packages + configuration. For example, `IMAGE_FEATURES += "ssh-server-openssh"` not only installs OpenSSH but also enables the service on first boot.

### Q12: How do you debug a failing recipe?

**Answer:**
1. Read the error message — it shows the exact task that failed and log location
2. Check `tmp/work/<arch>/<recipe>/<ver>/temp/log.do_<task>` for the full task log
3. Use `bitbake <recipe> -c devshell` to get an interactive shell in the exact build environment with all variables set
4. Use `bitbake -e <recipe>` to dump all variables and verify they're set correctly
5. Check if the issue is a missing DEPENDS, wrong SRCREV, or a patch that doesn't apply

---

## SECTION 7 — Real-World Embedded Linux Examples

### Example 1: Adding a CAN bus utility to your image

```bitbake
# In your image recipe or local.conf:
IMAGE_INSTALL:append = " can-utils iproute2"

# In your kernel bbappend, enable CAN:
# enable-can.cfg:
CONFIG_CAN=y
CONFIG_CAN_SOCKETCAN=y
CONFIG_CAN_MCP251X=y
```

### Example 2: Custom out-of-tree kernel module recipe

```bitbake
# recipes-kernel/my-module/my-module_1.0.bb
SUMMARY = "Custom CAN-FD driver module"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://COPYING;md5=..."

inherit module

SRC_URI = "git://github.com/myorg/my-canfd-driver.git;branch=main"
SRCREV = "abcdef1234"

S = "${WORKDIR}/git"

EXTRA_OEMAKE = "KDIR=${STAGING_KERNEL_BUILDDIR}"
```

### Example 3: Adding a systemd service

```bitbake
# recipes-apps/myapp/myapp_1.0.bb
SUMMARY = "My embedded application"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://LICENSE;md5=..."

SRC_URI = " \
    git://github.com/myorg/myapp.git;branch=main \
    file://myapp.service \
"
SRCREV = "abc123"
S = "${WORKDIR}/git"

inherit cmake systemd

SYSTEMD_SERVICE:${PN} = "myapp.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_install:append() {
    install -d ${D}${systemd_unitdir}/system
    install -m 0644 ${WORKDIR}/myapp.service \
        ${D}${systemd_unitdir}/system/
}
```

### Example 4: Qualcomm-specific considerations

In Qualcomm (QCOM) BSP layers (`meta-qcom`):
- `MACHINE = "qcom-armv8a"` or board-specific like `"sa8155p-adp"`
- Proprietary firmware (GPU, DSP, WiFi) handled via `qcom-firmware` recipes with `LICENSE = "Proprietary"`
- Fastboot/ADB integration via `android-tools` recipe
- Hexagon SDK tools bridged via special recipes
- Secure boot chain: XBL → ABL → u-boot/GRUB → kernel (may have restricted recipes)
- `IMAGE_FSTYPES = "ext4 squashfs wic"` — wic for full flash image with partition table

### Example 5: Creating a .wic (flashable image with partitions)

```
# myboard.wks (Wic Kickstart file)
part /boot --source bootimg-efi --label boot --active --align 4
part / --source rootfs --fstype=ext4 --label root --align 4096
part /data --fstype=ext4 --label data --size 512M --align 4096
```

```bitbake
# In machine conf:
WKS_FILE = "myboard.wks"
IMAGE_FSTYPES += "wic wic.bz2"
```

---

## SECTION 8 — Advanced Interview Questions

### A1: Explain the BitBake parsing process and variable scoping.

**Answer:** BitBake has a multi-phase parsing process:
1. **Configuration parsing** — reads `bblayers.conf`, then `layer.conf` for each layer, then `local.conf` and `site.conf`
2. **Recipe parsing** — for each recipe in BBFILES, parses the `.bb` and all matching `.bbappend` files
3. **Variable expansion** — lazy (`:=` is immediate, `=` is deferred)

Variable scoping: Variables in a recipe are recipe-scoped. Package-specific variables use `_${PN}` suffix (or `:${PN}` in newer syntax). Global variables from `.conf` files are inherited by recipes.

**Operator precedence (important!):**
- `=` — soft assignment (can be overridden)
- `:=` — immediate expansion (evaluated right now)
- `?=` — default (only if not already set)
- `??=` — weak default
- `:append` / `:prepend` — unconditionally appended/prepended (applied after all other assignments)

### A2: How does the sstate cache invalidation work?

**Answer:** Each task has a hash (called the **task signature hash** or `sigdata`). This hash is computed from:
- The recipe's variables relevant to that task
- The task code itself
- The sstate hashes of all upstream dependency tasks
- The checksums of any files referenced (patches, source files)

If ANY of these change, the hash changes and the cache is invalidated. `bitbake-diffsigs` can compare two sigdata files to find exactly what changed. This is critical for debugging "why is this rebuilding?"

### A3: How would you port Yocto to a new, unsupported board?

**Answer:** Create a BSP layer:
1. `bitbake-layers create-layer meta-myboard`
2. Create `conf/machine/myboard.conf` with TARGET_ARCH, kernel image type, DTB names, serial console, IMAGE_FSTYPES
3. Create a kernel recipe append with your defconfig and device tree
4. Create a bootloader recipe append if needed
5. Add any proprietary firmware recipes
6. Add the layer to `bblayers.conf`
7. Set `MACHINE = "myboard"` in `local.conf`
8. Build and iterate — typically starting with `core-image-minimal`

### A4: What is multilib and when do you use it?

**Answer:** Multilib allows building packages for multiple ABIs simultaneously. Common use case: a 64-bit ARM (aarch64) target that needs to run some 32-bit ARMv7 binaries (legacy applications). You configure multilib in `local.conf`:
```
require conf/multilib.conf
MULTILIBS = "multilib:lib32"
DEFAULTTUNE_virtclass-multilib-lib32 = "armv7thf-neon"
```
This is common in Android-adjacent embedded Linux platforms (where 32-bit ARM middleware exists alongside 64-bit apps).

### A5: How does Yocto handle licensing and license compliance?

**Answer:** Every recipe must declare `LICENSE` and `LIC_FILES_CHKSUM`. Yocto:
- Validates the checksum of the actual license file at build time
- Generates a `license.manifest` in the image deploy directory listing all package licenses
- Can generate a source archive for GPL compliance via `archiver` class
- `INCOMPATIBLE_LICENSE` variable lets you fail the build if any package has a forbidden license (e.g., block GPL-3.0 in proprietary products)

This is a real concern in embedded products — interviewers at automotive/medical companies often ask this.

### A6: What is the difference between `do_install` and `do_populate_sysroot`?

**Answer:** 
- `do_install` installs files into `${D}` (the package staging area / fake rootfs). This is what goes into the final image package.
- `do_populate_sysroot` (automatically derived from `do_install`) copies the *development files* (headers, `.so` symlinks, pkg-config files) from `${D}` into the shared sysroot where other recipes can find them for cross-compilation.

You normally don't override `do_populate_sysroot` — you control what gets sysrooted via `SYSROOT_DIRS`.

### A7: How would you share a sstate cache across a CI system?

**Answer:** Configure a central sstate mirror in `local.conf` or `site.conf`:
```
SSTATE_MIRRORS ?= "file://.* http://ci-server/sstate/PATH"
```
CI uploads new sstate artifacts after successful builds. Developers and other CI jobs download from this mirror. This means developers never rebuild what CI already built. Some teams also use `DL_DIR` on a shared NFS mount for download cache sharing.

### A8: What is devtool and when is it preferable to editing recipes directly?

**Answer:** `devtool` is an extensible SDK workflow tool that:
- Creates a **workspace layer** that takes priority over other layers
- Extracts recipe sources into a proper git repo for easy modification
- Lets you `devtool deploy-target` directly to hardware over SSH for rapid iteration
- Converts your changes back into patches with `devtool finish`

Preferable for: actively developing new code, debugging application builds, rapid prototyping. Less suitable for: production recipe authoring, complex multi-patch kernel work.

### A9: How do recipe overrides and OVERRIDES variable work?

**Answer:** `OVERRIDES` is a colon-separated list of override keys. BitBake uses it to select variable values conditionally:

```bitbake
OVERRIDES = "arm:aarch64:poky:myboard"

# This sets SRC_URI only when MACHINE is "myboard"
SRC_URI:myboard:append = " file://board-specific.patch"

# This sets a variable only for aarch64
EXTRA_CFLAGS:aarch64 = "-mfix-cortex-a53-835769"
```

Overrides replace the old `_<key>` suffix syntax in Kirkstone+. They allow architecture-specific, machine-specific, or distro-specific variable settings without if/else logic.

---

## SECTION 9 — Minimum Knowledge to Not Get Exposed

### If you claim "basic Yocto experience," you must be solid on:

1. **You can explain the build workflow end-to-end**: clone poky → `source oe-init-build-env` → set `MACHINE` in `local.conf` → `bitbake core-image-minimal` → output lands in `tmp/deploy/images/`

2. **You understand layers**: you've worked with a BSP layer (even if just consuming one), you know what `bblayers.conf` does, and you can explain why you'd create a custom layer rather than editing meta/

3. **You can write a basic recipe**: you know `SRC_URI`, `SRCREV`, `inherit`, `DEPENDS`, `RDEPENDS`, and `do_install` — even if you haven't memorized all syntax

4. **You know `.bbappend`**: this is almost always used in real projects — you add kernel patches, modify u-boot configs, customize package installs

5. **You can debug a build failure**: check the log in `tmp/work/`, use `bitbake -e`, run `devshell`

6. **You know IMAGE_INSTALL and IMAGE_FEATURES**: the two main ways to customize what goes into the image

7. **Know the Yocto release names**: Kirkstone (LTS), Dunfell (LTS), Honister, Langdale, Scarthgap — interviewers often ask which release you've used. Safe answer: "I've worked with Kirkstone."

8. **Don't claim to have written a BSP from scratch** if you haven't — claim "I've used and customized BSP layers provided by the silicon vendor"

### Red flags that expose inexperience:
- Not knowing the difference between DEPENDS and RDEPENDS
- Confusing `do_install` with installing to the host system
- Not knowing what `${D}` or `${S}` mean
- Claiming Yocto IS a Linux distro
- Not knowing what a `.bbappend` is

---

## SECTION 10 — One-Page Revision Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
               YOCTO QUICK REVISION SHEET
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

WHAT IS YOCTO: Build framework, not a distro. Generates
custom embedded Linux. Poky = reference implementation.

KEY COMPONENTS
  BitBake    = Build engine (like Make but smarter)
  Recipe     = How to build ONE package (.bb file)
  Layer      = Collection of recipes/config (git repo)
  BSP Layer  = Hardware-specific layer (machine conf,
               kernel, bootloader, firmware)
  Class      = Reusable logic (.bbclass, inherited)
  Image      = Final assembled rootfs recipe
  sstate     = Binary cache of task outputs

WORKFLOW
  source oe-init-build-env → edit local.conf →
  edit bblayers.conf → bitbake core-image-minimal

TASK PIPELINE
  fetch→unpack→patch→configure→compile→install→package

KEY VARIABLES
  MACHINE   = target hardware
  DISTRO    = distro policy (init, libc, flags)
  SRC_URI   = source location
  S         = source dir, D = destination (staging)
  DEPENDS   = build-time dep, RDEPENDS = runtime dep
  IMAGE_INSTALL += " mypkg" → add pkg to image
  IMAGE_FEATURES += "ssh-server-openssh debug-tweaks"

KEY COMMANDS
  source oe-init-build-env build/
  bitbake core-image-minimal
  bitbake <recipe> -c <task>      # run specific task
  bitbake <recipe> -c cleansstate # clean with cache
  bitbake -e <recipe>             # dump all variables
  bitbake-layers show-layers
  devtool modify <recipe>          # workspace edit
  devtool deploy-target app root@IP

RECIPE SKELETON
  SRC_URI = "git://...;branch=main"
  SRCREV = "deadbeef"
  inherit cmake
  DEPENDS = "openssl"
  RDEPENDS:${PN} = "python3"
  do_install() { install -m 0755 bin ${D}${bindir}/ }

BBAPPEND PATTERN
  FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
  SRC_URI:append = " file://mypatch.patch"

OVERRIDES SYNTAX (Kirkstone+)
  VAR:append = " value"    # old: VAR_append
  VAR:machine_name = "x"   # machine-specific

IMPORTANT DIRS
  tmp/deploy/images/<MACHINE>/   # kernel, rootfs, dtb
  tmp/work/<arch>/<pkg>/<ver>/   # per-recipe workdir
  tmp/work/.../temp/log.do_*     # task logs

LTS RELEASES: Dunfell (3.1), Kirkstone (4.0), Scarthgap
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## SECTION 11 — Yocto + Your Embedded Background (Talking Points)

Since you have embedded systems + Linux + driver experience, frame your answers around these connecting points in the interview:

- **Device drivers**: "In Yocto, I'd use a kernel module recipe with `inherit module`. Out-of-tree drivers go in their own recipe with `EXTRA_OEMAKE = 'KDIR=${STAGING_KERNEL_BUILDDIR}'`"

- **Cross-compilation**: "The Yocto SDK gives me `$CC`, `$CFLAGS`, and a sysroot — same as a manually set up cross-toolchain but fully automated and reproducible"

- **Qualcomm platforms**: "Qualcomm provides `meta-qcom` BSP layers. The secure boot chain (XBL→ABL→kernel) means the flash image layout in the `.wks` file must match the partition table expected by the bootloader"

- **CAN/automotive**: "I'd add kernel config fragments for `CONFIG_CAN_SOCKETCAN`, package `can-utils`, and write a systemd service recipe to configure the CAN interface at boot"

- **MQTT/IoT**: "Mosquitto has a recipe in `meta-openembedded`. I'd add it to IMAGE_INSTALL and use a `.bbappend` to drop in our custom `mosquitto.conf`"

---

*Good luck in your interview!*
