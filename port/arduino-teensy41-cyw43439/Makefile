#
# Makefile to assemble the BTstackTeensy Arduino library (modern recursive
# "src/" layout, see https://arduino.github.io/arduino-cli/latest/library-specification/)
# and package it as a .zip archive, mirroring what port/arduino/Makefile does
# for BlueKitchen's original (flat/legacy layout) Arduino port.
#
# Unlike port/arduino/Makefile, source files here are NOT flattened
# (--junk-paths): the modern Arduino library builder adds every subfolder of
# "src/" to the include search path and compiles everything under it
# recursively, so BTstack's own relative includes (e.g. "ble/att_db.h",
# "classic/sdp_util.h") keep working as long as the "ble"/"classic"/...
# subfolders are preserved -- so this Makefile copies files out of the
# surrounding repository preserving their relative paths under src/.
#
# Usage:
#   make            assemble the library into build/BTstackTeensy and zip it
#   make install    assemble and install into the local Arduino sketchbook
#   make clean      remove build/ and any previously built archives
#

DIR := .
BTSTACK_ROOT := $(realpath ../..)
VERSION := $(shell $(BTSTACK_ROOT)/tool/get_git_version.sh)

LIB_NAME := Teensy_Wireless_Shield_BTStack
BUILD_DIR := $(DIR)/build
STAGE_DIR := $(BUILD_DIR)/$(LIB_NAME)
ARCHIVE := $(BTSTACK_ROOT)/btstack-arduino-teensy-$(VERSION).zip

ARDUINO_LIBS := $(HOME)/Documents/Arduino/libraries

.PHONY: all release install stage clean

all: release

clean:
	rm -rf $(BUILD_DIR)
	rm -f $(BTSTACK_ROOT)/btstack-arduino-teensy-*.zip

# ---------------------------------------------------------------------------
# Assemble the library tree under $(STAGE_DIR)

