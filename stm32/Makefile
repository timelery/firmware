# (c) Copyright 2018 by Coinkite Inc. This file is covered by license found in COPYING-CC.
#
# Build micropython for stm32 (an ARM processor). Also handles signing of resulting firmware images.
#
MPY_TOP = ../external/micropython
PORT_TOP = $(MPY_TOP)/ports/stm32
MPY_CROSS = $(MPY_TOP)/mpy-cross/mpy-cross
PYTHON_MAKE_DFU = $(MPY_TOP)/tools/dfu.py
PYTHON_DO_DFU = $(MPY_TOP)/tools/pydfu.py

# aka ../cli/signit.py
SIGNIT = signit

MAKE_ARGS = BOARD=COLDCARD -j 4

all: COLDCARD/file_time.c
	cd $(PORT_TOP) && $(MAKE) $(MAKE_ARGS)

clean-mpy:
	rm -rf build

clean: clean-mpy
	cd $(PORT_TOP) && $(MAKE) $(MAKE_ARGS) clean

# These values used to make .DFU files. Flash memory locations.
FIRMWARE_BASE   = 0x08008000
BOOTLOADER_BASE = 0x08000000
FILESYSTEM_BASE = 0x080e0000

# Our version for this release.
VERSION_STRING = 3.2.2

#
# Sign and merge various parts
#
firmware-signed.bin: l-port/build-COLDCARD/firmware?.bin
	$(SIGNIT) sign $(VERSION_STRING)
firmware-signed.dfu: firmware-signed.bin Makefile
	$(PYTHON_MAKE_DFU) -b $(FIRMWARE_BASE):$< $@

# make the DFU file which is shared for upgrades
dfu: firmware-signed.dfu

# Build a binary, signed w/ production key
# - always rebuild binary
.PHONY: dev.dfu
dev.dfu: l-port/build-COLDCARD/firmware?.bin
	cd $(PORT_TOP) && $(MAKE) $(MAKE_ARGS)
	$(SIGNIT) sign $(VERSION_STRING) -o dev.bin
	$(PYTHON_MAKE_DFU) -b $(FIRMWARE_BASE):dev.bin dev.dfu

# This is fast for Coinkite devs, but no DFU support in the wild.
up: dev.dfu
	$(PYTHON_DO_DFU) -u dev.dfu

# Slow, but works with unmod-ed board: use USB protocol to upgrade (2 minutes)
dev: dev.dfu
	ckcc upgrade dev.dfu

COLDCARD/file_time.c: Makefile make_filetime.py
	./make_filetime.py COLDCARD/file_time.c $(VERSION_STRING)

# Make a factory release: using key #1
production.bin: firmware-signed.bin Makefile
	$(SIGNIT) sign $(VERSION_STRING) -r firmware-signed.bin -k 1 -o production.bin

# This is release of the bootloader that will be built into the release firmware.
BOOTLOADER_VERSION = 2.0.1

# This target just combines latest version of production firmware with bootrom into a DFU
# file, stored in ../releases with appropriately dated file name.
.PHONY: release
release: NEW_VERSION = $(shell $(SIGNIT) version production.bin)
release: RELEASE_FNAME = ../releases/$(NEW_VERSION)-coldcard.dfu
release: production.bin
	test ! -f $(RELEASE_FNAME)
	$(PYTHON_MAKE_DFU) -b $(FIRMWARE_BASE):production.bin \
		-b $(BOOTLOADER_BASE):bootloader/releases/$(BOOTLOADER_VERSION)/bootloader.bin \
		$(RELEASE_FNAME)
	@echo
	@echo 'Made release: ' $(RELEASE_FNAME)
	@echo

# Use DFU to install the latest production version you have on hand
dfu-latest: 
	$(PYTHON_DO_DFU) -u `ls -t1 ../releases/*.dfu | head -1`

# Use slow USB upload and reboot method.
latest:
	ckcc upgrade `ls -t1 ../releases/*.dfu | head -1`

.PHONY: code-committed
code-committed:
	@echo ""
	@echo "Are all changes commited already?"
	git diff --stat --exit-code .
	@echo '... yes'

.PHONY: sign-release
sign-release: PUBLIC_VERSION = $(shell $(SIGNIT) version production.bin)
sign-release:
	(cd ../releases; shasum -a 256 *.dfu *.md | sort -rk 2 | \
		gpg --clearsign -u A3A31BAD5A2A5B10 --digest-algo SHA256 --output signatures.txt --yes - )
	git commit -m "Signed for release: "$(PUBLIC_VERSION) ../releases/signatures.txt

