### OpenCL FFT (Fast Fourier Transform) ###

This an version is adapted from the original available on Apple's website,
http://developer.apple.com/mac/library/samplecode/OpenCL_FFT/index.html
to build across platforms and install a library.
  -- Matt McCormick 08 February 2010

===========================================================================
DESCRIPTION:

This example shows how OpenCL can be used to compute FFT. Algorithm implemented
is described in the following references

1) Fitting FFT onto the G80 Architecture
   by Vasily Volkov and Brian Kazian
   University of California, Berkeley, May 19, 2008
   http://www.cs.berkeley.edu/~kubitron/courses/cs258-S08/projects/reports/project6_report.pdf
   
2) High Performance Discrete Fourier Tansforms on Graphics Processors
   by Naga K. Govindaraju, Brandon Lloyd, Yuri Dotsenko, Burton Smith, and John Manferdelli
   Supercomputing 2008.
   http://portal.acm.org/citation.cfm?id=1413373
   
Current version only supports power of two transform sizes however it should be straight forward
to extend the sample to non-power of two but power of base primes like 3, 5, 7. 

Current version supports 1D, 2D, 3D batched transforms. 

Current version supports both in-place and out-of-place transforms.

Current version supports both forward and inverse transform.

Current version supports both plannar and interleaved data format.

Current version only supports complex-to-complex transform. For real 
transform, one can use plannar data format with imaginary array mem set to zero. 

Current version only supports transform on GPU device. Accelerate framework can be used on CPU.

Current version supports sizes that fits in device global memory although "Twist Kernel" is 
included in fft plan if user wants to virtualize (implement sizes larger than what can fit 
in GPU global memory).

Users can dump all the kernels and global, local dimensions with which these kernels are run 
so that they can not only inspect/modify these kernels and understand how FFT is being 
computed on GPU, but also create their own stand along app for executing FFT of size of
their interest.

For any given signal size n, sample crates a clFFT_Plan, that encapsulates the kernel string, 
associated compiled cl_program. Note that kernel string is generated at runtime based on 
input size, dimension (1D, 2D, 3D) and data format (plannar or interleaved) along with some 
device depended parameters encapsulated in the clFFT_Plan. These device dependent parameters 
are set such that kernel is generated for high performance meeting following requirements

   1) Access pattern to global memory (of-chip DRAM) is such that memory transaction 
      coalesceing is achieved if device supports it thus achieving full bandwidth
   2) Local shuffles (matrix transposes or data sharing among work items of a workgroup)
      are band conflict free if local memory is banked.
   3) Kernel is fully optimized for memory hierarcy meaning that it uses GPU's large 
      vector register file, which is fastest, first before reverting to local memory 
      for data sharing among work items to save global DRAM bandwidth and only then 
      reverts to global memory if signal size is such that transform cannnot be computed
      by singal workgroup and thus require global communation among work groups.
      
Users can modify these parameters to get best performance on their particular GPU.     

Users how really want to understand the details of implementation are highly encouraged 
to read above two references but here is a high level description.
At a higher the algorithm decomposes signal of length N into factors as 

                   N = N1 x N2 x N3 x N4 x .... Nn
                   
where the factors (N1, ....., Nn) are sorted such that N1 is largest. It thus decomposes 
N into n-dimensional matrix. It than applies fft along each dimension, multiply by twiddle
factors and transposes the matrix as follow 

                      N2 x N3 x N4 x ............ x Nn x N1   (fft along N1 and transpose)
                      N3 x N4 x N5 x ....    x Nn x N2 x N1   (fft along N2 and transpose)
                      N4 x N5 x N6 x .. x Nn x N3 x N2 x N1   (fft along N3 and transpose)
                      
                      ......
                     Nn x Nn-1 x Nn-2 x ........ N3 x N2 x N1 (fft along Nn and transpose)
                     
 Decomposition is such that algorithm is fully optimized for memory hierarchy. N1 (largest
 base radix) is constrained by maximum register usage by work item (largest size of in-register 
 fft) and product N2 x N3 .... x Nn determine the maximum size of work group which is constrained
 by local memory used by work group (local memory is used to share data among work items i.e.
 local transposes). Togather these two parameters determine the maximum size fft that can be 
 computed by just using register file and local memory without reverting to global memory 
 for transpose (i.e. these sizes do not require global transpose and thus no inter work group 
 communication). However, for larger sizes, global communication among workgroup is required
 and multiple kernel launches are needed depending on the size and the base radix used. 
 
 For details of parameters user can play with, please see the comments in fft_internal.h
 and kernel_string.cpp, which has the main kernel generator functions ... especially
 see the comments preceeding function getRadixArray and getGlobalRadixInfo.
 User can adjust these parameters you achieve best performance on his device. 

