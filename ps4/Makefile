# Orbis Store – PS4 homebrew Makefile
#
# Targets the OpenOrbis PS4 Toolchain v0.5.4+
# https://github.com/OpenOrbis/OpenOrbis-PS4-Toolchain
#
# Usage:
#   make          – build the PKG
#   make clean    – remove build artefacts
#   make validate – verify build artifacts without rebuilding
#
# Prerequisites:
#   - $OO_PS4_TOOLCHAIN set to the OpenOrbis toolchain root
#   - clang, llvm-ar, python3 on PATH
#
# Firmware Compatibility:
#   - SYSTEM_VER = 0 enables multi-firmware compatibility
#   - Works with GoldHEN and OpenOrbis environments
#   - Uses safe writable homebrew data path (/data/OrbisStore/)

TARGET   := OrbisStore
VERSION  := 01.00
TITLE_ID := ORBS00001
CONTENT_ID := IV0000-$(TITLE_ID)_00-ORBISSTORE000000

# CE-30008-1 FIX: Use safe homebrew data path instead of system title path
# /data/ is writable and available on all jailbroken PS4 firmware versions
# /user/app/NPXS39041/ was a system path causing permission/crash issues
HOMEBREW_DATA_PATH := /data/OrbisStore/

SRCS := \
    source/main.c      \
    source/http.c      \
    source/db.c        \
    source/ui.c      \
    source/store.c     \
    source/library.c   \
    source/redeem.c    \
    source/installer.c

# ── Toolchain Detection ──────────────────────────────────────────────────────

OO_TOOLCHAIN ?= $(OO_PS4_TOOLCHAIN)
ifeq ($(strip $(OO_TOOLCHAIN)),)
OO_TOOLCHAIN := /home/neox/Downloads/OpenOrbis/PS4Toolchain
endif
ifeq ($(wildcard $(OO_TOOLCHAIN)/include/stddef.h),)
$(error OpenOrbis toolchain not found at $(OO_TOOLCHAIN). Set OO_PS4_TOOLCHAIN to your OpenOrbis/PS4Toolchain path)
endif
SYSROOT := $(OO_TOOLCHAIN)

UNAME_S := $(shell uname -s)
ifeq ($(UNAME_S),Linux)
TOOL_BIN := $(OO_TOOLCHAIN)/bin/linux
endif
ifeq ($(UNAME_S),Darwin)
TOOL_BIN := $(OO_TOOLCHAIN)/bin/macos
endif
ifeq ($(strip $(TOOL_BIN)),)
$(error Unsupported platform $(UNAME_S). Supported: Linux, macOS)
endif

PKGTOOL_LIBS := $(CURDIR)/.pkgtool-libs/extracted/usr/lib/x86_64-linux-gnu
PKGTOOL_ENV := DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
ifneq ($(wildcard $(PKGTOOL_LIBS)/libssl.so.1.1),)
PKGTOOL_ENV := LD_LIBRARY_PATH=$(PKGTOOL_LIBS):$$LD_LIBRARY_PATH $(PKGTOOL_ENV)
endif
PKGTOOL := $(PKGTOOL_ENV) $(TOOL_BIN)/PkgTool.Core

CC      := clang
LD      := ld.lld
AR      := llvm-ar
STRIP   := llvm-strip

# Compiler flags with safe data path
CFLAGS  := \
    --target=x86_64-pc-freebsd12-elf \
    -isysroot $(SYSROOT) \
    -nostdinc \
    -I./source \
    -isystem $(SYSROOT)/include \
    -isystem $(SYSROOT)/include/c++/v1 \
    -fPIC \
    -funwind-tables \
    -fvisibility=hidden \
    -Wall \
    -Wextra \
    -D__PS4__ \
    -D_BSD_SOURCE \
    -DORBIS_STORE_DATA_PATH=\"$(HOMEBREW_DATA_PATH)\"

