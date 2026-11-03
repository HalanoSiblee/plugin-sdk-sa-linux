
# ============================================================================
# linux/sa/Makefile — single-file SA build for plugin-sdk-sa (Dryxio fork)
# Target: GTA San Andreas, Windows x86 (PE32) via MinGW-w64 GCC.
#
#   make -C linux/sa                # lib + all examples
#   make -C linux/sa lib            # library only
#   make -C linux/sa plugin PLUGIN=MinimalTest
#   make -C linux/sa plugins        # all examples
#   make -C linux/sa clean
#   make -C linux/sa CONFIG=debug   # debug build
#
# Prerequisites: sudo apt install g++-mingw-w64-i686-posix ccache
# ============================================================================

# SDK root = two levels up from this Makefile (linux/sa -> repo root)
SDK_DIR := $(abspath $(dir $(lastword $(MAKEFILE_LIST)))/../..)

# ---------------- toolchain ----------------
PREFIX ?= i686-w64-mingw32-
CC  := $(PREFIX)gcc
CXX := $(PREFIX)g++
AR  := $(PREFIX)ar

ifneq ($(shell command -v ccache 2>/dev/null),)
  CC  := ccache $(CC)
  CXX := ccache $(CXX)
endif

# ---------------- config ----------------
CONFIG ?= release
ifeq ($(CONFIG),debug)
  OPT := -g -O0 -DDEBUG
  LIBSUF := _d
  BUILDSUB := debug
else ifeq ($(CONFIG),release)
  OPT := -O2 -DNDEBUG
  LIBSUF :=
  BUILDSUB := release
else
  $(error CONFIG must be 'release' or 'debug' (got '$(CONFIG)'))
endif

# ---------------- output paths ----------------
BUILD := $(SDK_DIR)/build/sa/$(BUILDSUB)
OBJ   := $(BUILD)/obj
LIB   := $(BUILD)/lib
BIN   := $(BUILD)/bin

# ---------------- flags ----------------
# NOTE: do NOT add -Wl,--gc-sections or -s. They strip the SDK's DllMain
# and static initializers, producing a .asi that loads but does nothing.
WARN        := -Wno-invalid-offsetof -Wno-microsoft-include \
               -Wno-builtin-macro-redefined
CXX_COMMON  := -std=c++23 $(OPT) -fms-extensions -fpermissive -fcommon \
               $(WARN) -MMD -MP
C_COMMON    := -std=gnu11 $(OPT) -fcommon -MMD -MP
LIBS_BASE   := -shared -static -static-libgcc -static-libstdc++ -Wl,--as-needed

GAME_DEF := -DGTASA -DRW -DPLUGIN_SGV_10US

INCLUDES := \
    -I$(SDK_DIR)/plugin_sa \
    -I$(SDK_DIR)/plugin_sa/game_sa \
    -I$(SDK_DIR)/plugin_sa/game_sa/enums \
    -I$(SDK_DIR)/plugin_sa/game_sa/rw \
    -I$(SDK_DIR)/shared \
    -I$(SDK_DIR)/shared/game \
    -I$(SDK_DIR)/shared/extender \
    -I$(SDK_DIR)/shared/extensions \
    -I$(SDK_DIR)/shared/comp \
    -I$(SDK_DIR)/hooking \
    -I$(SDK_DIR)/safetyhook \
    -I$(SDK_DIR)/injector \
    -I$(SDK_DIR)/modutils \
    -I$(SDK_DIR)/stb

