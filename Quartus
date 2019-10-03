
CUR_MAKEFILE  = $(word $(words $(MAKEFILE_LIST)), $(MAKEFILE_LIST))
CUR_FILE_PATH = $(dir $(CUR_MAKEFILE))

# -----------------------------------------------------------------
# Altera executables
# -----------------------------------------------------------------
#

ifdef QUARTUS_PATH
	QUARTUS_MAP := $(QUARTUS_PATH)/bin64/quartus_map
	QUARTUS_FIT := $(QUARTUS_PATH)/bin64/quartus_fit
	QUARTUS_ASM := $(QUARTUS_PATH)/bin64/quartus_asm
	QUARTUS_STA := $(QUARTUS_PATH)/bin64/quartus_sta
	QUARTUS_SH  := $(QUARTUS_PATH)/bin64/quartus_sh
	QUARTUS_PGM := $(QUARTUS_PATH)/bin64/quartus_pgm
	QUARTUS_CPF := $(QUARTUS_PATH)/bin64/quartus_cpf
	QUARTUS_CDB := $(QUARTUS_PATH)/bin64/quartus_cdb
	QUARTUS_GUI := $(QUARTUS_PATH)/bin64/quartus
	QUARTUS_STP := $(QUARTUS_PATH)/bin64/quartus_stpw
	QSYS_GEN    := $(QUARTUS_PATH)/sopc_builder/bin/qsys-generate
	QSYS_EDIT   := $(QUARTUS_PATH)/sopc_builder/bin/qsys-edit
	SYSCON      := $(QUARTUS_PATH)/sopc_builder/bin/system-console
	JTAGCFG     := $(QUARTUS_PATH)/bin64/jtagconfig
else
	QUARTUS_MAP := quartus_map
	QUARTUS_FIT := quartus_fit
	QUARTUS_ASM := quartus_asm
	QUARTUS_STA := quartus_sta
	QUARTUS_SH  := quartus_sh
	QUARTUS_PGM := quartus_pgm
	QUARTUS_CPF := quartus_cpf
	QUARTUS_CDB := quartus_cdb
	QUARTUS_GUI := quartus
	QUARTUS_STP := quartus_stpw
	QSYS_GEN    := qsys-generate
	QSYS_EDIT   := qsys-edit
	SYSCON      := system-console
	JTAGCFG     := jtagconfig
endif

# -----------------------------------------------------------------
# Main target
# -----------------------------------------------------------------
#

all: smart.log $(PROJECT).asm.rpt $(PROJECT).sta.rpt

# -----------------------------------------------------------------
# Generate *.qsys to *.qip rules
# -----------------------------------------------------------------
#

ifdef QSYS_LIB_PATH
QSYS_LIB_PATH := $(QSYS_LIB_PATH),$$$$
else
QSYS_LIB_PATH := $$$$
endif

define qsys-qip-filename
	$(addprefix $(QSYS_OUT_DIR)/, \
	  $(addprefix $(subst .qsys,,$1), \
	    $(addprefix /synthesis/, $(subst .qsys,.qip,$1))))
endef

define gen-qsys-qip-rule
$1 : $2
	$(QSYS_GEN) $$^ \
		--part=$(DEVICE_PART) \
		--family=$(DEVICE_FAMILY) \
		--output-directory=$(QSYS_OUT_DIR)/$(strip $(subst .qsys,,$2)) \
		--search-path=$(QSYS_LIB_PATH) \
		--synthesis=VERILOG
endef

$(foreach i, $(QSYS_SRC), \
  $(eval \
    $(call gen-qsys-qip-rule, $(call qsys-qip-filename, $i), $i)))

QIP_QSYS_SRC := $(foreach i, $(QSYS_SRC), $(call qsys-qip-filename, $i))


# -----------------------------------------------------------------
# Project source files
# -----------------------------------------------------------------
#

PROJECT_FILES := $(PROJECT).qsf $(PROJECT).qpf

# Design files
PROJECT_SOURCES  = $(HDL_SRC)
PROJECT_SOURCES += $(QIP_SRC)
PROJECT_SOURCES += $(QIP_QSYS_SRC)

ifeq "$(words $(PROJECT_SOURCES))" "0"
$(error No design source files specified)
endif

# Other source files
PROJECT_SOURCES += $(SCRIPTS_SRC)
PROJECT_SOURCES += $(STP_FILE)

# Write file assignment to *.qsf file
define update-design-files
	$(info Run design files update...)
	$(QUARTUS_SH) -t $(CUR_FILE_PATH)/update-design-files.tcl \
		-p $(PROJECT) $1
endef

# -----------------------------------------------------------------
# I/O constraints assignments
# -----------------------------------------------------------------
#

ifdef IO_ASSIGNMENTS_TCL

define update-io-assignments
	@echo "Run I/O assignment update..."
	$(QUARTUS_SH) -t $(CUR_FILE_PATH)/update-io-assignments.tcl \
		-p $(PROJECT) -f $(IO_ASSIGNMENTS_TCL)
endef

else

$(warning I/O assignment script not specified)
define update-io-assignments
	@echo "I/O assignment script not specified..."
endef

endif

# -----------------------------------------------------------------
# Design flow
# -----------------------------------------------------------------
#

STAMP = touch

#--write_settings_files=on
MAP_ARGS += --read_settings_files=on
FIT_ARGS += --read_settings_files=on
ASM_ARGS += --read_settings_files=on

