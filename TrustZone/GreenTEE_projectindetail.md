# Green TEE — Complete Interview Preparation

## Resume Entry

**ARM TrustZone TEE Implementation & Debugging (Green TEE)**
Built and debugged a complete ARM TrustZone Trusted Execution Environment stack from source on QEMU, spanning TF-A boot firmware (BL1–BL31), a custom S-EL1 Trusted OS, Linux kernel TEE driver, and userspace client — diagnosing and fixing EL3 context management bugs, SMP context registration failures, and cross-world SMC calling convention issues.

---

## What This Project Is

Green TEE is a minimal, educational Trusted Execution Environment written from scratch. It demonstrates the complete TrustZone software stack:

- **TF-A (ARM Trusted Firmware)** — BL1, BL2, BL31 boot chain + green_teed SPD at EL3
- **Green TEE Core** — a bare-metal Trusted OS running at S-EL1
- **Linux Kernel Driver** — character device `/dev/green_tee` at NS-EL1
- **Userspace Client** — applications at NS-EL0 that request secure services
- **Three secure services** — PRINT (cross-world string), ENCRYPT (OTP), DECRYPT (OTP)

---

## The Complete Flow — What Actually Happens Internally

### Phase 1: Boot (Power ON → Linux prompt)

```
Power ON
  │
  ▼ EL3, Secure state
BL1 (arm-trusted-firmware/bl1/)
  - First code after reset vector
  - Sets up EL3 exception vectors (VBAR_EL3)
  - Initialises SRAM, minimal clocks
  - Loads BL2 from FIP (Firmware Image Package) into SRAM
  - ERET → S-EL1 (BL2)
  │
  ▼ S-EL1, Secure state
BL2 (arm-trusted-firmware/bl2/)
  - Loads ALL remaining images from FIP:
      BL31 → SRAM (EL3 runtime)
      BL32 → 0x0e100000 (Green TEE, Secure DRAM)
      BL33 → DRAM (U-Boot, NS world)
  - Builds bl_params linked list with load addresses
  - SMC → BL1 → BL1 passes control to BL31
  │
  ▼ EL3, Secure state (PERMANENT RESIDENT)
BL31 (arm-trusted-firmware/bl31/)
  - Installs EL3 exception vectors permanently
  - Registers runtime services:
      PSCI (power management)
      green_teed SPD (our TEE dispatcher)
  - Calls green_teed_setup():
      reads BL32 entry point from ep_info
      cm_set_context() — registers cpu_context for CPU0
      bl31_register_bl32_init(&green_teed_init)
  - Calls green_teed_init():
      cm_el1_sysregs_context_restore(SECURE)
      cm_set_next_eret_context(SECURE)  → SCR_EL3.NS = 0
      green_teed_enter_sp() → saves EL3 callee regs, calls el3_exit
      ERET → S-EL1 (Green TEE main)
  │
  ▼ S-EL1, Secure state
Green TEE main() (tee/kernel/main.c)
  - boot.S _start runs first:
      reads MPIDR_EL1 → get CPU ID
      sets SP = __tee_stack_base - (cpu_id × 32KB)
      clears BSS section
      writes vector_table address to VBAR_EL1
      calls main()
  - main() initialises:
      mm_heap_init()        — heap at 0x0E600000
      pl011_init()          — Secure UART → QEMU TCP:12345
      mmu_init()            — builds 3-level page tables, enables S-EL1 MMU
      generic_timer_init()  — ARM generic timer
      otp_init()            — reads CNTPCT_EL0 twice → 64-bit OTP key
                               KEY NEVER LEAVES S-EL1 SRAM
  - Builds vector_table struct with function pointers
  - Calls green_tee_smc_entry_done(&vector_table):
      SMC(ENTRY_DONE, vector_table_ptr) → EL3
      EL3 green_teed stores the function pointers
      EL3 ERET → BL31 resumes
  - BL31 ERets → NS-EL1 (U-Boot)
  │
  ▼ NS-EL1, Non-Secure state
U-Boot (BL33)
  - Initialises DDR, virtio block device
  - Loads Linux kernel Image + DTB + initramfs
  - booti → jumps to Linux
  │
  ▼ NS-EL1, Non-Secure state
Linux Kernel
  - Boots normally
  - green_tee driver init: misc_register() → creates /dev/green_tee
  - green_tee_arch_init(): SMC(LINUX_INIT) → EL3 logs "NS world established"
  - Login prompt appears
```

