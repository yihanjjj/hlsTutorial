# How to build a vector addition using Vitis and Vivado?

We are using Vitis HLS 2021.1 and Vivado 2021.1 for this tutorial. The FPGA is PYNQ-Z2.

### How to access Vitis and Vivado?

The recommended choice is using Vitis and Vivado that are preinstalled in Georgia Tech ECE server. Please find detailed information about setting up vpn and server [here](https://help.ece.gatech.edu/labs/names). Use the following cmd to locate the Vitis HLS 2021.1:

```sh
cd tools/software/xilinx/Vitis_HLS/2021.1/bin
```

Next, run the following command to start the Vitis.

```sh
vitis_hls
```

Vivado can be found by a similiar command:

```sh
cd tools/software/xilinx/Vivado/2021.1/bin
```

It can be used by entering the following command.

```sh
vivado
```

The second choice is to install the your own [Vitis and Vivado](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vitis.html). It is not suggested because of its large size, but your own software sometimes might be faster than using the one on the server.

### Design flow

In this section, we will introduce you the flow of building a vector addition function on Vitis and Vivado. We will first implement the c++ code on Vitis and export RTL design for later use in Vivado.

#### Vitis

Firstly, we need to create a new project. Let's use `vadd` as the project name.
It is not necessay to specify the top function nor the testbench now.
The Part Selection in the last page for initializing a new project, however, is very essential for this program to run.
Because we use PYNQ-Z2, please be sure to select the board with `xc7z020clg400-1` Part and `220` DSP.

Now we can write our acclerator in c++ and simulated with Vitis.
Create a new source called `top.cc`.
Copy paste the following code to `top.cc`.

```cpp
void add(int a[100], int b[100], int sum[100]) {
	#pragma HLS interface m_axi port=a depth=100 offset=slave bundle = INPUT
	#pragma HLS interface m_axi port=b depth=100 offset=slave bundle = INPUT
	#pragma HLS interface m_axi port=sum depth=100 offset=slave bundle = OUTPUT
	#pragma HLS interface s_axilite register port=return
	for (int i = 0; i < 100; i++) {
		sum[i] = a[i] + b[i];
	}

}
```

Click `Project > Project Synthesis`. Choose the `Synthesis` settings and
specify the `top.cc` as the top function. Next click the synthesis icon
(green triangle). If no error occurs after synthesizing,
click `Solution > Export RTL`.

Now we have done with Vitis and will switch to Vivado.

#### Vivado

Similar to creating a new Vitis project, the initial set up of a new vivado
project only requires the correct board selection. In our example,
we name our project as `adderProject`. Make sure you select the
same board as the one in Vitis (`xc7z020clg400-1` Part and `220` DSP).

For the next step, we will need to add our vector addition ip core to the Vivado
for building the block design. Click `IP Catalog` at the left column.
Right click the `Vivado Repository` and select `Add Repository`.
Select the folder that includes `Solution1`. After click the `select` button,
you will see a small page telling you that "1 repository was added to the project".
Expand the IPs tab. If you see the top ip with an orange icon, it means no issue
for now. If the icon is grey, you might want to check whether you choose the
same board for Vitis and Vivado.

Now we will build the block diagram. Click the `Create Block Design` at the
left column. Click the `+` icon at the upper side of the diagram.
Type `hls` for finding the add function ip.
Type `zynq 7 series` for finding the ip of PYNQ-Z2 family.

Becuase we specify two inputs and an output in our c++ code, we need to
initialize two buses on the fpga. Double click the `zynq 7 series` icon
on the block diagram. Select `PS-PL Configuration`. Then, select the
`AXI HP0 FPD` and `AXI HP1 FPD` under `Slave Interface > AXI HP` by
checking their boxes.

After this step, go back to the block diagram and click the `Run Connection Automation`.
Select the `All Automation` at the left column.
Click on the `S_AXI_HP0_FPD` and change its `Master Interface` to be
`/add_0/m_axi_OUTPUT_r`. Click `OK` to start connection automation.
When it finished, you will see a completed block diagram. To check its correctness,
click the `validation` (check icon) on the upper page.

We finished building the block diagram, and now we will create a wrapper for it.
Find the block diagram file under the design sources. Right click the
design file (whatever you name it) and select `Create HDL Wrapper`.
Choose `Let Vivado manage wrapper and auto-update` as option.
click `OK` to start.

Finally, it comes to our last step. Click the `Generate Bitstream` under
`PROGRAM AND DEBUG` division (at the lower left of the entire page).
Use the default settings (for our simple example) and start to run.

After generating the bitstream file, we need two files for running the vector addition on FPGA: one with
`.bit` as the extension and the other with `.hwh`. Bitstream file is easy to
locate. The `.hwh` file is under directory
`adderProject/adderProject.gen/sources_1/bd/design_1/hw_handoff`.
You can copy/paste these two files to a flash drive and put them on your own laptop for the next step.

### Testing on PYNQ-Z2

If you have not used the PYNQ-Z2 before, check this page for [setup](https://pynq.readthedocs.io/en/v2.6.1/getting_started/pynq_z2_setup.html#).

If you already have a basic idea of Jupyter on PYNQ-Z2, upload the `.bit` file
and the `.hwh` file to Jupyter. In the same folder, create a new `.ipynb` file
for writing the script.

Click [here](https://pynq.readthedocs.io/en/v2.0/overlay_design_methodology/overlay_tutorial.html) to access the overlay tutorial.

Note: If you need to find the address offset of your own registers, check the xadd_hw.h file under `solution1/impl/misc/drivers/add_v1_0/src` direcoty.

```python
import numpy as np
import pynq
from pynq import MMIO

overlay = pynq.Overlay('top.bit')

top_ip = overlay.top_0
top_ip.signature

size = 100*100
a_buffer = pynq.allocate((100, 100), np.uint32)
b_buffer = pynq.allocate((100, 100), np.uint32)
sum_buffer = pynq.allocate((100, 100), np.uint32)

# initialize input
a_buffer[:] = np.random.randint(low=0, high=100, size=(100, 100), dtype=np.uint32)
b_buffer[:] = 200

top_ip.write(0x00, 4)
fpga_state = top_ip.read(0x00)

aptr = a_buffer.physical_address
bptr = b_buffer.physical_address
sumptr = sum_buffer.physical_address
top_ip.write(0x00, 4)
if fpga_state == 4:
    top_ip.write(0x10, aptr)
    top_ip.write(0x1c, bptr)
    top_ip.write(0x28, sumptr)

%%timeit

add_ip.write(0x00, 1)
fpga_state = top_ip.read(0x00)

top_ip.write(0x00, 1)
isready = top_ip.read(0x00)
while( isready == 1 ):
    isready = add_ip.read(0x00)

print(a_buffer[0][39])
print(b_buffer[0][39])
print(sum_buffer[0][39])

addr_ap_ctrl = 0x0
mmio = MMIO(0x0, 0x1000)
bits = mmio.read(addr_ap_ctrl)
print(bits)
ap_idle = bits>>2 & 0x1
print(ap_idle)

```
