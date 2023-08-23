# IMCsim Programming Guide
This is a guide for programming the IMC devices to be simulated by IMCsim, using the libraries provided in this repository.

## Overview
Users/developers are supposed to program the **default parameterized IMC architecture**, using the **APIs provided**. The IMCsim simulator will then simulate the IMC device, and generate the corresponding output files.

If users want to customize a new IMC architecture, they can modify the **IMCsim core source code** to implement their desired architecture, and then develop new supporting APIs to program their device.

## Usage
1. Edit the `imc.c` file.
2. Edit the `main` function using the APIs provided below.
3. `make` to compile the program.

## APIs
APIs control how to write certain registers, whose values will determine the behavior of the IMC device.

### Device structure
TODO: a figure

### APIs
To drive components of the IMC architecture, one can use the following APIs. Basically, users' task is to find **respective addresses and vector lengths** for different types (inputs, outputs) of data, and start the computation in a right way to obtain the desired results.

Programming APIs:
| API |
| --- |
|    compute_add()|
|    compute_max()|
|    compute_mul()|
|    compute_mvm()|
|    compute_relu()|
|    fence()|
|    load_array()|
|    load_vector()|
|    load_vector_imm()|
|    load_input()|
|    load_rf()|
|    load_rf_imm()|
|    reg_read32()|
|    reg_write32()|
|    rx()|
|    store_buffer()|
|    store_output()|
|    tx()|

Debug APIs:
| API |
| --- |
|    reg_read32()|
|    reg_write32()|

One can implement most convolutional neural networks (CNNs, e.g., VGG, ResNet) using these APIs.

## Examples:
Note: all addresses used in parameters in these APIs are in **bytes**.
1. Single-bank MVM computation:
```c
	// manual mapping
	// compute MVM a (1,33) x b (33,61) = c (1,61)
	// single bank
	uint32_t id;
	id = 0;
	uint32_t bw_wei = 8;
	uint32_t num_row = 33;
	uint32_t num_col = 61;
	load_input(id, 0, 0, num_row);
	load_vector(id, 0, 0, num_row);
	for (int i=0;i<num_row;i++){
		load_array(id, 0+i*512/8, 0x400+i*num_col*8/bw_wei, num_col);
	}
	compute_mvm(id,0,0,num_row,0,num_col);
	store_buffer(id,0,0,num_col);
	// till now finish one MVM
	store_output(id,0,0x200,num_col);

```

2. Multibank MVM computation:
```c
int main(){
	// manual mapping
	// compute MVM a (1,1024) x b (1024,128) = c (1,128)
	// split b into 2 banks
	uint32_t id;
	id = 0;
	int bw_wei = 4;
	load_input(id, 0, 0, 512);
	load_vector(id, 0, 0, 512);
	load_array(id, 0, 0x400, 512*512/bw_wei);
	compute_mvm(id,0,0,512,0,128);
	store_buffer(id,0,0,128);
	// till now finish one MVM
	id = 1;
	load_input(id, 0, 512, 512);
	load_vector(id, 0, 0, 512);
	load_array(id, 0, 0x400+512*512/8, 512*512/bw_wei);
	compute_mvm(id, 0,0,512,0,128);
	store_buffer(id, 0,0, 128);
	// till now finish another MVM
	// transmit the result to merge
	tx(0,0,1,128);
	fence(0);
	fence(1);
	rx(1,128,0,128);
	compute_add(1,0,0,128,128);
	store_buffer(1,0,0,128);
	store_output(1,0,0x200,128);
}
```

3. Max pool task:

   *The example has not considered padding. If so, `load_rf_imm` is needed.*
```c
int main(){
	// compute maxpool
	// using the following APIs to write a maxpool
	// 4 * 4 matrix, applied to 2 * 2 kernel, stride = 2
	uint32_t length_size = 4;
	uint32_t kernel_size = 2;
	uint32_t stride = 2;
	load_input(0, 0, 0, length_size*length_size);
	for (int i=0;i<(length_size-kernel_size)/stride+1;i++){
		for (int j=0;j<(length_size-kernel_size)/stride+1;j++){
			load_rf(0, 0, length_size*kernel_size*i+kernel_size*j, kernel_size);
			load_rf(0, kernel_size, length_size*kernel_size*i+length_size+kernel_size*j, kernel_size);
			compute_max(0, 0x100+kernel_size*i+j, 0x00, kernel_size*kernel_size);
		}
	}
	store_output(0, 0x100, 0x200, 4);
}
```

4. Average pool task:

   *The example has not considered padding. If so, `load_rf_imm` is needed.*
```c
int main(){
	// compute maxpool
	// using the following APIs to write a maxpool
	// 4 * 4 matrix, applied to 2 * 2 kernel, stride = 2
	uint32_t length_size = 4;
	uint32_t kernel_size = 2;
	uint32_t stride = 2;
	uint32_t bw_act = 8;
	uint32_t ret_addr = 0x100;
	uint32_t output_size_x = (length_size-kernel_size)/stride+1;
	uint32_t output_size_y = (length_size-kernel_size)/stride+1;
	load_input(0, 0, 0, length_size*length_size);
	for (int j=0;j<output_size_y;j++){
		for (int i=0;i<output_size_x;i++){
			// compute each scalar here
			for (int k=0;k<kernel_size;k++){
				load_rf(0, k, length_size*stride*j+stride*i+length_size*k, kernel_size);
				compute_sum(0, k, k, kernel_size);
			}
			compute_sum(0, 0, 0, kernel_size);
			load_rf_imm(0, 1, kernel_size*kernel_size, 1);
			compute_div(0, 0, 0, 1, 1);
			store_buffer(0, ret_addr+output_size_y*j+i, 0, 1);
		}
	}
	store_output(0, ret_addr, 0x200, output_size_x*output_size_y);
}
```