cmake_minimum_required( VERSION 2.8 )
project( OPENCL_FFT )

set( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR} )

find_package( OpenCL REQUIRED )

# where all the executables go
set( CMAKE_RUNTIME_OUTPUT_DIRECTORY
  ${OPENCL_FFT_BINARY_DIR}/bin )

include_directories( ${OPENCL_INCLUDE_DIRS} )
include_directories( ${CMAKE_CURRENT_SOURCE_DIR} )

add_library( OpenCL_FFT SHARED
  fft_execute.cpp
  fft_kernelstring.cpp
  fft_setup.cpp
  )
set_target_properties( OpenCL_FFT 
  PROPERTIES SOVERSION 1.3 
  )
target_link_libraries( OpenCL_FFT
  ${OPENCL_LIBRARIES}
  )

install( TARGETS OpenCL_FFT
  LIBRARY
  DESTINATION lib
  )
install( FILES clFFT.h
  DESTINATION include
  )

include( CTest )
if( BUILD_TESTING )
  enable_testing()
  add_executable( OpenCL_FFT_performance
    main.cpp
    )
  target_link_libraries( OpenCL_FFT_performance
    OpenCL_FFT
    )
  add_test( OpenCL_FFT_performance
    ${OPENCL_FFT_BINARY_DIR}/bin/OpenCL_FFT_performance ${CMAKE_CURRENT_SOURCE_DIR}/param.txt
    )
endif()
