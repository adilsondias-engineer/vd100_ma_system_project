# vd100_ma_system_project

Vitis system project that integrates the AIE-ML graph, HLS kernels, and platform
into the final `aie.xclbin` and BOOT.BIN component artifacts.

---

## Overview

This is the integration hub of the pipeline. It runs the v++ linker to connect:

- `vd100-aie-ma-crossover` (AIE graph — `libadf.a`)
- `mm2s` HLS kernel
- `s2mm` HLS kernel
- `vd100_platform` (hardware platform — `.xpfm`)

And produces:
- `aie.xclbin` — deployed to VD100, loaded at runtime by XRT
- BOOT.BIN component CDOs — packaged into BOOT.BIN by Yocto

---

## Build Outputs

| Output | Location | Purpose |
|--------|----------|---------|
| `aie.xclbin` | `build/hw/binary_container_1.xclbin` | XRT runtime image |
| `aie.merged.cdo.bin` | `build/hw/package/aie.merged.cdo.bin` | BOOT.BIN `aie_image` partition |
| `aie.cdo.device.partition.reset.bin` | `build/hw/package/libadf/sw/` | BOOT.BIN `aie_dev_part` partition |
| `aie.8_0.elf` | (inside `aie.merged.cdo.bin`) | AIE core ELF — PLM loads at boot |

---

## v++ Link Configuration

The v++ linker resolves PLIO stream connections between PL kernels and the AIE array.
Key connectivity:

```
mm2s.s → ai_engine_0.mygraph.in      (PS DDR → AIE)
ai_engine_0.mygraph.out → s2mm.s     (AIE → PS DDR)
```

---

## Package Step

The v++ package step is critical. It produces `aie.merged.cdo.bin` which contains:

| CDO component | Purpose |
|---------------|---------|
| `aie.cdo.reset.bin` | AIE array reset sequence |
| `aie.cdo.clock.gating.bin` | Clock gate configuration |
| `aie.cdo.error.handling.bin` | BLOCK_NOCAXIMMERR setup |
| `aie.cdo.elfs.bin` | AIE core ELF loader |
| `aie.cdo.init.bin` | AIE tile initialisation |
| `aie.cdo.enable.bin` | Column clock enable |

**All of these must be present in BOOT.BIN** for AIE tiles to be ungated at boot.
The merged CDO is added to BOOT.BIN via the Yocto `xilinx-bootbin` bbappend in
`meta-vd100_v3`. See `vd100-aie-ma-crossover/README.md` for details.

---

## xclbin

```
UUID:     0f5096a5-b416-a54c-8035-9efc0e394fdc
Kernels:  mm2s (instance: mm2s_1), s2mm (instance: s2mm_1)
AIE graph: mygraph
```

Verify kernel names after any rebuild:

```bash
xclbinutil --info --input aie.xclbin | grep -E "Kernel|Instance"
```

---

## Build Steps

In Vitis IDE:

1. Open system project `vd100_ma_system_project`
2. Set active target: `Hardware`
3. **Build All** — runs v++ compile → link → package in sequence
4. Copy outputs to deployment locations:

```bash
# xclbin to VD100
scp build/hw/binary_container_1.xclbin root@vd100:~/files/build/aie.xclbin

# CDO files to Yocto meta layer
cp build/hw/package/aie.merged.cdo.bin \
   /work/yocto/sources/meta-vd100_v3/recipes-bsp/bootbin/files/

cp build/hw/package/libadf/sw/aie.cdo.device.partition.reset.bin \
   /work/yocto/sources/meta-vd100_v3/recipes-bsp/bootbin/files/
```

5. Rebuild Yocto boot image:

```bash
# On yoctoBuilder
bitbake xilinx-bootbin -c cleansstate && bitbake xilinx-bootbin
bitbake edf-linux-disk-image
```

---

## Timing Closure

| Metric | Value |
|--------|-------|
| WNS | 4.217 ns |
| WHS | 0.018 ns |
| PL clock | 100 MHz |

---

## SDT / DTS Generation

After the system project build, generate the device tree with zocl support:

```bash
sdtgen set_dt_param -dir sdt_out -zocl enable
```

This produces the correct 32-IRQ zocl DT node. The output feeds into the
`meta-vd100_v3/recipes-bsp/device-tree/` Yocto recipe.

> The `-zocl enable` flag is **mandatory**. Without it, the zocl driver probes
> with wrong IRQ configuration, hardware state is misreported, and AXI errors
> cause kernel panics instead of returning EINVAL.

---

## Notes

- **xclbin and BOOT.BIN must match** — if you rebuild the system project, both
  `aie.xclbin` and the BOOT.BIN CDO files must be updated together. Running a
  new xclbin against a BOOT.BIN built from old CDOs will fail.
- **`defer_aie_run` is NOT set** — PLM initialises the AIE array fully at boot.
  XRT does not need to enable clocks; they are enabled by the CDOs in BOOT.BIN.
- **No `xrtResetAIEArray`** — returns -1 on XCVE2302 with this platform, causes
  early exit. Do not call it.