---

### Phase 2: Runtime — encrypt.o /tmp/test.txt

```
NS-EL0: encrypt.o
  open("/dev/green_tee", O_RDWR)       → fd
  read(file_fd, buff, 4096)            → buff has plaintext
  ioctl(tee_fd, GREEN_TEE_ENCRYPT, buff)
    │
    │ SVC #0 instruction → trap to NS-EL1
    ▼
NS-EL1: Linux TEE Driver (green_tee_main.c)
  green_tee_ioctl():
    kzalloc(4096, GFP_KERNEL)          → kernel buffer
    strncpy_from_user(kbuf, buff, 4096)→ copy from userspace
    green_tee_arch_encrypt_data(kbuf)
      │
      ▼ (green_tee_arm.c)
    arm_smccc_1_1_smc(
      0x72000082,           ← Function ID:
                               bit31=0  (yield/std call)
                               bit30=1  (SMC64)
                               bits29:24=0x32 (OWNER_TRUSTED_OS)
                               bits15:0=0x82  (130=ENCRYPT)
      virt_to_phys(kbuf),   ← x1: physical address (NOT virtual!)
      0,0,0,0,0,0,          ← x2-x8 unused
      &res)
      │
      │ SMC #0 instruction → trap to EL3
      ▼
EL3: TF-A BL31 (green_teed_main.c)
  SMC handler fires via VBAR_EL3 vector
  Identifies owner: (smc_fid >> 24) & 0x3F = 0x32 = TRUSTED_OS
  Routes to green_teed_smc_handler()

  is_caller_non_secure(flags) == true:
    case GREEN_TEE_SMC_LINUX_ENCRYPT:
      cm_el1_sysregs_context_save(NON_SECURE)
        ← saves Linux's: SCTLR_EL1, TTBR0/1_EL1, TCR_EL1,
                         MAIR_EL1, VBAR_EL1, ELR_EL1, SPSR_EL1
      cm_set_elr_el3(SECURE, vector_table.yield_smc_entry)
        ← sets ELR_EL3 = green_tee_smc_handler address
      cm_el1_sysregs_context_restore(SECURE)
        ← restores Green TEE's saved system registers
      cm_set_next_eret_context(SECURE)
        ← SCR_EL3.NS = 0  (THE WORLD SWITCH HAPPENS HERE)
      SMC_RET4(&cpu_context, smc_fid, x1, x2, x3)
        ← ERET with args in registers → S-EL1
      │
      ▼ SCR_EL3.NS=0, ERET
S-EL1: Green TEE (green_tee_smc.c)
  green_tee_smc_handler(smc_fid=0x72000082, x1=phys_addr, ...)
    switch(smc_fid & 0xFFFF):  → case 130 (ENCRYPT):
      mmu_disable()
      mmu_map_range(x1, x1, x1+PAGE_SIZE,
                    PT_SECURE | PT_ATTR1_NORMAL | ...)
        ← maps the NS physical address into S-EL1 page tables
        ← marked PT_SECURE so Green TEE's MMU allows access
      mmu_invalidate_tlb()
      otp_enc_buffer((uint64_t*)x1, 4096)
        ← XORs every 8-byte word with otp_key
        ← otp_key is in .data section at 0x0e1xxxxx — SECURE DRAM
        ← Linux CANNOT read 0x0e000000-0x0f000000 (TZASC blocks it)
      mmu_enable()
      green_tee_smc_handled()
        SMC(HANDLED) → EL3
        │
        ▼
EL3: green_teed_smc_handler()
  is_caller_secure(flags) == true:
    case GREEN_TEE_SMC_HANDLED:
      cm_el1_sysregs_context_save(SECURE)
      cm_el1_sysregs_context_restore(NON_SECURE)
        ← restores Linux's system registers
      cm_set_next_eret_context(NON_SECURE)
        ← SCR_EL3.NS = 1  (WORLD SWITCH BACK)
      SMC_RET4(NS_context, 0, 0, 0, 0)
        ← ERET → NS-EL1, x0=0 (success)
      │
      ▼ SCR_EL3.NS=1, ERET
NS-EL1: Linux driver resumes
  arm_smccc_smc returns: res.a0 = 0
  copy_to_user(buff, kbuf, 4096)  ← encrypted data back to userspace
  kfree(kbuf)
  return 0
      │
      ▼ ERET → NS-EL0
NS-EL0: encrypt.o
  ioctl() returns 0
  lseek(fd, 0, SEEK_SET)
  write(fd, buff, 4096)    ← encrypted content written to file
```