stage: clean
	mkdir -p $(STAGE_DIR)/src
	mkdir -p $(STAGE_DIR)/examples

	# our own port glue (BTstackTeensy.h/.cpp, Teensy HAL, btstack_config.h)
	# and the Murata 1YN firmware patchram blob
	cp $(wildcard $(DIR)/src/*.c $(DIR)/src/*.cpp $(DIR)/src/*.h) $(STAGE_DIR)/src/
	mkdir -p $(STAGE_DIR)/src/firmware
	cp $(DIR)/src/firmware/*.c $(DIR)/src/firmware/*.h $(STAGE_DIR)/src/firmware/

	# BTstack core (src/*.c, minus LE Audio LC3 codecs and non-H4 HCI
	# transports, which this port doesn't use)
	for f in $(BTSTACK_ROOT)/src/*.c; do \
		base=$$(basename $$f); \
		case "$$base" in \
			btstack_lc3.c|btstack_lc3_google.c|btstack_lc3plus_fraunhofer.c|hci_transport_h5.c|hci_transport_em9304_spi.c) continue ;; \
		esac; \
		cp $$f $(STAGE_DIR)/src/; \
	done
	cp $(BTSTACK_ROOT)/src/*.h $(STAGE_DIR)/src/

	# BLE (GAP/GATT/SM)
	mkdir -p $(STAGE_DIR)/src/ble
	cp $(BTSTACK_ROOT)/src/ble/*.c $(BTSTACK_ROOT)/src/ble/*.h $(STAGE_DIR)/src/ble/

	## TODO - future
	## For the initial work/port of this library to teensy arduino, only a in-memory solution is 
	## provided for bluetooth link/bond database only. To make this work, remove the just copied
	## storage files related to the link/bond database.
	rm -f $(STAGE_DIR)/src/ble/le_device_db_tlv.[ch]

	# btstack.h and btstack_memory.h unconditionally reference every
	# standard GATT service's header whenever ENABLE_BLE is set, even
	# though this port only uses a plain custom GATT server/client (not
	# any of BTstack's bundled services) -- so the whole directory is
	# copied for its headers; the .c implementations come along too since
	# Arduino compiles everything under src/ recursively anyway.
	mkdir -p $(STAGE_DIR)/src/ble/gatt-service
	cp $(BTSTACK_ROOT)/src/ble/gatt-service/*.c $(BTSTACK_ROOT)/src/ble/gatt-service/*.h $(STAGE_DIR)/src/ble/gatt-service/

	# btstack.h also unconditionally includes "le-audio/le_audio.h" (used by
	# the Object Transfer Service headers above too), even though this port
	# doesn't use LE Audio/LC3.
	mkdir -p $(STAGE_DIR)/src/le-audio
	cp $(BTSTACK_ROOT)/src/le-audio/*.c $(BTSTACK_ROOT)/src/le-audio/*.h $(STAGE_DIR)/src/le-audio/

	# Classic (A2DP/AVDTP/AVRCP/HFP/SDP/RFCOMM/...)
	mkdir -p $(STAGE_DIR)/src/classic
	cp $(BTSTACK_ROOT)/src/classic/*.c $(BTSTACK_ROOT)/src/classic/*.h $(STAGE_DIR)/src/classic/

	# BCM chipset driver (Murata Type 1YN is CYW43439 silicon). Only the core
	# driver -- not btstack_chipset_bcm_download_firmware.c -- since the
	# firmware blob is supplied by our own src/firmware/ patchram file.
	mkdir -p $(STAGE_DIR)/src/chipset/bcm
	cp $(BTSTACK_ROOT)/chipset/bcm/btstack_chipset_bcm.c $(BTSTACK_ROOT)/chipset/bcm/btstack_chipset_bcm.h $(STAGE_DIR)/src/chipset/bcm/

	# embedded run loop / UART / audio / logging glue
	mkdir -p $(STAGE_DIR)/src/platform/embedded
	cp $(BTSTACK_ROOT)/platform/embedded/btstack_run_loop_embedded.c \
	   $(BTSTACK_ROOT)/platform/embedded/btstack_run_loop_embedded.h \
	   $(BTSTACK_ROOT)/platform/embedded/btstack_uart_block_embedded.c \
	   $(BTSTACK_ROOT)/platform/embedded/hci_dump_embedded_stdout.c \
	   $(BTSTACK_ROOT)/platform/embedded/hci_dump_embedded_stdout.h \
	   $(BTSTACK_ROOT)/platform/embedded/btstack_audio_embedded.c \
	   $(BTSTACK_ROOT)/platform/embedded/hal_audio.h \
	   $(BTSTACK_ROOT)/platform/embedded/hal_cpu.h \
	   $(BTSTACK_ROOT)/platform/embedded/hal_time_ms.h \
	   $(BTSTACK_ROOT)/platform/embedded/hal_uart_dma.h \
	   $(BTSTACK_ROOT)/platform/embedded/hal_tick.h \
	   $(STAGE_DIR)/src/platform/embedded/
	# our own port glue (hal_teensy_*.cpp, BTstackTeensy.cpp) bare-includes
	# "hal_cpu.h"/"hal_time_ms.h"/"hal_audio.h"/"hal_uart_dma.h"/
	# "btstack_run_loop_embedded.h"/"btstack_chipset_bcm.h"/
	# "hci_dump_embedded_stdout.h" -- since only top-level src/ is on
	# Arduino's include search path (not its subfolders), these need a
	# copy there too (verified collision-free against the rest of this
	# port's headers with a static #include scan across every bundled file).
	cp $(BTSTACK_ROOT)/platform/embedded/hal_cpu.h \
	   $(BTSTACK_ROOT)/platform/embedded/hal_time_ms.h \
	   $(BTSTACK_ROOT)/platform/embedded/hal_uart_dma.h \
	   $(BTSTACK_ROOT)/platform/embedded/hal_audio.h \
	   $(BTSTACK_ROOT)/platform/embedded/btstack_run_loop_embedded.h \
	   $(BTSTACK_ROOT)/platform/embedded/hci_dump_embedded_stdout.h \
	   $(BTSTACK_ROOT)/chipset/bcm/btstack_chipset_bcm.h \
	   $(STAGE_DIR)/src/

	# 3rd-party: bluedroid SBC encoder/decoder (A2DP/HFP audio codec),
	# micro-ecc (LE Secure Connections), rijndael (AES), yxml + md5 (OBEX/PBAP)
	#
	# The Arduino library builder only adds the top-level "src/" to the
	# include search path (unlike BTstack's own CMake/PlatformIO builds,
	# which add each 3rd-party include dir individually) -- and bluedroid's
	# encoder/decoder sources reference their own headers with bare
	# "#include <oi_...>"/"sbc_...h" names, not path-qualified ones. So
	# their include/*.h headers are copied flat into src/ itself (verified
	# collision-free against the rest of this port's headers), while the
	# srce/*.c implementation files can live anywhere recursive-compiled.
	mkdir -p $(STAGE_DIR)/src/3rd-party/bluedroid/encoder/srce
	mkdir -p $(STAGE_DIR)/src/3rd-party/bluedroid/decoder/srce
	mkdir -p $(STAGE_DIR)/src/3rd-party/micro-ecc
	mkdir -p $(STAGE_DIR)/src/3rd-party/rijndael
	mkdir -p $(STAGE_DIR)/src/3rd-party/yxml
	mkdir -p $(STAGE_DIR)/src/3rd-party/md5
	cp $(BTSTACK_ROOT)/3rd-party/bluedroid/encoder/include/*.h $(STAGE_DIR)/src/
	cp $(BTSTACK_ROOT)/3rd-party/bluedroid/encoder/srce/*.c    $(STAGE_DIR)/src/3rd-party/bluedroid/encoder/srce/
	cp $(BTSTACK_ROOT)/3rd-party/bluedroid/decoder/include/*.h $(STAGE_DIR)/src/
	cp $(BTSTACK_ROOT)/3rd-party/bluedroid/decoder/srce/*.c    $(STAGE_DIR)/src/3rd-party/bluedroid/decoder/srce/
	cp $(BTSTACK_ROOT)/3rd-party/bluedroid/decoder/srce/*.inc  $(STAGE_DIR)/src/3rd-party/bluedroid/decoder/srce/
	cp $(BTSTACK_ROOT)/3rd-party/micro-ecc/uECC.c \
	   $(BTSTACK_ROOT)/3rd-party/micro-ecc/uECC.h \
	   $(BTSTACK_ROOT)/3rd-party/micro-ecc/asm_arm.inc \
	   $(BTSTACK_ROOT)/3rd-party/micro-ecc/asm_avr.inc \
	   $(BTSTACK_ROOT)/3rd-party/micro-ecc/btstack_config_uECC.h \
	   $(STAGE_DIR)/src/3rd-party/micro-ecc/
	cp $(BTSTACK_ROOT)/3rd-party/rijndael/*.c $(BTSTACK_ROOT)/3rd-party/rijndael/*.h $(STAGE_DIR)/src/3rd-party/rijndael/
	cp $(BTSTACK_ROOT)/3rd-party/yxml/*.c     $(BTSTACK_ROOT)/3rd-party/yxml/*.h     $(STAGE_DIR)/src/3rd-party/yxml/
	cp $(BTSTACK_ROOT)/3rd-party/md5/*.c      $(BTSTACK_ROOT)/3rd-party/md5/*.h      $(STAGE_DIR)/src/3rd-party/md5/
	# src/btstack_crypto.c bare-includes "rijndael.h"/"uECC.h", yxml.c
	# bare-includes <yxml.h> with angle brackets (which never searches the
	# including file's own directory), and src/classic/pbap_client.c
	# bare-includes "md5.h" -- all need a copy on the top-level src/
	# include path for the same reason as the platform/embedded headers above.
	cp $(BTSTACK_ROOT)/3rd-party/rijndael/rijndael.h $(BTSTACK_ROOT)/3rd-party/yxml/yxml.h $(BTSTACK_ROOT)/3rd-party/md5/md5.h $(BTSTACK_ROOT)/3rd-party/micro-ecc/uECC.h $(BTSTACK_ROOT)/3rd-party/micro-ecc/btstack_config_uECC.h $(STAGE_DIR)/src/

	# library metadata + examples
	sed 's/@VERSION@/$(VERSION)/' $(DIR)/library.properties.in > $(STAGE_DIR)/library.properties
	cp $(DIR)/keywords.txt $(STAGE_DIR)/keywords.txt
	cp $(DIR)/README.md $(STAGE_DIR)/README.md
	cp -R $(DIR)/examples/. $(STAGE_DIR)/examples/

# ---------------------------------------------------------------------------

release: stage
	rm -f $(ARCHIVE)
	cd $(BUILD_DIR) && zip -rq $(ARCHIVE) $(LIB_NAME)
	cp $(ARCHIVE) $(BTSTACK_ROOT)/btstack-arduino-teensy-latest.zip
	@echo "Wrote $(ARCHIVE)"

install: stage
	rm -rf $(ARDUINO_LIBS)/$(LIB_NAME)
	mkdir -p $(ARDUINO_LIBS)
	cp -R $(STAGE_DIR) $(ARDUINO_LIBS)/$(LIB_NAME)
	@echo "Installed to $(ARDUINO_LIBS)/$(LIB_NAME)"
