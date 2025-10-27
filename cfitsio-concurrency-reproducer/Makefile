################################################################################
# Makefile to build cfitsio (autotools) with ThreadSanitizer and compile the
# enhanced race test program `enhanced_cfitsio_race_test_v2.cpp`.
#
# Usage examples:
#  make all                     # build cfitsio and compile the test executable
#  make build_cfitsio           # only configure & make cfitsio
#  make run                     # run the test with DYLD_LIBRARY_PATH and TSAN options
#  make clean                   # remove test binary and invoke `make clean` in cfitsio
#
# Variables you can override on the make command line or environment:
#  CFITS_DIR   Directory containing the cfitsio sources (default: current dir)
#  JOBS        Parallel jobs for make when building cfitsio (default: 4)
#  CXX, CC     Compilers
#  CXXFLAGS    Flags for C++ compilation (default includes -fsanitize=thread)
#  LDFLAGS     Linker flags (default includes -fsanitize=thread)
#
# Example:
#  make CFITS_DIR=/path/to/cfitsio JOBS=8 CXXFLAGS="-fsanitize=thread -g -O1 -fPIC"
#  make run
################################################################################

## By default assume the cfitsio source is the repository root (one level up from
## this Makefile). Override with CFITS_SRC on the make command line if needed.
CFITS_SRC ?= $(abspath $(CURDIR)/..)

# Directory where out-of-source builds and outputs go
BUILD_DIR ?= $(CURDIR)/build
# Per-cfitsio build directory (inside BUILD_DIR)
CFITS_BUILD_DIR ?= $(BUILD_DIR)/cfitsio_build
JOBS ?= 8
CC ?= cc
CXX ?= c++

# Sanitizer and build flags (override if needed)
# NOTE: -D_REENTRANT is REQUIRED for cfitsio's FFLOCK macros to work!
CFLAGS ?= -fsanitize=thread -g -O1 -fPIC -D_REENTRANT -pthread
CXXFLAGS ?= $(CFLAGS)
LDFLAGS ?= -fsanitize=thread -pthread

# Include and library paths

INCLUDES ?= -I$(CFITS_SRC)
# Where libtool will put the runtime libraries for the out-of-tree build
LIB_DIR ?= $(CFITS_BUILD_DIR)/.libs
LIBS ?= -L$(LIB_DIR) -lcfitsio -lm

# Source and target for the test program (sources are in this directory)
SRCS ?= enhanced_cfitsio_race_test_v2.cpp
TARGET ?= enhanced_cfitsio_test
# Place the test binary in BUILD_DIR
TARGET_PATH := $(BUILD_DIR)/$(TARGET)

.PHONY: all build_cfitsio run clean clean_cfitsio help

all: build_cfitsio $(TARGET_PATH)

## build cfitsio using the repository's configure && make

build_cfitsio:
	@echo "==> Building cfitsio (out-of-source)"
	@mkdir -p $(CFITS_BUILD_DIR)
	@cd $(CFITS_BUILD_DIR) && $(CFITS_SRC)/configure CFLAGS="$(CFLAGS)" LDFLAGS="$(LDFLAGS)" || (echo "configure failed"; exit 1)
	@cd $(CFITS_BUILD_DIR) && make -j$(JOBS)

## Compile the enhanced test program and link against the built cfitsio
$(TARGET_PATH): $(SRCS) | $(BUILD_DIR)
	@echo "==> Compiling $(SRCS) -> $@"
	$(CXX) $(CXXFLAGS) -D_REENTRANT -pthread $(INCLUDES) $^ -o $@ $(LDFLAGS) $(LIBS)

$(BUILD_DIR):
	@mkdir -p $(BUILD_DIR)

## Run the test with the sanitizer runtime options and save output
run: all
	@echo "==> Running $(TARGET) with TSAN and DYLD_LIBRARY_PATH=$(LIB_DIR)"
	DYLD_LIBRARY_PATH=$(LIB_DIR) TSAN_OPTIONS="history_size=7 second_deadlock_stack=1 suppressions=tsan.supp" $(TARGET_PATH) 2>&1 | tee $(BUILD_DIR)/tsan_enhanced_test_output.txt


clean: clean_cfitsio
	@echo "==> Removing built test binary and output"
	rm -f $(TARGET_PATH) $(BUILD_DIR)/tsan_enhanced_test_output.txt


clean_cfitsio:
	@echo "==> Running 'make clean' in $(CFITS_BUILD_DIR) (if available)"
	-cd $(CFITS_BUILD_DIR) && make clean || true
	@echo "==> (Optionally) remove the out-of-source build tree"
	@echo "    to remove completely: rm -rf $(CFITS_BUILD_DIR)"

help:
	@sed -n '1,120p' Makefile