---

## Issues We Hit and How We Fixed Them

### Issue 1: `LD_LIBRARY_PATH` blocking Buildroot

**Symptom:**
```
You seem to have the current working directory in your LD_LIBRARY_PATH. This doesn't work.
```

**Root cause:** Conda (miniconda3) adds `.` to `LD_LIBRARY_PATH`. The Qualcomm AI SDK adds a trailing `:` which creates an empty entry — equivalent to `.`. Buildroot explicitly refuses to run with either.

**Fix:**
```bash
unset LD_LIBRARY_PATH
```

**What to say in interview:** "Buildroot's dependency checker enforces a clean host environment. A trailing colon in `LD_LIBRARY_PATH` creates an implicit `.` entry, which can cause the build system to pick up host libraries instead of cross-compiled ones — leading to silent ABI mismatches. Buildroot catches this at startup."

---

### Issue 2: U-Boot failing with `bad value 'armv8-a+crc' for -march=`

**Symptom:**
```
cc1: error: bad value 'armv8-a+crc' for '-march=' switch
cc1: note: valid arguments: nocona core2 nehalem ...  (x86 list!)
```

**Root cause:** The top-level Makefile ran `make` inside `u-boot/` without setting `CROSS_COMPILE`. The host `gcc` (x86_64) was used to compile ARM64 code. `-march=armv8-a+crc` is valid for aarch64-gcc but completely unknown to x86 gcc.

**Fix:** Added `CROSS_COMPILE=aarch64-linux-gnu-` to the U-Boot make invocation.

**What to say in interview:** "This is a classic cross-compilation misconfiguration. U-Boot's build system uses `CROSS_COMPILE` as a prefix for all toolchain binaries. Without it, `$(CC)` resolves to the host compiler. The error was immediately obvious because the valid `-march` values listed were x86 architecture names, not ARM."

---

### Issue 3: U-Boot `gnutls/gnutls.h: No such file or directory`

**Symptom:**
```
tools/mkeficapsule.c:20:10: fatal error: gnutls/gnutls.h: No such file or directory
```

**Root cause:** U-Boot's `mkeficapsule` tool (for UEFI capsule update signing) requires libgnutls development headers. Ubuntu doesn't install this by default.

**Fix:**
```bash
sudo apt install -y libgnutls28-dev
```

---

### Issue 4: EL3 ASSERT `context_mgmt.c:1934` — Double SMC

**Symptom:** Every call to `print.o` caused:
```
ASSERT: lib/el3_runtime/aarch64/context_mgmt.c:1934
```
followed by Linux RCU stall warnings and a hung CPU.

**Root cause:** The `green_tee_smc_handler()` function in `tee/kernel/green_tee_smc.c` had a missing `return` statement:

```c
// BUGGY CODE
        green_tee_smc_handled();   // sends SMC_HANDLED → EL3 restores NS, returns to Linux

green_tee_smc_fail:
        green_tee_smc_failed();    // IMMEDIATELY sends SMC_FAILED → EL3 again!
```