$(PROJECT).map.rpt: map.chg $(IO_ASSIGNMENTS_TCL) $(PROJECT_SOURCES)
	$(call update-design-files, $(wordlist 3, $(words $^), $^))
	$(update-io-assignments)
	$(QUARTUS_MAP) $(MAP_ARGS) $(PROJECT)
	$(STAMP) fit.chg

$(PROJECT).fit.rpt: fit.chg $(PROJECT).map.rpt
	$(QUARTUS_FIT) $(FIT_ARGS) $(PROJECT)
	$(STAMP) asm.chg
	$(STAMP) sta.chg

$(PROJECT).asm.rpt: asm.chg $(PROJECT).fit.rpt
	$(QUARTUS_ASM) $(ASM_ARGS) $(PROJECT)

$(PROJECT).sta.rpt: sta.chg $(PROJECT).fit.rpt
	$(QUARTUS_STA) $(STA_ARGS) $(PROJECT)

smart.log: $(PROJECT_FILES)
	$(QUARTUS_SH) --determine_smart_action $(PROJECT) > smart.log

map.chg:
	$(STAMP) map.chg
fit.chg:
	$(STAMP) fit.chg
sta.chg:
	$(STAMP) sta.chg
asm.chg:
	$(STAMP) asm.chg

map: smart.log $(PROJECT).map.rpt
fit: smart.log $(PROJECT).fit.rpt
asm: smart.log $(PROJECT).asm.rpt
sta: smart.log $(PROJECT).sta.rpt
smart: smart.log

# -----------------------------------------------------------------
# Project initialization
# -----------------------------------------------------------------
#

# Create Quaruts II project
$(PROJECT).qpf:
	$(QUARTUS_SH) --prepare -f $(DEVICE_FAMILY) -d $(DEVICE_PART) \
		-t $(TOP_ENTITY) $(PROJECT)

# Update assignments *.qsf
$(PROJECT).qsf: $(PROJECT).qpf $(IO_ASSIGNMENTS_TCL)
	$(update-io-assignments)


# -----------------------------------------------------------------
# Clean targets
# -----------------------------------------------------------------
#

# Remove generated files
.PHONY: clean cleanall cleanall
clean:
	rm -rf db
	rm -rf incremental_db
	rm -rf greybox_tmp/
	rm -rf .qsys_edit
	rm -rf $(QSYS_OUT_DIR)
	rm -f *.rpt *.summary *.chg smart.log
	rm -f *.htm *.eqn *.pin *.smsg
	rm -f *.json *.xml *.txt
	rm -f *.done *.jdi *.sld *.qws
	rm -f *_inst.vhd *_inst.v *_bb.v *.cmp *.bsf
	rm -f *.sopcinfo

# Remove programming files
cleanbin: clean
	rm -f $(PROJECT).sof $(PROJECT).pof $(PROJECT).jic

# Remove project files
cleanall: clean cleanbin
	rm -f $(PROJECT).qpf $(PROJECT).qsf

# -----------------------------------------------------------------
# Useful automation targets
# -----------------------------------------------------------------
#

.PHONY: ioup dfup
# Force run update-io-assignments.tcl
ioup:
	$(update-io-assignments)


# Force run update-design-files.tcl
dfup: $(PROJECT_SOURCES)
	$(update-design-files) $^


# Create project
create: $(PROJECT).qpf
	$(update-design-files)
	$(update-io-assignments)


# Open project in Quartus GUI
.PHONY: gui
gui:
	$(QUARTUS_GUI) $(PROJECT)


# Open only SignalTap GUI
.PHONY: stp
stp:
	$(QUARTUS_STP) $(PROJECT) --stp_file=$(STP_FILE)


# Run system cnsole GUI
.PHONY: syscon
syscon:
	$(SYSCON)


# Open only QSys Edit GUI
.PHONY: qsysedit
qsysedit:
	$(QSYS_EDIT)


.PHONY: qsysedit
jtaglist:
	$(JTAGCFG)


# Download *.sof to device via JTAG
# CABLE - number of JTAG cable in jtagconfig output (jtaglist target)
.PHONY: pgmsof
pgmsof:
ifdef CABLE
	$(QUARTUS_PGM) --cable=$(CABLE) --mode="JTAG" --operation="P;$(PROJECT).sof"
else
	$(QUARTUS_PGM) --mode="JTAG" --operation="P;$(PROJECT).sof"
endif


# Get current date and time
CUR_TIME := $(shell $(QUARTUS_SH) --tcl_eval \
			clock format [clock seconds] -format {%G-%m-%d_%H-%M-%S})


# Convert *.sof file to *.jic for serial flash programming
# TM - name output *.jic file with convert time
.PHONY: sof2jic
sof2jic:
ifdef TM
	$(QUARTUS_CPF) -c \
		-d $(SERIAL_FLASH_DEVICE) \
		-s $(SERIAL_FLASH_LOADER_DEVICE) \
		$(PROJECT).sof \
		$(PROJECT)_$(CUR_TIME).jic
else
	$(QUARTUS_CPF) -c \
		-d $(SERIAL_FLASH_DEVICE) \
		-s $(SERIAL_FLASH_LOADER_DEVICE) \
		$(PROJECT).sof \
		$(PROJECT).jic
endif


# Burn serial flash
.PHONY: pgmjic
pgmjic:
ifdef CABLE
	$(QUARTUS_PGM) --cable=$(CABLE) --mode="JTAG" --operation="IPV;$(JIC)"
else
	$(QUARTUS_PGM) --mode="JTAG" --operation="IPV;$(JIC)"
endif
