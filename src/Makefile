# Required paths
ifndef LEGION_DIR
  $(error LEGION_DIR is not set)
endif
ifndef HTR_DIR
  $(error HTR_DIR is not set)
endif

# OS-specific options
ifeq ($(shell uname),Darwin)
  DYNLINK_PATH := DYLD_LIBRARY_PATH
else
  DYNLINK_PATH := LD_LIBRARY_PATH
endif

# CUDA options
USE_CUDA ?= 1

# OpenMP options
USE_OPENMP ?= 1

# HDF options
export USE_HDF ?= 1
export HDF_HEADER ?= hdf5.h
HDF_LIBNAME ?= hdf5

# C compiler options
CFLAGS += -O2 -Wall -Werror -fno-strict-aliasing -I$(LEGION_DIR)/runtime -I$(LEGION_DIR)/bindings/regent
CXXFLAGS += -std=c++11 -O3 -Wall -Werror -fno-strict-aliasing -I$(LEGION_DIR)/runtime -I$(LEGION_DIR)/bindings/regent

# Regent options
export INCLUDE_PATH := .
ifdef HDF_ROOT
  export INCLUDE_PATH := $(INCLUDE_PATH);$(HDF_ROOT)/include
  export $(DYNLINK_PATH) := $($(DYNLINK_PATH)):$(HDF_ROOT)/lib
endif
REGENT := $(LEGION_DIR)/language/regent.py
REGENT_FLAGS := -fflow 0 -finner 1
ifeq ($(DEBUG), 1)
  REGENT_FLAGS += -fcuda 0 -fbounds-checks 1
else
ifeq ($(USE_CUDA), 1)
  REGENT_FLAGS += -fcuda 1 -fcuda-offline 1
else
  REGENT_FLAGS += -fcuda 0
endif
endif
ifeq ($(USE_OPENMP), 1)
  REGENT_FLAGS += -fopenmp 1
else
  REGENT_FLAGS += -fopenmp 0
endif

# Link flags
ifdef CRAYPE_VERSION
  LINK_FLAGS += -Bdynamic
  LINK_FLAGS += $(CRAY_UGNI_POST_LINK_OPTS) -lugni
  LINK_FLAGS += $(CRAY_UDREG_POST_LINK_OPTS) -ludreg
endif
LINK_FLAGS += -L$(LEGION_DIR)/bindings/regent -lregent
ifdef HDF_ROOT
  LINK_FLAGS += -L$(HDF_ROOT)/lib
endif
ifeq ($(USE_HDF), 1)
  LINK_FLAGS += -l$(HDF_LIBNAME)
endif
LINK_FLAGS += -lm
 
ifeq ($(DEBUG), 1)
  REGENT_FLAGS += -g
  CFLAGS += -g
  CXXFLAGS += -g
  LINK_FLAGS += -g
endif

# Get git commit version and date
SOLVER_VERSION := ""$(shell git --no-pager describe --tags --always --dirty)" committed on "$(firstword $(shell git --no-pager show --date=short --format="%ad" --name-only))""

LEGION_VERSION := ""$(shell cd $(LEGION_DIR); git --no-pager describe --tags --always --dirty)" committed on "$(firstword $(shell cd $(LEGION_DIR); git --no-pager show --date=short --format="%ad" --name-only))""

.PHONY: default all clean force
.PRECIOUS: prometeo_%.o solver_version~ legion_version~ version.rg
.SUFFIXES: .rg .cc .h .o

default: prometeo_ConstPropMix.exec prometeo_AirMix.exec

Common= hdf_helper.rg util-desugared.rg math_utils.rg prometeo_const.rg prometeo_macro.rg \
		prometeo_grid.rg prometeo_metric.rg prometeo_IO.rg prometeo_cfl.rg prometeo_chem.rg \
		prometeo_init.rg prometeo_variables.rg prometeo_average.rg prometeo_rk.rg \
		prometeo_flux.rg prometeo_rhs.rg prometeo_operators.rg prometeo_stat.rg \
		prometeo_bc.rg prometeo_profiles.rg
ConstPropMix= ConstPropMix.rg
AirMix= Species.rg Reaction.rg AirMix.rg

clean:
	$(RM) *.exec *.o *-desugared.rg config_schema.h version.rg solver_version~ legion_version~

%-desugared.rg: %.rg
	./desugar.py $< > $@

prometeo_%.exec: prometeo_%.o prometeo_mapper.o config_schema.o json.o
	$(CXX) -o $@ $^ $(LINK_FLAGS)

.SECONDEXPANSION:
prometeo_%.o: version.rg prometeo-desugared.rg prometeo_mapper.h config_schema.o $$(wildcard $$**) $(Common)
	EOS="$*" $(REGENT) prometeo-desugared.rg $(REGENT_FLAGS)

prometeo_mapper.o: prometeo_mapper.cc prometeo_mapper.h config_schema.o
	$(CXX) $(CXXFLAGS) -c -o  $@ $<

config_schema.o: process_schema.rg config_schema.lua json.h util-desugared.rg
	$(REGENT) process_schema.rg config_schema.lua $(REGENT_FLAGS)

json.o: json.c json.h
	$(CC) $(CFLAGS) -c -o $@ $<

# recompile version.h dependants when GIT_VERSION changes, uses temporary file version~
solver_version~: force
	@echo "$(SOLVER_VERSION)" | cmp -s - $@ || echo "$(SOLVER_VERSION)" > $@

legion_version~: force
	@echo "$(LEGION_VERSION)" | cmp -s - $@ || echo "$(LEGION_VERSION)" > $@

version.rg: solver_version~ legion_version~
	@touch $@
	@echo Solver version $(SOLVER_VERSION)
	@echo Legion version $(LEGION_VERSION)
	@echo "-- This file is automatically generated, please do not touch it"                > $@
	@echo ""                                                                              >> $@
	@echo "local Exports = {}"                                                            >> $@
	@echo ""                                                                              >> $@
	@echo "local SolverLength = "$(shell echo $(SOLVER_VERSION) | awk "{print length}")"" >> $@
	@echo "Exports.SolverVersion = \""$(SOLVER_VERSION)"\""                               >> $@
	@echo ""                                                                              >> $@
	@echo "local LegionLength = "$(shell echo $(LEGION_VERSION) | awk "{print length}")"" >> $@
	@echo "Exports.LegionVersion = \""$(LEGION_VERSION)"\""                               >> $@
	@echo ""                                                                              >> $@
	@echo "Exports.Length = math.max(SolverLength, LegionLength)"                         >> $@
	@echo ""                                                                              >> $@
	@echo "return Exports"                                                                >> $@
	@echo ""                                                                              >> $@