Green TEE was sending TWO SMC calls to EL3 in sequence:
1. `SMC_HANDLED` — EL3 saves Secure context, restores NS context, ERET to Linux
2. `SMC_FAILED` — fires from S-EL1 immediately after, but EL3 is now in NS state with no valid Secure context registered → `cm_get_context(SECURE)` returns NULL → assert

**Fix:**
```c
        green_tee_smc_handled();
        return;                    // ← added this

green_tee_smc_fail:
        green_tee_smc_failed();
```

**What to say in interview:** "This is a real-world Secure Monitor protocol violation. The SMCCC spec defines a strict call/return contract — once a Secure Payload calls SMC with a completion code, EL3 tears down the Secure context and restores Non-Secure. Any subsequent SMC from S-EL1 arrives at EL3 with an inconsistent world state. I decoded the EL3 backtrace addresses using `aarch64-linux-gnu-addr2line` against `bl31.elf` to pinpoint the exact assert, then traced it to the missing return."

---

### Issue 5: EL3 ASSERT on SMP — `cm_get_context(SECURE)` returns NULL

**Symptom:** Same assert even after fixing Issue 4. The backtrace was identical: `context_mgmt.c:1934`.

**Root cause:** QEMU was launched with `-smp 6` (6 virtual CPUs). TF-A's `green_teed_setup()` registered the Secure context (`cm_set_context()`) only for the boot CPU (CPU0). When Linux issued the SMC from `print.o`, the Linux scheduler happened to run the ioctl on CPU4. CPU4 had no Secure context registered in TF-A's per-CPU data structure. `cm_get_context(SECURE)` returned NULL → assert at line 1934.

**Decoded with:**
```bash
aarch64-linux-gnu-addr2line -e bl31.elf 0xe0a21d4 0xe0a0234 0xe0a2c08 0xe0a4494 0xe0a2038
# Output: context_mgmt.c:1934 ← cm_set_elr_el3() called from green_teed_main.c:144
```

**Fix:** Changed `-smp 6` to `-smp 1` in QEMU args.

**Proper production fix** (not done here): In `green_teed_smc_handler()`, for each CPU that issues its first SMC, call `cm_set_context()` and `cm_init_my_context()` to register the Secure context for that CPU dynamically. OP-TEE handles this properly in its SPD.

**What to say in interview:** "TF-A maintains per-CPU context structures. The `cm_set_context()` call during SPD setup only initialises the boot CPU. On an SMP system, secondary CPUs need their own Secure context registered before they can execute SMC calls that switch to Secure world. This is why OP-TEE's `opteed` SPD handles CPU hotplug events — it registers Secure context for each CPU as it comes online via the `cpu_on_entry` power management callback."

---

## Interview Questions and Answers

### TrustZone Fundamentals

**Q: What is the NS bit and where is it physically?**

It is bit 0 of `SCR_EL3` (Secure Configuration Register), only writable from EL3. When NS=1, the CPU is in Non-Secure state. This bit propagates to the AXI bus as the `HNONSEC` signal, which the TZASC uses to allow or deny memory transactions in hardware.

**Q: Can Linux kernel code read Secure DRAM directly?**

No. Even if Linux issues a load instruction targeting a Secure physical address, the TZASC (TrustZone Address Space Controller) raises a DECERR bus error before the transaction reaches the memory controller. This is enforced in hardware — no software workaround exists from the NS world.

**Q: What is the only way to switch from Normal World to Secure World?**

The `SMC` (Secure Monitor Call) instruction. It triggers a synchronous exception that traps to EL3. EL3 code (TF-A BL31) decides whether to switch worlds, saves the current world's context, restores the other world's context, and does `ERET` with `SCR_EL3.NS` set appropriately.

**Q: What registers must be saved during a world switch?**

