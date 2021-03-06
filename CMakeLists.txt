#Requires the following environment variables:
#TALSH_GPU
#TALSH_GPU_ARCH
#TALSH_BLAS
#TALSH_FINE_TIMING (Optional)
#nvcc in system PATH

cmake_minimum_required(VERSION 3.7 FATAL_ERROR)
project (TALSH C CXX Fortran)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

include(FindCUDA)
find_package(MPI REQUIRED)

#TODO:
#CUDA FLAGS - cuTT
#Make Blas optional
#Cray, Clang compiler flags
#debug/dev mode flags
#fpic flags
# DF := -WF, when using IBM compilers

#set(TALSH_GPU_ARCH 35)

if (TALSH_GPU)
	set(TALSH_GPU_DEFS "CUDA_ARCH=${TALSH_GPU_ARCH}0")
else()
	set(TALSH_GPU_DEFS NO_GPU)
endif()

set(TALSH_CXX_FLAGS ${CMAKE_CXX_FLAGS} -O3)
set(TALSH_Fortran_FLAGS ${CMAKE_Fortran_FLAGS} -O3)

if(CMAKE_Fortran_COMPILER_ID STREQUAL "Intel")
  set(TALSH_Fortran_FLAGS ${TALSH_Fortran_FLAGS} -fpic -fpp -vec-threshold4 -qopenmp -mkl=parallel)
elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "GNU")
  set(TALSH_Fortran_FLAGS ${TALSH_Fortran_FLAGS} -fPIC -fopenmp)
elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "PGI")
  set(TALSH_Fortran_FLAGS ${TALSH_Fortran_FLAGS} -fpic -mp -Mcache_align -Mstandard -Mallocatable=03)
elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "XL")
  set(TALSH_Fortran_FLAGS ${TALSH_Fortran_FLAGS} -qpic=large -qsmp=omp)
elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "Cray")
  set(TALSH_Fortran_FLAGS ${TALSH_FORTRAN_FLAGS} -hpic)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "PGI")
  set(TALSH_CXX_FLAGS ${TALSH_CXX_FLAGS} -silent -w -Mnovect)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Cray")
  set(TALSH_CXX_FLAGS ${TALSH_CXX_FLAGS} -h std=c++11)
endif()

if (NOT TALSH_BLAS)
       set(NO_BLAS NO_BLAS)
endif()

set(TALSH_CXX_COMPILE_DEFS ${TALSH_GPU_DEFS} ${NO_BLAS} NO_AMD NO_PHI LINUX)
set(TALSH_Fortran_COMPILE_DEFS ${TALSH_GPU_DEFS} ${NO_BLAS} NO_AMD NO_PHI LINUX)

if (TALSH_GPU)
       set(TALSH_INCLUDES ${MPI_C_INCLUDE_PATH} ${MPI_Fortran_INCLUDE_PATH} ${CUDA_INCLUDE_DIRS})
else()
       set(TALSH_INCLUDES ${MPI_C_INCLUDE_PATH} ${MPI_Fortran_INCLUDE_PATH})
endif()

set (TALSH_FORTRAN_SOURCES
	 dil_basic.F90
	 stsubs.F90
	 combinatoric.F90
	 symm_index.F90
	 timers.F90
	 tensor_algebra.F90
	 tensor_algebra_cpu.F90
	 tensor_algebra_cpu_phi.F90
	 tensor_dil_omp.F90
	 talshf.F90
	 )

set (TALSH_CXX_SOURCES
	timer.cpp
	byte_packet.cpp
	tensor_method.cpp
	mem_manager.cpp
	talshc.cpp
	talsh_task.cpp
	talshxx.cpp
        )

if(TALSH_GPU)
	set(TALSH_CUDA_SOURCES tensor_algebra_gpu_nvidia.cu)
else()
	configure_file(tensor_algebra_gpu_nvidia.cu ${PROJECT_SOURCE_DIR}/tensor_algebra_gpu_nvidia.cpp COPYONLY)
	set(TALSH_CXX_SOURCES ${TALSH_CXX_SOURCES} tensor_algebra_gpu_nvidia.cpp)
endif()

#set(TALSH_CUDA_FLAGS -O3 -lineinfo -w -D_FORCE_INLINES
#                                               -std=c++11 -Xptxas --warn-on-spills -arch=sm_${TALSH_GPU_ARCH})

if(CMAKE_CXX_COMPILER_ID STREQUAL "Cray")
	set(TALSH_CUDA_FLAGS  ${TALSH_CUDA_FLAGS} -O3 -lineinfo -w -D_FORCE_INLINES
		-Xptxas --warn-on-spills;-gencode arch=compute_${TALSH_GPU_ARCH},code=sm_${TALSH_GPU_ARCH}; -ccbin /usr/bin/g++)
else()
set(TALSH_CUDA_FLAGS  ${TALSH_CUDA_FLAGS} -O3 -lineinfo -w -D_FORCE_INLINES
                -std=c++11 -Xptxas --warn-on-spills;-gencode arch=compute_${TALSH_GPU_ARCH},code=sm_${TALSH_GPU_ARCH}; -ccbin /usr/bin/g++)
endif()

if(TALSH_FINE_TIMING)
  set(TALSH_CUDA_FLAGS ${TALSH_CUDA_FLAGS} -DGPU_FINE_TIMING)
endif()

add_library(talsh_cxx OBJECT ${TALSH_CXX_SOURCES})
target_include_directories(talsh_cxx PUBLIC ${TALSH_INCLUDES})
target_compile_definitions(talsh_cxx PRIVATE ${TALSH_CXX_COMPILE_DEFS})
target_compile_options(talsh_cxx PUBLIC ${TALSH_CXX_FLAGS})


if(TALSH_GPU)
	add_library(talsh_fort OBJECT ${TALSH_FORTRAN_SOURCES})
	target_include_directories(talsh_fort PUBLIC ${TALSH_INCLUDES})
	target_compile_definitions(talsh_fort PRIVATE ${TALSH_Fortran_COMPILE_DEFS})
	target_compile_options(talsh_fort PUBLIC ${TALSH_Fortran_FLAGS})

	set_source_files_properties(${TALSH_CUDA_SOURCES} PROPERTIES CUDA_SOURCE_PROPERTY_FORMAT OBJ)
	cuda_add_library(talsh ${TALSH_CUDA_SOURCES} $<TARGET_OBJECTS:talsh_cxx> $<TARGET_OBJECTS:talsh_fort> OPTIONS ${TALSH_CUDA_FLAGS})
	
	#cuda_wrap_srcs(talsh_cuda_ptx PTX tensor_algebra_gpu_nvidia ${TALSH_CUDA_SOURCES} OPTIONS ${TALSH_CUDA_FLAGS})
else()
	add_library(talsh STATIC ${TALSH_FORTRAN_SOURCES} $<TARGET_OBJECTS:talsh_cxx>)
	target_include_directories(talsh PUBLIC ${TALSH_INCLUDES})
	target_compile_definitions(talsh PRIVATE ${TALSH_Fortran_COMPILE_DEFS})
	target_compile_options(talsh PUBLIC ${TALSH_Fortran_FLAGS})
endif()


set(TALSH_HEADERS
	timer.h
	byte_packet.h
	tensor_algebra.h
	mem_manager.h
	talsh_complex.h
	talsh.h
	tensor_method.hpp
	talsh_task.hpp
	talshxx.hpp
        )

install(TARGETS talsh
        LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
        ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)

install(FILES ${TALSH_HEADERS}
        DESTINATION ${CMAKE_INSTALL_PREFIX}/include)