LDFLAGS := \
    -m elf_x86_64 \
    -pie \
    --script $(OO_TOOLCHAIN)/link.x \
    --eh-frame-hdr \
    -L$(OO_TOOLCHAIN)/lib \
    -lpthread \
    -lc \
    -lkernel \
    -lSceUserService \
    -lScePad \
    -lSceSystemService \
    -lSceVideoOut \
    -lSceBgft \
    -lSceNet \
    $(OO_TOOLCHAIN)/lib/crt1.o

# ── Build ───────────────────────────────────────────────────────────────────

OBJS    := $(SRCS:.c=.o)
ELF     := $(TARGET).elf
OELF    := $(TARGET).oelf

PKG_DIR := pkg_root
PKG_OUT := $(TARGET).pkg
GP4_OUT := package.gp4
PKG_TMP := $(CONTENT_ID).pkg

# Asset directories to auto-include (if they exist)
ASSET_DIRS := assets data fonts lang images themes resources config

.PHONY: all clean validate validate-artifacts

all: $(PKG_OUT)
	@echo "✓ Build successful: $(PKG_OUT)"

# ── COMPILATION ─────────────────────────────────────────────────────────────

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# ── LINKING ─────────────────────────────────────────────────────────────────

$(ELF): $(OBJS)
	$(LD) $(OBJS) -o $@ $(LDFLAGS)
	@echo "✓ Linked: $@"

# ── OELF GENERATION ─────────────────────────────────────────────────────────

$(OELF): $(ELF)
	$(TOOL_BIN)/create-fself -in=$< -out=$@ --eboot "eboot.bin" --paid 0x3800000000000011
	@echo "✓ Generated Open-ELF: $@"

# ── PACKAGE DIRECTORY SETUP ─────────────────────────────────────────────────

$(PKG_DIR)/sce_sys/param.sfo: $(OELF)
	@mkdir -p $(PKG_DIR)/sce_sys
	@cp $(OELF) $(PKG_DIR)/eboot.bin
	@echo "→ Creating param.sfo..."
	$(PKGTOOL) sfo_new $@
	$(PKGTOOL) sfo_setentry $@ APP_TYPE --type Integer --maxsize 4 --value 1
	$(PKGTOOL) sfo_setentry $@ APP_VER --type Utf8 --maxsize 8 --value '$(VERSION)'
	$(PKGTOOL) sfo_setentry $@ ATTRIBUTE --type Integer --maxsize 4 --value 0
	$(PKGTOOL) sfo_setentry $@ CATEGORY --type Utf8 --maxsize 4 --value 'gd'
	$(PKGTOOL) sfo_setentry $@ CONTENT_ID --type Utf8 --maxsize 48 --value '$(CONTENT_ID)'
	$(PKGTOOL) sfo_setentry $@ DOWNLOAD_DATA_SIZE --type Integer --maxsize 4 --value 0
	$(PKGTOOL) sfo_setentry $@ SYSTEM_VER --type Integer --maxsize 4 --value 0
	$(PKGTOOL) sfo_setentry $@ TITLE --type Utf8 --maxsize 128 --value '$(TARGET)'
	$(PKGTOOL) sfo_setentry $@ TITLE_ID --type Utf8 --maxsize 12 --value '$(TITLE_ID)'
	$(PKGTOOL) sfo_setentry $@ VERSION --type Utf8 --maxsize 8 --value '$(VERSION)'
	@echo "✓ Created param.sfo with multi-firmware support (SYSTEM_VER=0)"

# ── ASSET PREPARATION ───────────────────────────────────────────────────────
# Automatically copy runtime asset directories into PKG if they exist

.PHONY: copy-assets
copy-assets: $(PKG_DIR)/sce_sys/param.sfo
	@for dir in $(ASSET_DIRS); do \
		if [ -d "$$dir" ]; then \
			echo "→ Copying runtime assets: $$dir/"; \
			cp -r "$$dir" "$(PKG_DIR)/"; \
		fi; \
	done
	@echo "✓ Runtime assets prepared"

# ── GP4 GENERATION ──────────────────────────────────────────────────────────
# Generate GP4 project file with all packaged files automatically detected