General purpose registers x0–x30, SP_EL0, SP_EL1. EL1 system registers: SCTLR_EL1, TTBR0_EL1, TTBR1_EL1, TCR_EL1, MAIR_EL1, AMAIR_EL1, VBAR_EL1, ELR_EL1, SPSR_EL1, CONTEXTIDR_EL1. EL3 return state: ELR_EL3, SPSR_EL3. SIMD/FP registers if the TEE uses them.

---

### TF-A Architecture

**Q: What is the difference between BL31 and BL32?**

BL31 is the EL3 Secure Monitor — permanent resident, highest privilege, manages context switching. BL32 is the Secure OS payload (Green TEE, OP-TEE) running at S-EL1. BL31 loads and initialises BL32 via the SPD plugin, then manages all cross-world transitions on BL32's behalf.

**Q: What is a Secure Payload Dispatcher (SPD)?**

A TF-A plugin that manages a specific Secure OS. It registers with BL31's runtime service framework using `DECLARE_RT_SVC`, receives SMC calls for its SMCCC owner ID range, and is responsible for: initialising the Trusted OS at boot, saving/restoring NS and Secure EL1 contexts on every SMC, and dispatching requests to the Trusted OS.

**Q: What is a FIP?**

Firmware Image Package — a single binary container (like a tar archive) holding all firmware images: BL1, BL2, BL31, BL32 (TEE), BL33 (U-Boot), certificates. BL2 parses the FIP to extract and load each image. Simplifies flash layout — one file instead of images at fixed offsets.

**Q: What does `cm_set_next_eret_context(SECURE)` do?**

It configures ELR_EL3 and SPSR_EL3 for the upcoming `ERET` instruction. Specifically it sets `SCR_EL3.NS = 0` (Secure state), loads the Secure world's saved ELR into ELR_EL3 (the address to return to in S-EL1), and loads the Secure SPSR. When `ERET` executes, the CPU atomically changes EL, security state, and PC.

---

### Green TEE Specific

**Q: How does Green TEE's OTP key stay secret from Linux?**

`otp_key` is a `static uint64_t` in `tee/crypto/otp.c`, compiled into the `.data` section of `tee.elf`. The linker places it at a virtual address inside `0x0e100000–0x0f000000`. TF-A's QEMU platform code configures the TZASC to mark this entire physical range as Secure-only. Any NS bus transaction to this range is rejected in hardware — Linux cannot read it even with `/dev/mem` or a custom kernel driver.

**Q: Why does Green TEE call `mmu_disable()` before `mmu_map_range()` during encrypt?**

Green TEE's `mmu_map_range()` modifies the live page table that the S-EL1 MMU is currently using. If the MMU is active while page table entries are being written, a speculative TLB walk could read a partially-written entry, causing a translation fault or incorrect mapping. Disabling the MMU makes the modification safe, after which `mmu_invalidate_tlb()` and `mmu_enable()` restore the correct state.

**Q: Why does the Linux driver pass `virt_to_phys(buff)` instead of the virtual address?**

Green TEE has a completely independent MMU and page tables from Linux. Linux virtual addresses are meaningless in S-EL1's address space. `virt_to_phys()` converts the kernel virtual address to the physical address, which is a hardware reality both worlds share. Green TEE then uses `mmu_map_range()` to create a temporary S-EL1 mapping for that physical address.

**Q: What is the SMCCC function ID for GREEN_TEE_ENCRYPT?**

`0x72000082` — constructed as:
- bit 31 = 0: Yield/Standard call (not fast)
- bit 30 = 1: SMC64
- bits 29:24 = 0x32 (50): OWNER_TRUSTED_OS
- bits 15:0 = 0x82 (130): GREEN_TEE_SMC_LINUX_ENCRYPT

**Q: What caused the SMP assert and how would you fix it properly in production?**

TF-A's `green_teed_setup()` calls `cm_set_context()` only for the boot CPU. With `-smp 6`, Linux can schedule the ioctl on any CPU. If CPU4 issues the SMC, `cm_get_context(SECURE)` returns NULL for that CPU → assert. The production fix is to handle `cpu_on_entry` in the SPD: when a secondary CPU boots, call `cm_init_my_context()` with the Secure EP info to register its context. OP-TEE's `opteed` does exactly this.

