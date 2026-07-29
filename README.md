# kv260-dpu-scaler-overlay

Vitis Accelerator Flow build for Kria SOM: DPU B4096 + VVAS multiscaler overlay for custom defect detection inference.

## Build Method

This design was built using the Vitis Accelerator Flow (`v++ --link`).
The DPU and Multiscaler kernels (.xo files) were generated from the
reference repositories below, then linked together using the configuration
files in this repo (`system.cfg`, `system_link.tcl`).

### Vivado Platform Configuration

A custom Vitis platform was created in Vivado with the following configuration:
- Zynq UltraScale+ MPSoC: AXI HPC0, HP0, HP1, HP2, HP3 (slave interfaces) enabled
- Clocking Wizard: 300MHz, 600MHz, 100MHz outputs
- AXI Interrupt Controller (axi_intc_0) for DPU/scaler interrupts
- 3x Processor System Reset blocks (one per clock domain)

The platform was exported as an extensible XSA, converted to .xpfm using
XSCT, and used as the base for kernel linking via v++.

## Reference Repositories Used

- **DPU TRD (DPUCZDX8G):** https://github.com/Xilinx/Vitis-AI/tree/2.5/reference_design
- **VVAS (Multiscaler IP):** https://github.com/Xilinx/VVAS/tree/vvas-rel-v2.0

---

# How to Build the Overlay

## Prerequisites

- Ubuntu 20.04 or 22.04 (native or WSL)
- Vivado 2022.1
- Vitis 2022.1
- Git

> **Note:** Pre-built deployment artifacts are available in the `deploy/`
> folder if you want to skip the build process and deploy directly.

---

## Directory Structure

After following this tutorial, your directory structure will look like this:

```
kv260_build/
├── dpu/                          # DPU TRD (from Vitis-AI repo)
├── scaler/                       # VVAS Multiscaler (from VVAS repo)
├── platform/                     # Vitis platform files
├── link/                         # v++ link outputs (xclbin, xsa, logs)
├── deploy/                       # Final deployment files
│   ├── kv260_dpu_scaler.xclbin
│   ├── kv260_dpu_scaler.bit.bin
│   ├── kv260_dpu_scaler.dtbo
│   └── shell.json
└── kv260-dpu-scaler-overlay/     # This repository
    ├── config/                   # DPU and scaler config files
    ├── deploy/                   # Pre-built deployment artifacts
    ├── hw/                       # Platform hardware files
    ├── sw/                       # Platform software files
    └── kv260_base_platform.xpfm  # Custom Vitis platform
```

---

## Platform Options

Choose one of the following options for the Vitis platform:

**Option 1 — Use AMD's pre-built platform (fastest)**
Download from: https://github.com/Xilinx/kria-vitis-platforms

**Option 2 — Use our custom platform (recommended)**
Already included in this repository (`kv260_base_platform.xpfm`, `hw/`, `sw/`).
Use this path in the build steps below:
```
~/kv260_build/kv260-dpu-scaler-overlay/kv260_base_platform.xpfm
```

**Option 3 — Build your own platform from scratch**
Follow AMD's official Vitis Platform Flow tutorial:
https://xilinx.github.io/Vitis-Tutorials/2022-1/build/html/docs/Vitis_Platform_Creation/Design_Tutorials/01-Edge-KV260/step1.html

---

## Step 0: Clone Repositories

```bash
mkdir kv260_build && cd kv260_build

# DPU TRD
git clone --branch v2.5 --single-branch --depth 1 \
  https://github.com/Xilinx/Vitis-AI.git dpu

# VVAS Multiscaler
git clone --branch vvas-rel-v2.0 --single-branch --depth 1 \
  https://github.com/Xilinx/VVAS.git scaler/VVAS

# This repository
git clone https://github.com/Yunus2155/kv260-dpu-scaler-overlay.git
```

---

## Step 1: Configure and Build DPU

