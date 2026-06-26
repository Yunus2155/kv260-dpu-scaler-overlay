# kv260-dpu-scaler-overlay
Vitis Accelerator Flow build for Kria SOM: DPU B4096 + VVAS multiscaler  overlay for custom defect detection inference.

## Build Method

This design was built using the Vitis Accelerator Flow (`v++ --link`).
The DPU and Multiscaler kernels (.xo files) were generated from the 
reference repositories below, then linked together using the configuration 
files in this repo (`system.cfg`, `system_link.tcl`).

### Vivado Platform Configuration

A custom Vitis platform was created in Vivado with the following 
configuration:
- Zynq UltraScale+ MPSoC: AXI HPC0, HP0, HP1, HP2, HP3 (slave interfaces) enabled
- Clocking Wizard: 300MHz, 600MHz, 100MHz outputs
- AXI Interrupt Controller (axi_intc_0) for DPU/scaler interrupts
- 3x Processor System Reset blocks (one per clock domain)

The platform was exported as an extensible XSA, converted to .xpfm using 
XSCT, and used as the base for kernel linking via v++.

## Reference Repositories Used

- **DPU TRD (DPUCZDX8G):** https://github.com/Xilinx/Vitis-AI/tree/2.5/reference_design
- **VVAS (Multiscaler IP):** https://github.com/Xilinx/VVAS/tree/vvas-rel-v2.0