# Tag source code associate with built release version.
# - do "make release" before this step!
# - also edit/commit ChangeLog text too
# - this target will "git add" the new binary in ../release/*dfu 
# - update & sign signatures file
# - and tag everything
tag-source: PUBLIC_VERSION = $(shell $(SIGNIT) version production.bin)
tag-source: sign-release code-committed
	git add ../releases/$(PUBLIC_VERSION)-coldcard.dfu
	git commit -am "New release: "$(PUBLIC_VERSION)
	git push
	echo "Taging version: " $(PUBLIC_VERSION)
	git tag -a $(PUBLIC_VERSION) -m "Release "$(PUBLIC_VERSION)
	git push --tags

# DFU file of boot and main code
# - bootloader is last so it can fail if already installed (maybe)
#
mostly.dfu: firmware-signed.bin bootloader/bootloader.bin Makefile
	$(PYTHON_MAKE_DFU) \
			-b $(FIRMWARE_BASE):firmware-signed.bin \
			-b $(BOOTLOADER_BASE):bootloader/bootloader.bin $@

# send everything
m-dfu: mostly.dfu
	$(PYTHON_DO_DFU) -u mostly.dfu

# Clear the internal filesystem (for dev-mistakes recovery)
# - unused?
.PHONY: wipe-fs
wipe-fs: 
	dd if=/dev/urandom of=tmp.bin bs=512 count=1
	$(PYTHON_MAKE_DFU) -b $(FILESYSTEM_BASE):tmp.bin tmp.dfu
	$(PYTHON_DO_DFU) -u tmp.dfu
	rm tmp.bin tmp.dfu

# unused
stlink:
	cd $(PORT_TOP) && $(MAKE) $(MAKE_ARGS) deploy-stlink

# useless, will be ignored by bootloader
unsigned-dfu:
	cd $(PORT_TOP) && $(MAKE) $(MAKE_ARGS) deploy

# see COLDCARD/mpconfigboard.mk
tags: 
	cd $(PORT_TOP) && $(MAKE) $(MAKE_ARGS) tags
checksum: 
	cd $(PORT_TOP) && $(MAKE) $(MAKE_ARGS) checksum
files:
	cd $(PORT_TOP) && $(MAKE) $(MAKE_ARGS) files

# OLD dev junk?
# compile and freeze python code
PY_FILES = $(shell find ../shared -name \*.py)
ALL_MPY_FILES = $(addprefix build/, $(PY_FILES:../shared/%.py=%.mpy))
MPY_FILES = $(filter-out build/obsolete/%, $(ALL_MPY_FILES))

build/%.mpy: ../shared/%.py Makefile
	mkdir -p $(dir $@)
	$(MPY_CROSS) -o $@ -s $*.py $<


# In another window: 
#
#		openocd -f l-port/boards/openocd_stm32l4.cfg
#
# Can do:
# - "load" which writes the flash (medium speed, lots of output on st-util)
# - "cont" starts/continues system
# - "br main" sets breakpoints
# - "mon reset" to reset micro
# - and so on
#
debug:
	arm-none-eabi-gdb $(PORT_TOP)/build-COLDCARD/firmware.elf -x gogo.gdb

# detailed listing, very handy
OBJDUMP = arm-none-eabi-objdump
firmware.lss: $(PORT_TOP)/build-COLDCARD/firmware.elf
	$(OBJDUMP) -h -S $< > $@

# Dump sizes of all frozen py files; requires recent build.
.PHONY: sizes
sizes:
	wc -c l-port/build-COLDCARD/frozen_mpy/*.mpy | sort -n

# Measure flash impact of a single file. Great for before/after.
# 	make F=foo.py size
# where: foo.py is anything in ../shared
size:
	$(MPY_CROSS) -o tmp.mpy -s $F ../shared/$F
	wc -c tmp.mpy

# one time setup, after repo checkout
setup:
	cd $(MPY_TOP) ; git submodule update --init lib/stm32lib
	cd $(MPY_TOP)/mpy-cross ; make
	-ln -s $(PORT_TOP) l-port
	-ln -s $(MPY_TOP) l-mpy
	-cd $(PORT_TOP)/boards ; ln -s ../../../../../stm32/COLDCARD COLDCARD

# EOF