---

### What You Actually Did (Interview Story)

> "I built Green TEE — a minimal ARM TrustZone TEE — completely from source on QEMU. This involved building five components: ARM Trusted Firmware (BL1/BL2/BL31 + a custom EL3 Secure Payload Dispatcher), a bare-metal Trusted OS running at S-EL1, a Linux kernel character device driver, and a userspace client.
>
> The interesting part was the debugging. After getting the system to boot, every SMC call caused an EL3 assertion in TF-A's context management code. I used `aarch64-linux-gnu-addr2line` against `bl31.elf` to decode the backtrace — it pointed to `cm_set_elr_el3()` asserting that `cm_get_context(SECURE)` returned NULL.
>
> I found two bugs. First, the S-EL1 Trusted OS was issuing two consecutive SMC calls to EL3 — a HANDLED followed immediately by a FAILED due to a missing `return` statement. After EL3 processes HANDLED it restores NS context and the Secure context is torn down, so the second SMC arrived with no valid Secure context.
>
> The second bug was an SMP issue. TF-A's SPD registered the Secure context only for CPU0 during boot. With 6 vCPUs, Linux scheduled the ioctl on CPU4, which had no Secure context — same assert. Fixed by reducing to single CPU for this educational project; the production fix is registering per-CPU Secure context in the cpu_on_entry power management callback, which is what OP-TEE's opteed SPD does.
>
> The working system successfully crosses the TrustZone boundary: a userspace app issues an ioctl, the Linux kernel driver calls arm_smccc_smc with the physical buffer address, TF-A saves NS-EL1 context and switches SCR_EL3.NS=0, the Trusted OS maps the NS buffer into its Secure page tables, XOR-encrypts it with a key that never leaves Secure DRAM, SMCs back to EL3, which restores NS context and returns the encrypted data to Linux."

---

## Memory Map Reference

```
Physical Address    Region                  Access
─────────────────────────────────────────────────────
0x0e000000          __secure_mem_begin      Secure only (TZASC)
0x0e100000          Green TEE _start        Secure only
  .text.asm         boot.S, exceptions.S
  .text             C code
  .data             otp_key lives here ← Linux cannot reach this
  .rodata
  .bss
  .stack (↓)
0x0f000000          __tee_limit / stack top Secure only
─────────────────────────────────────────────────────
0x40000000+         Linux DRAM (NS world)   NS + Secure read
0x44231000          Shared buffer example   NS (mapped by TEE temporarily)
─────────────────────────────────────────────────────
0x09000000          PL011 UART0             NS-EL1 (Linux console)
0x09040000          PL011 UART1             Secure (Green TEE → TCP:12345)
0x08000000          GICv3 Distributor       EL3 + NS-EL1
```

---

## Key Commands Reference

```bash
# Build
unset LD_LIBRARY_PATH
make tee && make tfa          # rebuild after TEE changes
make linux                    # rebuild after driver changes
make                          # full build

# Run
make nc                       # Terminal 2: listen for Secure UART
make run                      # Terminal 1: launch QEMU

# Debug
make debug                    # QEMU with -s -S (waits for GDB)
make gdb                      # loads all ELF symbols, connects to :1234

# Decode EL3 backtraces
aarch64-linux-gnu-addr2line -e arm-trusted-firmware/build/qemu/debug/bl31/bl31.elf <addr>

# Disassemble Green TEE
aarch64-linux-gnu-objdump -D tee/build/tee.elf | less

# Inspect SMC function IDs
aarch64-linux-gnu-objdump -D tee/build/tee.elf | grep -B2 -A10 "smc"

# Stop QEMU
Ctrl+A then X
```

---

*Built on Ubuntu 24.04 | QEMU 8.2.2 | aarch64-linux-gnu-gcc 13.3.0 | TF-A v2.9 | Linux 6.12.0-rc7*
