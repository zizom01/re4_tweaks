# RE4 Tweaks - Architecture Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Directory Structure](#directory-structure)
3. [Core Components](#core-components)
4. [Architecture & Data Flow](#architecture--data-flow)
5. [Build System](#build-system)
6. [Key Technologies](#key-technologies)

---

## Project Overview

**RE4 Tweaks** is a comprehensive modification framework for Resident Evil 4's Steam "UHD" port. It operates as a **DLL injection mod** that provides 50+ tweaks, fixes, and enhancements without requiring modifications to the game's executable.

### What It Does

- **Display Fixes**: FOV adjustment, ultrawide aspect ratio support, DPI scaling, V-Sync control
- **Input Improvements**: Raw mouse input, mouse turning modes, controller sensitivity tweaks, custom key bindings
- **Audio Enhancements**: Separate volume sliders for music/SFX/cutscenes
- **Rendering Fixes**: Film grain removal, blur effects, blurry image fixes, Vulkan renderer support via DXVK
- **Gameplay Fixes**: 60 FPS physics corrections, QTE improvements, frame rate optimizations
- **Game Expansion**: Trainer features, debug menu access, modding API extensions

### How It Works

The mod operates by:
1. **DLL Injection**: Masquerading as `dinput8.dll` (DirectInput wrapper)
2. **Memory Scanning**: Finding game functions and data structures via pattern matching
3. **Function Hooking**: Intercepting game functions to modify behavior
4. **Memory Patching**: Directly modifying game code and data in memory
5. **Runtime Integration**: Providing an overlay UI and real-time configuration

---

## Directory Structure

```
re4_tweaks/
├── dllmain/              # Core mod implementation (45+ source files)
│   ├── dllmain.cpp       # DLL entry point & initialization orchestrator
│   ├── SDK/              # Reverse-engineered game API headers (from GC version)
│   │   ├── player.h      # Player character structures
│   │   ├── enemy.h       # Enemy entity structures
│   │   ├── item.h        # Item system structures
│   │   ├── camera.h      # Camera system
│   │   ├── pad.h         # Controller input structures
│   │   └── ...
│   ├── Settings.h/cpp    # Configuration management (INI parsing/saving)
│   ├── Game.h/cpp        # Game state, version detection, pointer management
│   ├── Patches.h         # Declaration of all init functions
│   ├── [Feature modules] # Individual feature implementations
│   │   ├── AspectRatioTweaks.cpp
│   │   ├── AudioTweaks.cpp
│   │   ├── CameraTweaks.cpp
│   │   ├── ControllerTweaks.cpp
│   │   ├── D3D9hook.cpp
│   │   ├── DisplayTweaks.cpp
│   │   ├── FrameRateFixes.cpp
│   │   ├── Gameplay.cpp
│   │   ├── input.cpp
│   │   ├── KeyboardMouseTweaks.cpp
│   │   ├── MouseTurning.cpp
│   │   ├── QTEFixes.cpp
│   │   ├── Trainer.cpp
│   │   └── ...
│   └── UI components/
│       ├── cfgMenu.cpp   # In-game configuration menu (F1)
│       ├── ToolMenu.cpp  # Debug menu integration
│       └── UI_*.cpp      # Various UI overlays
├── Wrappers/             # DLL injection system
│   ├── dinput8/          # Primary wrapper (DirectInput)
│   ├── xinput1_3/        # Alternative wrapper (XInput)
│   ├── winmm/            # Alternative wrapper (Windows Multimedia)
│   └── wrappers.cpp      # Wrapper initialization logic
├── dxvk/                 # DXVK Vulkan renderer (git submodule)
│   ├── src/d3d9/         # Direct3D 9 to Vulkan translation layer
│   ├── src/dxvk/         # Core Vulkan rendering implementation
│   └── include/          # Vulkan and SPIR-V headers
├── external/             # Third-party libraries (git submodules)
│   ├── imgui/            # Immediate mode GUI framework
│   ├── injector/         # Memory hooking and code injection library
│   ├── spdlog/           # Fast C++ logging library
│   ├── simpleini/        # INI file parser
│   ├── json/             # JSON parsing library
│   ├── freetype/         # Font rendering library
│   ├── inih/             # Lightweight INI parser
│   └── ...
├── includes/             # Shared header files
├── settings/             # Configuration file templates
├── dist/                 # Distribution files (built DLLs, INI templates)
└── re4_tweaks.sln        # Visual Studio solution file
```

---

## Core Components

### 1. DLL Entry Point (`dllmain/dllmain.cpp`)

The main entry point that orchestrates the entire initialization sequence.

```cpp
// Simplified flow
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        Init_Wrappers();          // Setup DLL wrapping for dinput8/xinput/winmm
        ExceptionHandler();       // Setup crash dump handler
        Init_Main();              // Core initialization
        break;
    }
}

void Init_Main()
{
    // 1. Game detection and version identification
    Game::init();
    
    // 2. Input system initialization
    Input::init();
    
    // 3. Load configuration from INI file
    ReadSettings();
    
    // 4. Initialize all feature modules
    re4t::init::DisplayTweaks();
    re4t::init::AspectRatioTweaks();
    re4t::init::AudioTweaks();
    re4t::init::KeyboardMouseTweaks();
    re4t::init::ControllerTweaks();
    re4t::init::FrameRateFixes();
    re4t::init::QTEfixes();
    re4t::init::Gameplay();
    // ... and many more
    
    // 5. Setup UI and trainer
    re4t::init::ToolMenu();
    re4t::init::Trainer();
}
```

**Key responsibilities:**
- Initialize DLL wrapper system
- Set up exception handling for crash dumps
- Detect game version and locate memory addresses
- Load user configuration
- Initialize all feature modules
- Set up UI overlay system

### 2. Game Interface (`dllmain/Game.cpp` & `dllmain/SDK/`)

Manages interaction with the game's memory and data structures.

**Game.cpp** provides:
- Game version detection (detecting different Steam releases)
- Pattern-based memory scanning to locate functions and data
- Pointer management for game structures
- Game state tracking

**SDK/** contains reverse-engineered headers:
- Reconstructed from GameCube debug symbols
- Provides typed access to game's internal structures
- Enables type-safe modification of game state

Example structures:
```cpp
// player.h
struct cPlayer {
    Vec pos;           // Player position
    Vec rot;           // Player rotation
    int health;        // Current health
    int maxHealth;     // Maximum health
    // ... many more fields
};

// enemy.h
struct cEnemy {
    EnemyID id;
    Vec pos;
    int health;
    float scale;
    // ... enemy-specific data
};
```

### 3. Configuration System (`dllmain/Settings.cpp`)

Manages all user-configurable settings through INI files.

**Features:**
- Reads `dinput8.ini` on startup
- Provides default values for all settings
- Saves changes made via in-game menu
- Supports per-setting enable/disable flags
- Validates input ranges

**Structure:**
```ini
[DISPLAY]
FOV = 1.15
FixAspectRatio = 1
UltraWideAspectSupport = 1

[MOUSE]
CameraImprovements = 1
UseRawMouseInput = 1
DetachCameraFromAim = 0

[FRAMERATE]
FixFallingItemsSpeed = 1
FixCompartmentsSpeed = 1
FixTurningSpeed = 1
```

### 4. Memory Hooking System (`external/injector/`)

The injector library provides low-level memory manipulation capabilities:

**Pattern Scanning:**
```cpp
// Find function by byte pattern
auto pattern = hook::pattern("55 8B EC 83 EC ? 56 8B 75 08");
if (pattern.count_hint(1).empty())
    return; // Pattern not found

// Hook the function
injector::MakeJMP(pattern.get_first(), &MyHookedFunction);
```

**Memory Writing:**
```cpp
// Patch constant values
injector::WriteMemory<float>(0x12345678, 1.5f, true);

// NOP out instructions
injector::MakeNOP(0x12345678, 5); // Replace 5 bytes with NOPs
```

**Function Hooking:**
```cpp
// Trampoline hook - calls original before/after custom code
static void* originalFunc = nullptr;
originalFunc = injector::MakeCALL(address, &MyHook).get();
```

### 5. Input System (`dllmain/input.cpp`)

Handles all input processing and custom key bindings.

**Capabilities:**
- DirectInput and XInput support
- Raw mouse input integration
- Custom key binding system
- Configurable controller dead zones
- QTE key remapping with proper on-screen prompts

**Key Features:**
- Intercepts input before game processes it
- Translates keyboard keys to game-expected values
- Provides mouse turning mode (camera vs. character control)
- Enables keyboard-only features (e.g., inventory flipping)

### 6. Rendering Integration (`dllmain/D3D9hook.cpp`)

Hooks into DirectX 9 rendering pipeline for:
- **ImGui Integration**: Renders overlay UI (config menu, trainer, debug info)
- **DXVK Support**: Can redirect D3D9 calls to Vulkan via DXVK
- **Visual Fixes**: Applies shader and rendering fixes
- **Performance Monitoring**: FPS counter, frame timing

**Hook Points:**
```cpp
// Hooked functions
IDirect3DDevice9::EndScene()   // Render ImGui overlays
IDirect3DDevice9::Reset()      // Handle device loss/restoration
IDirect3DDevice9::Present()    // Frame presentation
```

### 7. Feature Modules

Each feature is implemented as an independent module with an `init()` function:

#### Display Tweaks (`DisplayTweaks.cpp`)
- FOV modification
- V-Sync control
- DPI scaling fixes
- Refresh rate handling

#### Aspect Ratio Tweaks (`AspectRatioTweaks.cpp`)
- Ultrawide (21:9, 32:9) support
- 16:10 black bar removal
- HUD repositioning
- Projection matrix fixes

#### Frame Rate Fixes (`FrameRateFixes.cpp`)
- Physics timing corrections for 60 FPS
- Animation speed fixes
- QTE timing adjustments
- Falling items speed correction

#### Audio Tweaks (`AudioTweaks.cpp`)
- Volume slider separation
- Audio restoration features
- Sound effect fixes

#### Trainer (`Trainer.cpp`)
- Health/ammo modification
- Item spawning
- Position manipulation
- Game state editing

### 8. UI System

#### Configuration Menu (`cfgMenu.cpp`)
- In-game overlay (F1 key)
- Live setting adjustment
- Saves changes to INI
- Organized by category
- Uses ImGui framework

#### Debug Menu (`ToolMenu.cpp`)
- Enables hidden in-game debug menu
- Adds custom menu entries
- Save game functionality
- DOF/Blur controls

---

## Architecture & Data Flow

### Initialization Sequence

```
Windows loads bio4.exe
    ↓
Windows imports dinput8.dll (our mod DLL)
    ↓
DllMain(DLL_PROCESS_ATTACH) called
    ↓
┌─────────────────────────────────────┐
│ 1. Init_Wrappers()                  │
│    - Setup DirectInput/XInput proxy │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 2. ExceptionHandler()               │
│    - Setup crash dump generation    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 3. Init_Main()                      │
└─────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────────────────┐
│ 3a. Game::init()                                     │
│     - Detect game version (pattern matching)         │
│     - Find game functions (pattern scanning)         │
│     - Locate global data pointers                    │
└──────────────────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────────────────┐
│ 3b. Input::init()                                    │
│     - Hook input functions                           │
│     - Initialize key mappings                        │
└──────────────────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────────────────┐
│ 3c. ReadSettings()                                   │
│     - Parse dinput8.ini                              │
│     - Load user preferences                          │
└──────────────────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────────────────┐
│ 3d. Initialize Feature Modules                       │
│     - re4t::init::DisplayTweaks()                    │
│     - re4t::init::AspectRatioTweaks()                │
│     - re4t::init::AudioTweaks()                      │
│     - re4t::init::KeyboardMouseTweaks()              │
│     - re4t::init::ControllerTweaks()                 │
│     - re4t::init::FrameRateFixes()                   │
│     - ... (40+ more modules)                         │
└──────────────────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────────────────┐
│ 3e. Setup Hooks                                      │
│     - WndProcHook() - Window messages                │
│     - D3D9 EndScene() - Rendering                    │
│     - Input hooks - Custom input handling            │
└──────────────────────────────────────────────────────┘
    ↓
Game starts normally with all mods active
```

### Runtime Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                       GAME LOOP                             │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
┌───────────────┐  ┌────────────────┐  ┌──────────────────┐
│ Input Events  │  │ Game Logic     │  │ Rendering        │
└───────────────┘  └────────────────┘  └──────────────────┘
        │                   │                   │
        ↓                   ↓                   ↓
┌───────────────┐  ┌────────────────┐  ┌──────────────────┐
│ Input Hooks   │  │ Memory Patches │  │ EndScene Hook    │
│ - Raw Mouse   │  │ - 60FPS Fixes  │  │ - ImGui UI       │
│ - Custom Keys │  │ - QTE Timing   │  │ - Trainer        │
│ - Controller  │  │ - FOV Changes  │  │ - Config Menu    │
└───────────────┘  └────────────────┘  └──────────────────┘
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
                            ↓
                ┌─────────────────────┐
                │  Modified Game      │
                │  Behavior            │
                └─────────────────────┘
```

### Memory Patching Process

```
┌─────────────────────────────────────────────────────────────┐
│ Feature Module Init (e.g., re4t::init::FrameRateFixes())   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
        ┌───────────────────────────────────────┐
        │ 1. Find Target Code via Pattern      │
        │    pattern("55 8B EC 83 EC ? 56")    │
        └───────────────────────────────────────┘
                            │
                            ↓
        ┌───────────────────────────────────────┐
        │ 2. Verify Pattern Found               │
        │    if (pattern.empty()) return;       │
        └───────────────────────────────────────┘
                            │
                            ↓
        ┌───────────────────────────────────────┐
        │ 3. Apply Modification                 │
        │    Option A: Hook function            │
        │    Option B: Patch bytes              │
        │    Option C: Write constant           │
        └───────────────────────────────────────┘
                            │
                            ↓
        ┌───────────────────────────────────────┐
        │ Game now executes modified code       │
        └───────────────────────────────────────┘
```

---

## Build System

### Build Configuration

**Platform**: Windows (x86/32-bit only)
**Toolchain**: Visual Studio 2022, MSVC
**Language**: C++17
**Output**: Win32 DLL

### Solution Structure

```
re4_tweaks.sln
├── dinput8.vcxproj          # Main project (builds dinput8.dll)
├── xinput1_3.vcxproj        # Alternative wrapper (builds xinput1_3.dll)
└── winmm.vcxproj            # Alternative wrapper (builds winmm.dll)
```

### Build Configurations

- **Debug**: Debug symbols, no optimizations, verbose logging
- **Release**: Full optimizations, minimal logging

### Dependencies

**System Libraries:**
- `d3d9.lib` - Direct3D 9
- `dinput8.lib` - DirectInput 8
- `xinput.lib` - XInput
- `winmm.lib` - Windows Multimedia
- `wininet.lib` - Internet functions (auto-updater)
- `urlmon.lib` - URL Moniker (downloads)

**Third-Party Libraries (statically linked):**
- `freetype.lib` - Font rendering
- `libdisplay-info_x86.lib` - Display information

**Header-Only Libraries:**
- ImGui (GUI framework)
- spdlog (logging)
- simpleini (INI parsing)
- nlohmann/json (JSON)
- injector (memory hooking)

### Build Process

1. **Precompile Headers**: `pch.h` / `pch.cpp`
2. **Compile Source**: All `.cpp` files in `dllmain/`
3. **Compile Wrappers**: Wrapper-specific code
4. **Link**: Combine objects with libraries
5. **Output**: `dinput8.dll` (and optional `xinput1_3.dll`, `winmm.dll`)
6. **Post-Build**: Copy to `dist/` folder with INI templates

### Compiler Flags (Release)

- `/O2` - Maximum speed optimization
- `/GL` - Whole program optimization
- `/GS-` - Disable security checks (game mod context)
- `/std:c++17` - C++17 standard
- `/permissive-` - Strict conformance
- `/MP` - Multi-processor compilation

---

## Key Technologies

### 1. DLL Injection / Proxy DLL

**Technique**: The mod uses a "proxy DLL" approach:
- Game tries to load `dinput8.dll` (DirectInput library)
- Our mod DLL is named `dinput8.dll` and loads first
- We forward actual DirectInput calls to the real system DLL
- This gives us code execution in the game's process

**Alternatives**: The same technique works with `xinput1_3.dll` and `winmm.dll` for compatibility with other mods.

### 2. Memory Pattern Scanning

**Purpose**: Find functions and data without hardcoded addresses (resilient to game updates)

**Technique**:
```cpp
// Define byte pattern (? = wildcard)
auto pattern = hook::pattern("55 8B EC 83 EC ? 56 8B 75 08");

// Scan game memory
uintptr_t address = pattern.get_first<uintptr_t>();

// Use the address
auto func = (FuncType)address;
```

### 3. Function Hooking

**Detours / Trampolines**:
```cpp
// Original function pointer
typedef void (*OriginalFunc_t)(int param);
OriginalFunc_t OriginalFunc = nullptr;

// Hook function
void HookedFunc(int param) {
    // Pre-processing
    ModifyParam(&param);
    
    // Call original
    OriginalFunc(param);
    
    // Post-processing
    LogResult();
}

// Install hook
OriginalFunc = injector::MakeCALL(address, HookedFunc).get();
```

### 4. ImGui Integration

**Immediate Mode GUI**:
- Renders directly on top of game
- Uses EndScene hook
- Provides configuration menus, trainer UI, overlays

**Rendering Flow**:
```cpp
IDirect3DDevice9::EndScene() [HOOKED]
    ↓
ImGui::NewFrame()
    ↓
Render UI components (menus, windows, overlays)
    ↓
ImGui::Render()
    ↓
Original EndScene() continues
```

### 5. DXVK (Vulkan Renderer)

**Optional Feature**: Translates Direct3D 9 calls to Vulkan

**Benefits**:
- Better performance on modern GPUs
- Lower CPU overhead
- Better multi-threading

**Integration**: DXVK is compiled as separate DLLs that can be optionally enabled via configuration.

### 6. Reverse Engineering Techniques

**Game SDK Reconstruction**:
- Debug symbols from GameCube version
- Manual reverse engineering
- Pattern recognition
- Structure alignment analysis

**Result**: Accurate C++ headers matching game's internal structures.

---

## Summary

RE4 Tweaks is a sophisticated mod framework that demonstrates advanced Windows programming, reverse engineering, and game modding techniques. It achieves non-invasive integration with the game through DLL injection, uses pattern-based memory scanning for resilience against updates, and provides a robust configuration system for user customization.

The modular architecture allows easy addition of new features, and the extensive use of reverse-engineered structures provides type-safe access to game internals. The project serves both as a practical mod for improving the game experience and as an educational resource for understanding game modding techniques.