```bash
cp kv260-dpu-scaler-overlay/config/dpu_conf.vh \
   dpu/DPUCZDX8G/prj/Vitis/dpu_conf.vh

cd dpu/DPUCZDX8G/prj/Vitis
source /tools/Xilinx/Vitis/2022.1/settings64.sh
make binary_container_1/dpu.xo DEVICE=SOM
cd ~/kv260_build
```

---

## Step 2: Configure and Build Multiscaler

```bash
cp kv260-dpu-scaler-overlay/config/v_multi_scaler_config.h \
   scaler/VVAS/vvas-accel-hw/multiscaler/v_multi_scaler_config.h

cd scaler/VVAS/vvas-accel-hw/multiscaler
source /tools/Xilinx/Vitis/2022.1/settings64.sh
make PLATFORM_FILE=~/kv260_build/kv260-dpu-scaler-overlay/kv260_base_platform.xpfm
cd ~/kv260_build
```

---

## Step 3: Link Kernels (v++ --link)

```bash
mkdir -p ~/kv260_build/link
cp kv260-dpu-scaler-overlay/config/system.cfg ~/kv260_build/link/
cp kv260-dpu-scaler-overlay/config/system_link.tcl ~/kv260_build/link/

cd ~/kv260_build/link
source /tools/Xilinx/Vitis/2022.1/settings64.sh
v++ --link \
    --platform ~/kv260_build/kv260-dpu-scaler-overlay/kv260_base_platform.xpfm \
    --target hw \
    --config system.cfg \
    -o kv260_dpu_scaler.xclbin \
    ~/kv260_build/dpu/DPUCZDX8G/prj/Vitis/binary_container_1/dpu.xo \
    ~/kv260_build/scaler/VVAS/vvas-accel-hw/multiscaler/xo/v_multi_scaler.xo
```

> ⚠️ This step takes approximately 1 hour.

---

## Step 4: Generate .bit.bin

```bash
cd ~/kv260_build/link
mkdir -p bit_extract
unzip kv260_dpu_scaler.xsa -d bit_extract
cd bit_extract
echo 'all: { [destination_device = pl] vpl_gen_fixed.bit }' > boot.bif
bootgen -w -arch zynqmp -process_bitstream bin -image boot.bif
cd ~/kv260_build
```

---

## Step 5: Generate .dtbo

```bash
cd ~/kv260_build/platform/kv_260_base_platform
source /tools/Xilinx/Vitis/2022.1/settings64.sh
xsct -eval "createdts -hw kv260_base_platform.xsa -zocl \
  -platform-name kv260_base_platform -git-branch xlnx_rel_v2022.1 \
  -overlay -compile -out ./dt_output"

cd dt_output/dt_output/kv260_base_platform/psu_cortexa53_0/device_tree_domain/bsp
cpp -nostdinc -undef -x assembler-with-cpp -I . pl-final.dts pl-final.pp.dts
dtc -@ -O dtb -o pl.dtbo -I dts pl-final.pp.dts
cd ~/kv260_build
```

---

## Step 6: Collect Deployment Files

```bash
mkdir -p ~/kv260_build/deploy

cp ~/kv260_build/link/kv260_dpu_scaler.xclbin ~/kv260_build/deploy/
cp ~/kv260_build/link/bit_extract/vpl_gen_fixed.bit.bin \
   ~/kv260_build/deploy/kv260_dpu_scaler.bit.bin
cp ~/kv260_build/platform/kv_260_base_platform/dt_output/dt_output/kv260_base_platform/psu_cortexa53_0/device_tree_domain/bsp/pl.dtbo \
   ~/kv260_build/deploy/kv260_dpu_scaler.dtbo

cat > ~/kv260_build/deploy/shell.json << 'EOF'
{
  "shell_type": "XRT_FLAT",
  "num_slots": "1"
}
EOF
```

Your `deploy/` folder now contains the 4 files needed for deployment:
- `kv260_dpu_scaler.xclbin`
- `kv260_dpu_scaler.bit.bin`
- `kv260_dpu_scaler.dtbo`
- `shell.json`
