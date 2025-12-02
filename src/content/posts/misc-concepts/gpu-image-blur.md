---
title: GPU Accelerated Image Blurring Kernel
published: 2025-12-01
description: 'Implementation of a Image Processing Kernel that is accelerated through the use of the CUDA API.'
image: ''
tags: [
'gpu-programming'
]
category: 'Programming'
draft: false
lang: 'en'
---

# Motivation

In this project, we will be exploring how to implement a Gaussian Blurring program that will inherently alter each pixel of an image in order to take a new pixel value that will be affected by a pixel's surrounding neighbors. 
![Image Alt Text](https://github.com/MrGrinchFx/Gaussian-Blur/raw/main/building.png) ![Image Alt Text](https://github.com/MrGrinchFx/Gaussian-Blur/raw/main/blurredBuilding.png)

A naive approach to “blurring” an image would be to visit each pixel take the greyscale value of the surrounding 8 pixels and average it with the current pixel. However, this strategy leads to a muddier and less appealing result and is often not used in the real world. Instead, a different approach, the Gaussian Blur, is more common in real-world applications such as Blender or photo editors.

# Strategy

A Gaussian blur is a 2-D convolution operator that is used to blur images in order to reduce detail and noise. A Gaussian blur is technically represented as a matrix of weights, also called a kernel or a mask. The convolution is the process of adding each pixel of the image to its local neighbors, weighted by the matrix. The example below shows how a 3x3 kernel matrix is applied to the pixel located at coordinate (1,1) (Figure 1). In order to perform an entire 2-D convolution, such an operation would need to be applied to each pixel of the original image.
Note that when applying the kernel matrix towards the edges or the corners, the “missing” pixels should be replaced by the nearest existing pixels (those directly on the edges, or corners).

![Figure 1](https://github.com/MrGrinchFx/Gaussian-Blur/raw/main/gaussianBlur.JPG)

![Figure 2](https://github.com/MrGrinchFx/Gaussian-Blur/raw/main/equation.JPG)
## Gaussian kernel

The kernel matrix for a Gaussian blur is created by the 2-D Gaussian function depicted in Figure 2.
x is the distance from the origin in the horizontal axis,
y is the distance from the origin in the vertical axis,
and σ (sigma) is the standard deviation of the Gaussian distribution.
In order to generate a proper kernel matrix, the only information needed is the sigma value. The order of the matrix is derived from sigma



# CPU Implementation

For the first part of the project, we will first be implementing a serial version of the program in hopes that there will be an avenue that we can further improve the program by employing thousands of slaves (GPU threads) to compute our output file blazingly fast. First, we have to do the dirty work of taking the proper command line parameters and storing them accordingly, open the input file as an unsigned char array, and allocate enough space for our output file. After this, we can then begin to create our kernel_matrix which is going to be created using the convolution equation. 
```c
float* create_gaussian_kernel(int* order, float sigma) {
   // calculate size of the kernel matrix based on command line inputs
   int half_size = (int)(sigma * 3);
   *order = 2 * half_size + 1;

   float* kernel = (float*)malloc((*order) * (*order) * sizeof(float));
   float sum = 0.0;
   for (int x = -half_size; x <= half_size; ++x) {
       for (int y = -half_size; y <= half_size; ++y) {
           int idx = (x + half_size) * (*order) + (y + half_size);
           kernel[idx] = expf(-(x*x + y*y) / (2 * sigma * sigma));
           sum += kernel[idx];
       }
   }

   for (int i = 0; i < (*order) * (*order); ++i) {
       kernel[i] /= sum;
   }

   return kernel;
}
```
After creating this matrix, we will then apply the Gaussian blur by going through the whole input image while also handling the edge cases that arise when the kernel matrix extends past the boundaries of the input matrix. 
```c
void apply_gaussian_blur(unsigned char* input, unsigned char* output, int width, int height, float* kernel, int order) {
   int half_size = order / 2;
   // iterate through 2D image
   for (int i = 0; i < height; ++i) {
       for (int j = 0; j < width; ++j) {
           float sum = 0.0;
           //iterate through kernel matrix
           for (int ki = -half_size; ki <= half_size; ++ki) {
               for (int kj = -half_size; kj <= half_size; ++kj) {
                   int ni = i + ki;
                   int nj = j + kj;
                   //Handling Edge Case when a pixel has missing neighbors
                   if (ni < 0) ni = 0;
                   if (nj < 0) nj = 0;
                   if (ni >= height) ni = height - 1;
                   if (nj >= width) nj = width - 1;
                   int kernel_idx = (ki + half_size) * order + (kj + half_size);
                   sum += input[ni * width + nj] * kernel[kernel_idx];
               }
           }
           output[i * width + j] = (unsigned char)sum;
       }
   }
}
```


After we have passed through the whole array, via a very large double-for-loop setup, we can now take this new output matrix and format it into a PGM file and return it.
```c
void write_pgm(const char* filename, const unsigned char* data, int width, int height, int maxval) {
   FILE* file = fopen(filename, "wb");
   if (!file) {
       exit(EXIT_FAILURE);
   }
   // Important for Binary PGM files
   fprintf(file, "P5\n%d %d\n%d\n", width, height, maxval);
   fwrite(data, 1, width * height, file);
   fclose(file);
}
```

 In order to test the performance of this serial program, we will subject it to 4 different image files of varying sizes to see how performant our program is currently. Below are the results:

## Results

![Serial Results](https://github.com/MrGrinchFx/Gaussian-Blur/raw/main/serialResults.JPG)

# GPU Implementation
As we can see from the results, the serial implementation struggles to take on larger-sized input maps and has a quadratically growing runtime. In order to combat this, we can use the help of the GPU’s thousands of cores to help tackle this very computationally intensive program. In the following section.

To parallelize this further using our GPU computing power, we will have to first allocate each image file its own pieces of memory in the GPU as well as the image’s kernel to send it over to the device (GPU) from the host (CPU). 
```c
void apply_gaussian_blur(unsigned char* input, unsigned char* output, int width, int height, float* kernel, int order) {
   unsigned char *d_input, *d_output;
   float *d_kernel;
   size_t size = width * height * sizeof(unsigned char);
   size_t size_kernel = order * order * sizeof(float);

//cuda_check is just a simple macro function that checks the return of cuda functions to check if they were completed successfully.

   //allocate memory for input and output arrays as well as kernel
   cuda_check(cudaMalloc(&d_input, size));
   cuda_check(cudaMalloc(&d_output, size));
   cuda_check(cudaMalloc(&d_kernel, size_kernel));
   
   //Copy over input and kernel arrays to the GPU
   cuda_check(cudaMemcpy(d_input, input, size, cudaMemcpyHostToDevice));
   cuda_check(cudaMemcpy(d_kernel, kernel, size_kernel, cudaMemcpyHostToDevice));
    
   //define the thread block dimensions (
   dim3 threadsPerBlock(16, 16);
   dim3 numBlocks((width + threadsPerBlock.x - 1) / threadsPerBlock.x, (height + threadsPerBlock.y - 1) / threadsPerBlock.y);
   
   // call the kernel function (not related to image kernel)
   apply_gaussian_blur_kernel<<<numBlocks, threadsPerBlock>>>(d_input, d_output, width, height, d_kernel, order);


   //When done send result back to CPU code
   cuda_check(cudaMemcpy(output, d_output, size, cudaMemcpyDeviceToHost));
   // free memory
   cuda_check(cudaFree(d_input));
   cuda_check(cudaFree(d_output));
   cuda_check(cudaFree(d_kernel));
}
```



Once we allocate the CUDA memory necessary, we will then call upon the Gaussian blur function as the global function that will run on each of the CUDA threads that we’ll employ to do our computations. Once we send that maps to the device we will allow the CUDA threads to compute their section of the map and send it back to the host. Once the host receives the chunk of the output map, we will then recombine and package the output map as a PGM image.
```c
__global__ void apply_gaussian_blur_kernel(unsigned char* input, unsigned char* output, int width, int height, float* kernel, int order) {
   int half_size = order / 2;
   int i = blockIdx.y * blockDim.y + threadIdx.y;
   int j = blockIdx.x * blockDim.x + threadIdx.x;

   if (i < height && j < width) {
       float sum = 0.0;
       for (int ki = -half_size; ki <= half_size; ++ki) {
           for (int kj = -half_size; kj <= half_size; ++kj) {
               int ni = i + ki;
               int nj = j + kj;

               if (ni < 0) ni = 0;
               if (nj < 0) nj = 0;
               if (ni >= height) ni = height - 1;
               if (nj >= width) nj = width - 1;

               int kernel_idx = (ki + half_size) * order + (kj + half_size);
               sum += input[ni * width + nj] * kernel[kernel_idx];
           }
       }
       output[i * width + j] = (unsigned char)sum;
   }
}
```

## Results

In the same manner that we benchmarked the previous serial version, we will also run our parallelized version against the four variably sized input images and determine its performance based on the time it takes to return an output. Below are the average times of both implementations placed side-by-side for comparison:



![CUDA vs Serial Results](https://github.com/MrGrinchFx/Gaussian-Blur/raw/main/CudaVsSerial.JPG)



As can be seen from the results, we are now seeing much better results as the total runtime of larger files has been drastically decreased. Although the program still grows quadratically, since we changed nothing about the algorithm, we are still able to experience major speedups due to the usage of CUDA cores in our GPU.

For completeness, here is the main function for the CUDA implementation (similar to serial):


```c
//Sample Command: ./gaussian_blur_cuda.cu image.pgm output.pgm 4
int main(int argc, char* argv[]) {
   if (argc != 4) {
       return -1;
   }
   const char* input_filename = argv[1];
   const char* output_filename = argv[2];
   float sigma = atof(argv[3]);

   int width, height, maxVal;
   unsigned char* input_image = read_pgm(input_filename, &width, &height, &maxVal);
   int order;
   float* kernel = create_gaussian_kernel(&order, sigma);
   unsigned char* output_image = (unsigned char*)malloc(width * height);
   
   apply_gaussian_blur(input_image, output_image, width, height, kernel, order);
   
   write_pgm(output_filename, output_image, width, height, maxVal);
  
   free(input_image);
   free(output_image);
   free(kernel);
   return 0;
}
```