Description of API Calls
=========================
clFFT_Plan clFFT_CreatePlan( cl_context context, clFFT_Dim3 n, clFFT_Dimension dim, clFFT_DataFormat dataFormat, cl_int *error_code );

This function creates a plan and returns a handle to it for use with other functions below. 
context    context in which things are happening
n          n.x, n.y, n.z contain the dimension of signal (length along each dimension)
dim        much be one of clFFT_1D, clFFT_2D, clFFT_3D for one, two or three dimensional fft
dataFormat much be either clFFT_InterleavedComplexFormat or clFFT_SplitComplexFormat for either interleaved or plannar data (real and imaginary)
error_code pointer for getting error back in plan creation. In case of error NULL plan is returned
==========================
void clFFT_DestroyPlan( clFFT_Plan plan );

Function to release/free resources
==========================
cl_int clFFT_ExecuteInterleaved( cl_command_queue queue, clFFT_Plan plan, cl_int batchSize, clFFT_Direction dir, 
								 cl_mem data_in, cl_mem data_out,
								 cl_int num_events, cl_event *event_list, cl_event *event );
								 
Function for interleaved fft execution.
queue      command queue for the device on which fft needs to be executed. It should be present in the context for this plan was created
plan       fft plan that was created using clFFT_CreatePlan
batchSize  size of the batch for batched transform
dir        much be either clFFT_Forward or clFFT_Inverse for forward or inverse transform
data_in    input data
data_out   output data. For in-place transform, pass same mem object for both data_in and data_out
num_events, event_list and event are for future use for letting fft fit in other CL based application pipeline through event dependency.
Not implemented in this version yet so these parameters are redundant right now. Just pass NULL.

=========================
cl_int clFFT_ExecutePlannar( cl_command_queue queue, clFFT_Plan plan, cl_int batchSize, clFFT_Direction dir, 
							 cl_mem data_in_real, cl_mem data_in_imag, cl_mem data_out_real, cl_mem data_out_imag,
							 cl_int num_events, cl_event *event_list, cl_event *event );
							 
Same as above but for plannar data type.							 
=========================
cl_int clFFT_1DTwistInterleaved( clFFT_Plan plan, cl_mem mem, size_t numRows, size_t numCols, size_t startRow, clFFT_Direction dir );

Function for applying twist (twiddle factor multiplication) for virtualizing computation of very large ffts that cannot fit into global
memory at once but can be decomposed into many global memory fitting ffts followed by twiddle multiplication (twist) followed by transpose
followed by again many global memory fitting ffts.

=========================
cl_int clFFT_1DTwistPlanner( clFFT_Plan plan, cl_mem mem_real, cl_mem mem_imag, size_t numRows, size_t numCols, size_t startRow, clFFT_Direction dir );

Same fucntion as above but for plannar data
=========================	

void clFFT_DumpPlan( clFFT_Plan plan, FILE *file);	

Function to dump the plan. Passing stdout to file prints out the plan to standard out. It prints out
the kernel string and local, global dimension with which each kernel is executed in this plan.
						
==================================================================================
IMPORTANT NOTE ON PERFORMANCE:

Currently there are a few known performance issues (bug) that this sample has discovered
in rumtime and code generation that are being actively fixed. Hence, for sizes >= 1024, 
performance is much below the expected peak for any particular size. However, we have 
internally verified that once these bugs are fixed, performance should be on par with 
expected peak. Note that these are bugs in OpenCL runtime/compiler and not in this
sample.

===========================================================================
BUILD REQUIREMENTS:

Mac OS X v10.6 or later

If you are running in Xcode, be sure to pass file name "param.txt". You can do that
by double clicking OpenCL_FFT under executable and then click on Argument tab and 
add ./../../param.txt under "Arguments to be passed on launch" section. 

===========================================================================
RUNTIME REQUIREMENTS:

. Mac OS X v10.6 or later with OpenCL 1.0
. For good performance, device should support local memory. 
  FFT performance critically depend on how efficiently local shuffles 
  (matrix transposes) using local memory to reduce external DRAM bandwidth
  requirement.

===========================================================================
PACKAGING LIST:

AccelerateError.pdf
clFFT.h
Error.pdf
fft_base_kernels.h
fft_execute.cpp
fft_internal.h
fft_kernelstring.cpp
fft_setup.cpp
main.cpp
Makefile
OpenCL_FFT.xcodeproj
OpenCLError.pdf
param.txt
procs.h
ReadMe.txt

===========================================================================
CHANGES FROM PREVIOUS VERSIONS:

Version 1.0
- First version.

===========================================================================
Copyright (C) 2008 Apple Inc. All rights reserved.