$(GP4_OUT): copy-assets
	@echo "→ Generating GP4 project..."
	@echo "  Files in $(PKG_DIR):"
	@find $(PKG_DIR) -type f | sed 's/^/    /'
	$(TOOL_BIN)/create-gp4 -out $@ --content-id=$(CONTENT_ID) --files "$$(find $(PKG_DIR) -type f | tr '\n' ' ')"
	@echo "✓ Generated: $@"
	@echo "→ Normalizing paths in GP4..."
	@sed -i 's|targ_path="$(PKG_DIR)/|targ_path="|g' $@
	@echo "✓ GP4 paths normalized"

# ── BUILD VALIDATION ────────────────────────────────────────────────────────

.PHONY: validate
validate: validate-artifacts

validate-artifacts:
	@echo "─────────────────────────────────────────────────"
	@echo "BUILD VALIDATION"
	@echo "─────────────────────────────────────────────────"
	@if [ -f "$(ELF)" ]; then \
		echo "✓ $(ELF) exists"; \
		echo "  Size: $$(wc -c < $(ELF)) bytes"; \
	else \
		echo "✗ MISSING: $(ELF)"; \
		exit 1; \
	fi
	@if [ -f "$(OELF)" ]; then \
		echo "✓ $(OELF) exists"; \
		echo "  Size: $$(wc -c < $(OELF)) bytes"; \
	else \
		echo "✗ MISSING: $(OELF)"; \
		exit 1; \
	fi
	@if [ -f "$(PKG_DIR)/eboot.bin" ]; then \
		echo "✓ $(PKG_DIR)/eboot.bin exists"; \
		echo "  Size: $$(wc -c < $(PKG_DIR)/eboot.bin) bytes"; \
	else \
		echo "✗ MISSING: $(PKG_DIR)/eboot.bin"; \
		exit 1; \
	fi
	@if [ -f "$(PKG_DIR)/sce_sys/param.sfo" ]; then \
		echo "✓ $(PKG_DIR)/sce_sys/param.sfo exists"; \
		echo "  Size: $$(wc -c < $(PKG_DIR)/sce_sys/param.sfo) bytes"; \
	else \
		echo "✗ MISSING: $(PKG_DIR)/sce_sys/param.sfo"; \
		exit 1; \
	fi
	@asset_count=$$(find $(PKG_DIR) -type f | wc -l); \
	if [ $$asset_count -ge 2 ]; then \
		echo "✓ Assets packaged: $$asset_count files"; \
	else \
		echo "⚠ Only core files present (eboot.bin, param.sfo)"; \
	fi
	@echo "─────────────────────────────────────────────────"

# ── PACKAGE GENERATION ──────────────────────────────────────────────────────

$(PKG_OUT): $(GP4_OUT) validate-artifacts
	@echo "→ Building PKG..."
	@if ! $(PKGTOOL) pkg_build $< .; then \
		echo "✗ FAILED: pkg_build error. Aborting (no placeholder)."; \
		exit 1; \
	fi
	@if [ ! -f "$(PKG_TMP)" ]; then \
		echo "✗ FAILED: pkg_build did not create $(PKG_TMP). Aborting."; \
		exit 1; \
	fi
	@if [ "$(PKG_TMP)" != "$(PKG_OUT)" ]; then \
		mv -f "$(PKG_TMP)" "$(PKG_OUT)"; \
		echo "→ Renamed $(PKG_TMP) → $(PKG_OUT)"; \
	fi
	@pkg_size=$$(wc -c < $(PKG_OUT)); \
	if [ $$pkg_size -gt 1000 ]; then \
		echo "✓ PKG created successfully"; \
		echo "  File: $(PKG_OUT)"; \
		echo "  Size: $$pkg_size bytes ($$(( $$pkg_size / 1024 / 1024 ))MB)"; \
	else \
		echo "✗ FAILED: PKG file too small ($$pkg_size bytes). Package is broken."; \
		rm -f $(PKG_OUT); \
		exit 1; \
	fi

clean:
	@echo "Cleaning build artifacts..."
	rm -f $(OBJS) $(ELF) $(OELF) $(PKG_OUT) $(GP4_OUT) $(PKG_TMP) eboot.bin
	rm -rf $(PKG_DIR)
	@echo "✓ Clean complete"
