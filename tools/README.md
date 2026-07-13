<!--
Copyright (C) 2026 SharpEmu Emulator Project
SPDX-License-Identifier: GPL-2.0-or-later
-->

# SharpEmu Tools

Developer tools that verify the Gen5 shader translation pipeline with
synthetic instruction programs — no game dumps involved anywhere.

## SharpEmu.Tools.ShaderDump

Feeds hand-assembled Gen5 (gfx10) instruction words — cross-checked against
LLVM's AMDGPU target definitions — through the real
`Gen5ShaderTranslator` → `Gen5SpirvTranslator` pipeline and writes the
resulting vertex and compute SPIR-V blobs to disk. Uses reflection, so it
needs no changes to emulator source.

```
dotnet run --project tools/SharpEmu.Tools.ShaderDump -- [output-directory]
```

The dumped blobs can be inspected with the standard Khronos tooling:

```
spirv-val --target-env vulkan1.3 spv/exec-cs.spv
spirv-dis spv/muls-cs.spv
```

## SharpEmu.Tools.GpuConformance

Executes the `exec-cs.spv` blob on a real Vulkan device (preferring a
discrete GPU) and compares the values it writes into a storage buffer
against CPU-computed expectations. Creating the compute pipeline doubles as
a driver-acceptance check for SharpEmu's emitted SPIR-V.

```
dotnet run --project tools/SharpEmu.Tools.GpuConformance -- <path>/exec-cs.spv
```

Expected output on a conforming translation:

```
executing on: <your GPU>
driver accepted the SPIR-V (pipeline created)
PASS  v_fmac_f32  fma(1.5, 2.25, 10.0): gpu=0x41560000 expected=0x41560000
PASS  v_mul_hi_i32 hi(0x7FFFFFFF*0x10003): gpu=0x00008001 expected=0x00008001
PASS  v_mul_lo_i32 lo(0x7FFFFFFF*0x10003): gpu=0x7FFEFFFD expected=0x7FFEFFFD
PASS  no stray writes past offset 8 (sentinel intact)
RESULT: all values match
```

The `exec` program exercises `v_fmac_f32`, `v_mul_hi_i32`, `v_mul_lo_i32`
and `buffer_store_dword`, so it covers the decode table, the ALU emitter,
the exec-mask store path, and the guest-buffer descriptor plumbing in one
dispatch.

Note: the `sopp` test program currently reports `unknown-sopp` decode
failures — that is the tool working as intended, detecting that the gfx10
SOPP hint instructions are not yet in the decode table (see #108). It
starts passing when that lands.

Neither project is part of the main solution; they build and run
independently with the repository's pinned .NET SDK.