# ---------------- sources ----------------
SDK_CPPS := \
    $(wildcard $(SDK_DIR)/plugin_sa/game_sa/*.cpp) \
    $(wildcard $(SDK_DIR)/shared/*.cpp) \
    $(wildcard $(SDK_DIR)/shared/comp/*.cpp) \
    $(wildcard $(SDK_DIR)/shared/comp/plugins/*.cpp) \
    $(wildcard $(SDK_DIR)/shared/extensions/*.cpp) \
    $(wildcard $(SDK_DIR)/shared/game/*.cpp) \
    $(wildcard $(SDK_DIR)/hooking/*.cpp) \
    $(wildcard $(SDK_DIR)/safetyhook/*.cpp)

SDK_CFILES := $(wildcard $(SDK_DIR)/safetyhook/*.c)

SDK_OBJS := \
    $(patsubst $(SDK_DIR)/%.cpp,$(OBJ)/%.o,$(SDK_CPPS)) \
    $(patsubst $(SDK_DIR)/%.c,  $(OBJ)/%.o,$(SDK_CFILES))

SDK_LIB := $(LIB)/libPlugin$(LIBSUF).a

# ---------------- targets ----------------
.PHONY: all lib plugin plugins clean

all: lib plugins

lib: $(SDK_LIB)

$(SDK_LIB): $(SDK_OBJS)
	@mkdir -p $(dir $@)
	@echo "  AR      $@"
	@$(AR) rcs $@ $^

$(OBJ)/%.o: $(SDK_DIR)/%.cpp
	@mkdir -p $(dir $@)
	@echo "  CXX     $<"
	@$(CXX) $(CXX_COMMON) $(GAME_DEF) $(INCLUDES) -c $< -o $@

$(OBJ)/%.o: $(SDK_DIR)/%.c
	@mkdir -p $(dir $@)
	@echo "  CC      $<"
	@$(CC) $(C_COMMON) $(GAME_DEF) $(INCLUDES) -c $< -o $@

# ---------------- plugins ----------------
PLUGIN      ?=
PLUGIN_DIR  := $(SDK_DIR)/examples/$(PLUGIN)
PLUGIN_SRCS := $(wildcard $(PLUGIN_DIR)/*.cpp)
PLUGIN_OBJS := $(patsubst $(PLUGIN_DIR)/%.cpp,$(OBJ)/plugins/$(PLUGIN)/%.o,$(PLUGIN_SRCS))
PLUGIN_ASI  := $(BIN)/$(PLUGIN).SA.asi
ifeq ($(PLUGIN),saterminal)
    PLUGIN_INC  := -I$(SDK_DIR)/shared/dxsdk
    PLUGIN_LIBS := $(SDK_DIR)/shared/dxsdk/d3dx9.lib
endif
plugin: $(PLUGIN_ASI)

$(PLUGIN_ASI): $(PLUGIN_OBJS) $(SDK_LIB)
	@mkdir -p $(dir $@)
	@echo "  LD      $@"
	@$(CXX) $(PLUGIN_OBJS) -o $@ $(LIBS_BASE) -L$(LIB) -lPlugin$(LIBSUF)
	@echo "  ----    inspecting output:"
	@file $@
	@i686-w64-mingw32-objdump -p $@ | grep -i 'AddressOfEntryPoint' || true
	@ls -lh $@ | awk '{print "  ----    size: "$$5}'

$(OBJ)/plugins/$(PLUGIN)/%.o: $(PLUGIN_DIR)/%.cpp $(SDK_LIB)
	@mkdir -p $(dir $@)
	@echo "  CXX     $<"
	@$(CXX) $(CXX_COMMON) $(GAME_DEF) $(INCLUDES) -I$(PLUGIN_DIR) -c $< -o $@

EXAMPLE_NAMES := $(sort $(notdir $(patsubst %/,%,$(dir $(wildcard $(SDK_DIR)/examples/*/*.cpp)))))

plugins:
	@test -n "$(EXAMPLE_NAMES)" || { echo "No examples found"; exit 1; }
	@for n in $(EXAMPLE_NAMES); do \
	    $(MAKE) --no-print-directory plugin PLUGIN=$$n || exit 1; \
	done

clean:
	rm -rf $(BUILD)

-include $(SDK_OBJS:.o=.d) $(PLUGIN_OBJS:.o=.d)