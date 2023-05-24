.PHONY : all clean

all: dbm_miniapp.x

clean:
	rm -fv *.o */*.o ../offload/*.o

realclean: clean
	rm -fv *.x

CFLAGS := -fopenmp -g -O3 -march=native -Wall -Wextra -Wno-vla-parameter

HIP_PLATFORM = $(shell hipconfig --platform)

GPU_CC = hipcc
ifeq (${HIP_PLATFORM}, nvidia)
GPU_CC = nvcc
ARCH := sm_80 #update the ARCH based on the current platform 
HIP_BACKEND = __HIP_PLATFORM_NVIDIA__
GPU_FLAGS := -g -O3 -lineinfo -arch $(ARCH) -Wno-deprecated-gpu-targets -Xcompiler "$(CFLAGS)" -D__OFFLOAD_CUDA
CFLAGS += -D__OFFLOAD_CUDA
else ifeq (${HIP_PLATFORM}, amd)
ARCH := gfx90a
HIP_PATH := $(ROCM_PATH)
HIP_BACKEND = __HIP_PLATFORM_AMD__
LIBARCHFLAG := -lamdhip64 -lrocblas
GPU_FLAGS := -fPIE -g -O3 --offload-arch=$(ARCH) -Wall -Wextra -Werror -D__OFFLOAD_HIP -D$(HIP_BACKEND)
else ifeq (${HIP_PLATFORM}, intel)
CFLAGS += -I${HIP_PATH}/include
HIP_BACKEND = __HIP_PLATFORM_SPIRV__
LIBARCHFLAG := -L${MKLROOT}/lib/intel64
GPU_FLAGS := -D__OFFLOAD_HIP
CFLAGS += -I$(MKLROOT)/include
endif

ifeq (${HIP_PLATFORM}, intel)
	OBJECT_C := dbm_multiply_cpu.o
	LIBS += -L${MKLROOT}/lib/intel64 -Wl,--no-as-needed -lmkl_intel_lp64 -lmkl_sequential -lmkl_core -lpthread -lm -ldl
else
	LIBS += -fopenmp -ldl -lstdc++ -lc -lm
	LIBS += -L${HIP_PATH}/hipblas/lib -L${HIP_PATH}/lib $(LIBARCHFLAG) -lhipblas
endif

ALL_HEADERS := $(shell find . -name "*.h") $(shell find ../offload/ -name "*.h")
ALL_OBJECTS := ../offload/offload_library.o \
        dbm_distribution.o \
        dbm_library.o \
        dbm_matrix.o \
        dbm_mempool.o \
        dbm_mpi.o \
        dbm_multiply.o \
        dbm_multiply_comm.o \
        dbm_shard.o

OBJECT_C := dbm_multiply_cpu.o

ifeq (${HIP_PLATFORM}, amd)
CFLAGS += -I${HIP_PATH}/include -D__OFFLOAD_HIP -D$(HIP_BACKEND)
endif

ALL_OBJECTS += dbm_multiply_gpu.o
ALL_OBJECTS += $(OBJECT_C)

ifeq (${HIP_PLATFORM}, nvidia)
LIBS += -lcudart
%.o: %.cu $(ALL_HEADERS)
	cd $(dir $<); $(GPU_CC) -c $(GPU_FLAGS) $(notdir $<)
else
%.o: %.cpp $(ALL_HEADERS)
	cd $(dir $<); $(GPU_CC) -c $(GPU_FLAGS) $(notdir $<)
endif

%.o: %.c $(ALL_HEADERS)
	cd $(dir $<); $(CC) -c $(CFLAGS) $(notdir $<)

$(OBJECT_C): dbm_multiply_cpu.c
	cd $(dir $<); $(CC) -c $(CFLAGS) $(notdir $<)

dbm_miniapp.x: dbm_multiply_gpu_kernel.o dbm_miniapp.o $(ALL_OBJECTS)
	$(GPU_CC) $(CFLAGS) -o $@ $^ $(LIBS)

#EOF
