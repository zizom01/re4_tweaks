# How RE4 Tweaks Works - Comprehensive Guide

## Table of Contents
1. [Quick Overview](#quick-overview)
2. [Installation and Loading Process](#installation-and-loading-process)
3. [Initialization Workflow](#initialization-workflow)
4. [Feature Implementation Examples](#feature-implementation-examples)
5. [Configuration System](#configuration-system)
6. [User Interface](#user-interface)
7. [Runtime Operation](#runtime-operation)
8. [Development Workflow](#development-workflow)
9. [Troubleshooting and Debugging](#troubleshooting-and-debugging)

---

## Quick Overview

**RE4 Tweaks** is a DLL mod for Resident Evil 4 (Steam UHD version) that injects code into the game process to fix bugs, add features, and enhance the experience. It works by:

1. **Masquerading** as a system DLL (`dinput8.dll`)
2. **Scanning** game memory to find functions and data
3. **Hooking** game functions to modify behavior
4. **Patching** memory to fix bugs and change constants
5. **Providing** an overlay UI for real-time configuration

The mod is **non-invasive** - it doesn't modify the game executable and can be easily removed.

---

## Installation and Loading Process

### Step 1: File Placement

User copies these files to `Resident Evil 4\Bin32\`:
```
dinput8.dll        ← Our mod DLL
dinput8.ini        ← Configuration file
re4_tweaks/        ← Resources folder
```

### Step 2: Windows DLL Loading

When `bio4.exe` (the game) starts:

```
bio4.exe launches
    ↓
Windows reads import table
    ↓
Needs dinput8.dll (DirectInput library)
    ↓
Searches for dinput8.dll:
    1. Application directory (Bin32/) ← FOUND!
    2. System directory
    3. ...
    ↓
Loads OUR dinput8.dll instead of system version
    ↓
Windows calls DllMain(DLL_PROCESS_ATTACH)
    ↓
Our initialization code runs!
```

**This is called "DLL Hijacking" or "DLL Proxying"** - a legitimate technique for modding.

### Step 3: Proxy Forwarding

To prevent breaking the game's DirectInput functionality:

```cpp
// In Wrappers/dinput8/dinput8.cpp

// Load the real system dinput8.dll
HMODULE hRealDll = LoadLibrary("C:\\Windows\\System32\\dinput8.dll");

// Forward DirectInput calls to real DLL
HRESULT DirectInput8Create(...) {
    // Get real function from system DLL
    auto RealFunc = GetProcAddress(hRealDll, "DirectInput8Create");
    
    // Call it with same parameters
    return RealFunc(...);
}
```

This way:
- Game gets functional DirectInput
- We get code execution in game process
- Everyone is happy!

---

## Initialization Workflow

### Phase 1: DLL Entry Point

```cpp
// dllmain.cpp
BOOL APIENTRY DllMain(HMODULE hModule, DWORD reason, LPVOID reserved)
{
    if (reason == DLL_PROCESS_ATTACH)
    {
        // Save our DLL handle
        GlobalModuleHandle = hModule;
        
        // Don't need thread attach/detach notifications
        DisableThreadLibraryCalls(hModule);
        
        // Setup DLL forwarding
        Init_Wrappers();
        
        // Setup crash dump generation
        ExceptionHandler();
        
        // Main initialization
        Init_Main();
    }
    return TRUE;
}
```

### Phase 2: Game Detection (`Game::init()`)

Before we can modify anything, we need to find it:

```cpp
void Game::init()
{
    // 1. Detect game version by scanning for version strings
    auto pattern = hook::pattern("52 65 73 69 64 65 6E 74"); // "Resident"
    // ... analyze to determine version
    
    // 2. Find base addresses
    gameBaseAddress = (uintptr_t)GetModuleHandle(NULL);
    
    // 3. Scan for critical functions
    
    // Example: Find player update function
    // Pattern: "55 8B EC 83 EC 10 56 57 8B 7D 08"
    auto playerUpdatePattern = hook::pattern("55 8B EC 83 EC 10 56 57");
    if (!playerUpdatePattern.empty()) {
        pPlayerUpdate = playerUpdatePattern.get_first<uintptr_t>();
    }
    
    // 4. Find global data pointers
    
    // Example: Find pointer to player structure
    // Pattern scan finds instruction that references it
    auto playerPtrPattern = hook::pattern("A1 ? ? ? ? 85 C0 74 ? 8B 48");
    if (!playerPtrPattern.empty()) {
        // Extract pointer from instruction
        pPlayerPtr = *(uintptr_t**)(playerPtrPattern.get_first<uintptr_t>(1));
    }
}
```

**Why pattern scanning?**
- Game updates might change addresses
- Different game versions have different memory layouts
- Patterns are more resilient than hardcoded addresses

### Phase 3: Configuration Loading

```cpp
void ReadSettings()
{
    CSimpleIniA ini;
    ini.LoadFile("dinput8.ini");
    
    // Read settings from INI
    re4t::cfg->bFixAspectRatio = ini.GetBoolValue("DISPLAY", "FixAspectRatio", true);
    re4t::cfg->fFOVAdditional = ini.GetDoubleValue("DISPLAY", "FOV", 1.0);
    re4t::cfg->bUseRawMouseInput = ini.GetBoolValue("MOUSE", "UseRawMouseInput", true);
    // ... hundreds of settings
}
```

### Phase 4: Feature Module Initialization

Each feature is initialized independently:

```cpp
void Init_Main()
{
    // ... previous initialization ...
    
    // Display & Graphics
    re4t::init::DisplayTweaks();          // FOV, V-Sync, DPI fixes
    re4t::init::AspectRatioTweaks();      // Ultrawide support
    re4t::init::FilterXXFixes();          // Graphics filter fixes
    
    // Input
    re4t::init::KeyboardMouseTweaks();    // Mouse improvements
    re4t::init::ControllerTweaks();       // Controller fixes
    
    // Audio
    re4t::init::AudioTweaks();            // Volume sliders
    
    // Gameplay
    re4t::init::FrameRateFixes();         // 60 FPS corrections
    re4t::init::QTEfixes();               // QTE improvements
    re4t::init::Gameplay();               // Various gameplay fixes
    
    // Advanced
    re4t::init::Trainer();                // Cheat/trainer features
    re4t::init::ModExpansion();           // Modding API
    
    // ... 40+ more modules
}
```

---

## Feature Implementation Examples

### Example 1: FOV (Field of View) Modification

**Problem**: Game has narrow FOV that causes motion sickness for some players.

**Solution**: Modify the projection matrix calculation.

```cpp
// DisplayTweaks.cpp - re4t::init::DisplayTweaks()

void DisplayTweaks()
{
    if (re4t::cfg->fFOVAdditional == 1.0f)
        return; // No FOV change requested
    
    // 1. Find the function that sets up projection matrix
    auto pattern = hook::pattern("D9 05 ? ? ? ? D8 0D ? ? ? ? E8");
    if (pattern.empty())
        return; // Pattern not found, can't apply fix
    
    // 2. This instruction loads FOV value from memory
    //    Original: fld dword ptr [FOV_address]
    //    We want to multiply FOV by user's value
    
    // Get address where FOV constant is stored
    uintptr_t fovAddress = pattern.get_first<uintptr_t>(2);
    
    // Read original FOV value
    float originalFOV = *(float*)fovAddress;
    
    // Calculate new FOV
    float newFOV = originalFOV * re4t::cfg->fFOVAdditional;
    
    // Write new FOV value back to memory
    injector::WriteMemory<float>(fovAddress, newFOV, true);
}
```

**Result**: Every frame, game uses modified FOV value when rendering.

### Example 2: Raw Mouse Input

**Problem**: Game uses DirectInput which applies mouse acceleration/smoothing.

**Solution**: Hook input processing and inject raw mouse data.

```cpp
// KeyboardMouseTweaks.cpp

// Store original function pointer
typedef void (*ProcessMouseInput_t)(int dx, int dy);
ProcessMouseInput_t OriginalProcessMouseInput = nullptr;

// Our hook function
void HookedProcessMouseInput(int dx, int dy)
{
    if (re4t::cfg->bUseRawMouseInput)
    {
        // Get raw mouse input from Windows
        RAWINPUT rawInput;
        GetRawInputData(..., &rawInput, ...);
        
        // Use raw delta instead of DirectInput delta
        dx = rawInput.data.mouse.lLastX;
        dy = rawInput.data.mouse.lLastY;
        
        // Apply user sensitivity multiplier
        dx *= re4t::cfg->fMouseSensitivity;
        dy *= re4t::cfg->fMouseSensitivity;
    }
    
    // Call original function with our modified values
    OriginalProcessMouseInput(dx, dy);
}

void Init_RawMouseInput()
{
    // Find the mouse input processing function
    auto pattern = hook::pattern("55 8B EC 83 EC 08 8B 45 08 89 45 F8");
    
    // Install hook
    OriginalProcessMouseInput = 
        injector::MakeCALL(pattern.get_first(), HookedProcessMouseInput).get();
    
    // Register for raw input notifications
    RAWINPUTDEVICE rid;
    rid.usUsagePage = 0x01; // Generic desktop
    rid.usUsage = 0x02;     // Mouse
    rid.dwFlags = 0;
    rid.hwndTarget = GameWindow;
    RegisterRawInputDevices(&rid, 1, sizeof(rid));
}
```

**Result**: Mouse movements feel more responsive and accurate.

### Example 3: 60 FPS Falling Items Fix

**Problem**: At 60 FPS, items fall twice as fast (physics tied to framerate).

**Solution**: Modify the gravity constant to compensate.

```cpp
// FrameRateFixes.cpp

void FrameRateFixes()
{
    if (!re4t::cfg->bFixFallingItemsSpeed)
        return;
    
    // 1. Find where gravity is applied to falling items
    //    Pattern: instruction that adds gravity to Y velocity
    auto pattern = hook::pattern("D8 05 ? ? ? ? D9 9E ? ? ? ?");
    
    if (pattern.empty())
        return;
    
    // 2. Get address of gravity constant
    //    Instruction: fadd dword ptr [gravity_address]
    uintptr_t gravityAddr = pattern.get_first<uintptr_t>(2);
    
    // 3. Original gravity is calibrated for 30 FPS
    //    At 60 FPS, applied twice per 30 FPS frame
    //    So we need to halve it
    
    float originalGravity = *(float*)gravityAddr;
    float correctedGravity = originalGravity * 0.5f;
    
    // 4. Write corrected value
    injector::WriteMemory<float>(gravityAddr, correctedGravity, true);
}
```

**Result**: Items fall at correct speed regardless of framerate.

### Example 4: Ultrawide Aspect Ratio Support

**Problem**: Game crops image on ultrawide monitors (21:9, 32:9), HUD off-screen.

**Solution**: Modify aspect ratio calculations and HUD positioning.

```cpp
// AspectRatioTweaks.cpp

void AspectRatioTweaks()
{
    if (!re4t::cfg->bUltraWideAspectSupport)
        return;
    
    // Detect monitor aspect ratio
    float monitorAspect = (float)ScreenWidth / (float)ScreenHeight;
    
    if (monitorAspect < 2.0f)
        return; // Not ultrawide, nothing to do
    
    // 1. Fix projection matrix aspect ratio
    auto projPattern = hook::pattern("D8 0D ? ? ? ? D9 5D E0");
    if (!projPattern.empty()) {
        float* aspectAddr = projPattern.get_first<float*>(2);
        *aspectAddr = monitorAspect;
    }
    
    // 2. Reposition HUD elements
    //    Find HUD positioning code and hook it
    auto hudPattern = hook::pattern("D9 86 ? ? ? ? D8 0D ? ? ? ? D9 9E");
    if (!hudPattern.empty()) {
        // Hook function that positions HUD elements
        injector::MakeCALL(hudPattern.get_first(), HookedPositionHUD);
    }
}

void HookedPositionHUD(HUDElement* element)
{
    // Original positioning
    element->x = element->baseX;
    element->y = element->baseY;
    
    // Adjust for ultrawide
    float aspect = (float)ScreenWidth / (float)ScreenHeight;
    float standardAspect = 16.0f / 9.0f;
    float aspectDiff = aspect / standardAspect;
    
    // Move elements inward on ultrawide
    if (element->alignedToEdge) {
        element->x *= aspectDiff;
    }
}
```

**Result**: Game properly fills ultrawide screen, HUD stays visible.

---

## Configuration System

### INI File Structure

```ini
; dinput8.ini

[DISPLAY]
; Increase FOV (1.0 = default, 1.2 = 20% wider view)
FOV = 1.15

; Fix aspect ratio for ultrawide monitors
FixAspectRatio = 1
UltraWideAspectSupport = 1

[MOUSE]
; Use raw input for better accuracy
UseRawMouseInput = 1

; Mouse turning mode (0 = camera, 1 = character)
MouseTurning = 0

; Sensitivity multiplier
MouseSensitivity = 1.0

[FRAMERATE]
; Fix item falling speed at 60 FPS
FixFallingItemsSpeed = 1

; Fix door/chest opening speed at 60 FPS
FixCompartmentsSpeed = 1

[KEYBOARD]
; Allow reload without aiming first
AllowReloadWithoutAiming = 1

; Custom QTE keys
QTE_key_1 = A
QTE_key_2 = D

[TRAINER]
; Enable trainer features
EnableTrainer = 1
TrainerEnableKeys = Ctrl+F1
```

### Reading Configuration

```cpp
// Settings.cpp

void ReadSettings()
{
    CSimpleIniA ini;
    SI_Error rc = ini.LoadFile("dinput8.ini");
    
    if (rc < 0) {
        // INI file doesn't exist or can't be read
        // Use defaults and create file
        UseDefaultSettings();
        WriteSettings(); // Create INI with defaults
        return;
    }
    
    // Helper macro for reading bool values
    #define READ_BOOL(section, key, var, default) \
        var = ini.GetBoolValue(section, key, default)
    
    // Helper macro for reading float values
    #define READ_FLOAT(section, key, var, default) \
        var = (float)ini.GetDoubleValue(section, key, default)
    
    // Read all settings
    READ_BOOL("DISPLAY", "FixAspectRatio", cfg->bFixAspectRatio, true);
    READ_FLOAT("DISPLAY", "FOV", cfg->fFOVAdditional, 1.0f);
    READ_BOOL("MOUSE", "UseRawMouseInput", cfg->bUseRawMouseInput, true);
    // ... hundreds more settings
}
```

### Saving Configuration

```cpp
void WriteSettings()
{
    CSimpleIniA ini;
    
    // Set values
    ini.SetBoolValue("DISPLAY", "FixAspectRatio", cfg->bFixAspectRatio);
    ini.SetDoubleValue("DISPLAY", "FOV", cfg->fFOVAdditional);
    // ... all settings
    
    // Add comments
    ini.SetValue("DISPLAY", "FOV", nullptr, 
                 "; Field of View multiplier (1.0 = default, 1.2 = wider)");
    
    // Write to file
    ini.SaveFile("dinput8.ini");
}
```

---

## User Interface

### Configuration Menu (F1)

**Implementation**: ImGui overlay rendered on top of game.

```cpp
// cfgMenu.cpp

void RenderConfigMenu()
{
    if (!ImGui::Begin("RE4 Tweaks Configuration", &showConfigMenu)) {
        ImGui::End();
        return;
    }
    
    // Tabs for different categories
    if (ImGui::BeginTabBar("ConfigTabs")) {
        
        // Display tab
        if (ImGui::BeginTabItem("Display")) {
            ImGui::SliderFloat("FOV", &cfg->fFOVAdditional, 0.5f, 2.0f);
            ImGui::Checkbox("Fix Aspect Ratio", &cfg->bFixAspectRatio);
            ImGui::Checkbox("Ultrawide Support", &cfg->bUltraWideAspectSupport);
            ImGui::Checkbox("Disable V-Sync", &cfg->bDisableVSync);
            ImGui::EndTabItem();
        }
        
        // Mouse tab
        if (ImGui::BeginTabItem("Mouse")) {
            ImGui::Checkbox("Raw Mouse Input", &cfg->bUseRawMouseInput);
            ImGui::SliderFloat("Sensitivity", &cfg->fMouseSensitivity, 0.1f, 5.0f);
            ImGui::Checkbox("Mouse Turning", &cfg->bMouseTurning);
            ImGui::EndTabItem();
        }
        
        // ... more tabs
        
        ImGui::EndTabBar();
    }
    
    // Save button
    if (ImGui::Button("Save Settings")) {
        WriteSettings();
        ImGui::OpenPopup("Saved");
    }
    
    ImGui::End();
}

// Hook into D3D9 rendering
HRESULT WINAPI HookedEndScene(IDirect3DDevice9* pDevice)
{
    // Initialize ImGui if needed
    if (!ImGuiInitialized) {
        ImGui_ImplDX9_Init(pDevice);
        ImGuiInitialized = true;
    }
    
    // Start new ImGui frame
    ImGui_ImplDX9_NewFrame();
    ImGui_ImplWin32_NewFrame();
    ImGui::NewFrame();
    
    // Render our UI
    if (showConfigMenu)
        RenderConfigMenu();
    
    if (showTrainer)
        RenderTrainer();
    
    // Finalize and render
    ImGui::EndFrame();
    ImGui::Render();
    ImGui_ImplDX9_RenderDrawData(ImGui::GetDrawData());
    
    // Call original EndScene
    return OriginalEndScene(pDevice);
}
```

### Keyboard Input Handling

```cpp
// Window procedure hook for detecting F1 key
LRESULT CALLBACK HookedWndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    // Let ImGui handle input first
    if (ImGui_ImplWin32_WndProcHandler(hwnd, msg, wParam, lParam))
        return true;
    
    // Check for hotkeys
    if (msg == WM_KEYDOWN) {
        if (wParam == VK_F1) {
            // Toggle config menu
            showConfigMenu = !showConfigMenu;
            return 0; // Don't pass to game
        }
        
        if (wParam == VK_F2 && GetAsyncKeyState(VK_CONTROL)) {
            // Ctrl+F2 - Toggle trainer
            showTrainer = !showTrainer;
            return 0;
        }
    }
    
    // Pass to original window procedure
    return CallWindowProc(OriginalWndProc, hwnd, msg, wParam, lParam);
}
```

---

## Runtime Operation

### Frame-by-Frame Processing

```
Game Frame Begins
    ↓
┌────────────────────────────────────┐
│ 1. INPUT PHASE                     │
└────────────────────────────────────┘
    │
    ├─→ WndProc (Window Messages)
    │   ├─→ HookedWndProc intercepts
    │   ├─→ Check for hotkeys (F1, F2, etc)
    │   ├─→ ImGui processes input
    │   └─→ Pass remaining to game
    │
    ├─→ Raw Input (Mouse/Keyboard)
    │   ├─→ Hook intercepts raw input
    │   ├─→ Apply mouse sensitivity
    │   ├─→ Transform for mouse turning mode
    │   └─→ Inject into game input
    │
    └─→ Controller Input
        ├─→ Hook XInput calls
        ├─→ Apply deadzone modifications
        ├─→ Apply sensitivity multiplier
        └─→ Return to game
    ↓
┌────────────────────────────────────┐
│ 2. GAME LOGIC PHASE                │
└────────────────────────────────────┘
    │
    ├─→ Player Update
    │   └─→ Uses modified FOV value from memory
    │
    ├─→ Camera Update
    │   └─→ Aspect ratio fix affects projection
    │
    ├─→ Physics Update
    │   └─→ Corrected gravity for 60 FPS
    │
    ├─→ AI Update
    │   └─→ If trainer active, may be frozen
    │
    └─→ QTE Processing
        └─→ Modified timing for fairness at 60 FPS
    ↓
┌────────────────────────────────────┐
│ 3. RENDERING PHASE                 │
└────────────────────────────────────┘
    │
    ├─→ Game Rendering (Direct3D 9)
    │   ├─→ Draw world geometry
    │   ├─→ Draw characters/enemies
    │   ├─→ Draw effects
    │   └─→ Draw HUD
    │
    ├─→ EndScene() called
    │   ├─→ HookedEndScene() intercepts
    │   ├─→ Game rendering completes
    │   ├─→ ImGui overlay rendering begins
    │   │   ├─→ Config menu (if open)
    │   │   ├─→ Trainer UI (if enabled)
    │   │   ├─→ Debug info (if enabled)
    │   │   └─→ FPS counter
    │   └─→ Original EndScene() continues
    │
    └─→ Present() - Display frame
    ↓
┌────────────────────────────────────┐
│ 4. FRAME LIMITING                  │
└────────────────────────────────────┘
    │
    └─→ Custom frame limiter (if enabled)
        └─→ More efficient than game's default
    ↓
Next Frame Begins
```

### Memory Layout

```
Game Process Memory Space
├─ bio4.exe (game code & data)
│  ├─ .text section (code)
│  │  ├─ PlayerUpdate() ← Hooked
│  │  ├─ CameraUpdate() ← Hooked
│  │  ├─ ProcessInput() ← Hooked
│  │  └─ ... many functions
│  ├─ .data section (global variables)
│  │  ├─ pPlayer ← Accessed by mod
│  │  ├─ pCamera ← Accessed by mod
│  │  ├─ GameState ← Accessed by mod
│  │  └─ ...
│  └─ .rdata section (constants)
│     ├─ FOV_value ← Modified by mod
│     ├─ Gravity_value ← Modified by mod
│     └─ ...
│
├─ dinput8.dll (our mod)
│  ├─ .text section (mod code)
│  │  ├─ HookedPlayerUpdate()
│  │  ├─ HookedCameraUpdate()
│  │  ├─ HookedProcessInput()
│  │  └─ ... hook functions
│  ├─ .data section (mod data)
│  │  ├─ Configuration settings
│  │  ├─ UI state
│  │  └─ Trainer data
│  └─ Trampolines (jump bridges)
│     ├─ PlayerUpdate_trampoline → original code
│     ├─ CameraUpdate_trampoline → original code
│     └─ ...
│
└─ Other DLLs (d3d9.dll, kernel32.dll, etc.)
```

---

## Development Workflow

### Adding a New Feature

**Example: Add option to change player walk speed**

#### 1. Add Configuration Setting

```cpp
// Settings.h
struct Settings {
    // ... existing settings ...
    
    // New setting
    float fPlayerWalkSpeedMultiplier = 1.0f;
};

// Settings.cpp - ReadSettings()
READ_FLOAT("GAMEPLAY", "PlayerWalkSpeedMultiplier", 
           cfg->fPlayerWalkSpeedMultiplier, 1.0f);

// Settings.cpp - WriteSettings()
ini.SetDoubleValue("GAMEPLAY", "PlayerWalkSpeedMultiplier", 
                   cfg->fPlayerWalkSpeedMultiplier);
ini.SetValue("GAMEPLAY", "PlayerWalkSpeedMultiplier", nullptr,
             "; Walk speed multiplier (1.0 = normal, 2.0 = double speed)");
```

#### 2. Implement the Feature

```cpp
// Gameplay.cpp - Add new function
void Init_WalkSpeedMultiplier()
{
    if (cfg->fPlayerWalkSpeedMultiplier == 1.0f)
        return; // No change needed
    
    // Find where walk speed is calculated
    // Use IDA or Ghidra to find the pattern
    auto pattern = hook::pattern("D9 05 ? ? ? ? D8 8E ? ? ? ?");
    
    if (pattern.empty()) {
        spdlog::error("Failed to find walk speed pattern");
        return;
    }
    
    // Get address of walk speed constant
    uintptr_t speedAddr = pattern.get_first<uintptr_t>(2);
    
    // Modify it
    float originalSpeed = *(float*)speedAddr;
    float newSpeed = originalSpeed * cfg->fPlayerWalkSpeedMultiplier;
    injector::WriteMemory<float>(speedAddr, newSpeed, true);
    
    spdlog::info("Walk speed multiplier applied: {}", 
                 cfg->fPlayerWalkSpeedMultiplier);
}

// Gameplay.cpp - re4t::init::Gameplay()
void Gameplay()
{
    // ... existing features ...
    
    Init_WalkSpeedMultiplier(); // Add our new feature
}
```

#### 3. Add UI Control

```cpp
// cfgMenu.cpp - RenderConfigMenu()

if (ImGui::BeginTabItem("Gameplay")) {
    // ... existing gameplay options ...
    
    if (ImGui::SliderFloat("Walk Speed", 
                           &cfg->fPlayerWalkSpeedMultiplier, 
                           0.5f, 3.0f)) {
        // Value changed, mark as needing restart
        needsRestart = true;
    }
    
    if (needsRestart) {
        ImGui::TextColored(ImVec4(1,1,0,1), 
                          "Restart game for changes to take effect");
    }
    
    ImGui::EndTabItem();
}
```

#### 4. Test

```
1. Build mod (Visual Studio → Build Solution)
2. Copy dinput8.dll to game directory
3. Launch game
4. Press F1 to open config menu
5. Change walk speed slider
6. Save settings
7. Restart game
8. Test walking - should be faster/slower
```

#### 5. Debug if Needed

```cpp
// Add logging
spdlog::info("Pattern search result: {} matches", pattern.count());
spdlog::info("Speed address: 0x{:X}", speedAddr);
spdlog::info("Original speed: {}", originalSpeed);
spdlog::info("New speed: {}", newSpeed);
```

---

## Troubleshooting and Debugging

### Crash Dumps

If crash dumping is enabled (CrashDumps folder exists):

```cpp
// Exception handler
LONG WINAPI CustomExceptionHandler(EXCEPTION_POINTERS* pException)
{
    // Create minidump
    HANDLE hFile = CreateFile(
        "CrashDumps\\crash.dmp",
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        NULL);
    
    if (hFile != INVALID_HANDLE_VALUE) {
        MINIDUMP_EXCEPTION_INFORMATION dumpInfo;
        dumpInfo.ThreadId = GetCurrentThreadId();
        dumpInfo.ExceptionPointers = pException;
        dumpInfo.ClientPointers = FALSE;
        
        MiniDumpWriteDump(
            GetCurrentProcess(),
            GetCurrentProcessId(),
            hFile,
            MiniDumpNormal,
            &dumpInfo,
            NULL,
            NULL);
        
        CloseHandle(hFile);
    }
    
    return EXCEPTION_EXECUTE_HANDLER;
}
```

### Logging

```cpp
// Setup logging
#include <spdlog/spdlog.h>
#include <spdlog/sinks/basic_file_sink.h>

void SetupLogging()
{
    auto logger = spdlog::basic_logger_mt(
        "re4_tweaks",
        "re4_tweaks.log");
    spdlog::set_default_logger(logger);
    spdlog::set_level(spdlog::level::debug);
    spdlog::flush_on(spdlog::level::info);
}

// Use throughout code
spdlog::info("Mod initialized successfully");
spdlog::warn("Pattern not found, feature disabled");
spdlog::error("Failed to hook function at 0x{:X}", address);
```

### Common Issues

#### Pattern Not Found
```cpp
auto pattern = hook::pattern("55 8B EC...");
if (pattern.empty()) {
    spdlog::error("Pattern not found - game version mismatch?");
    return;
}
```
**Solution**: Update pattern for new game version, or make feature optional.

#### Hooking Wrong Function
```cpp
// Verify we found the right function
auto pattern = hook::pattern("55 8B EC 83 EC 10");
if (pattern.count() > 1) {
    spdlog::warn("Pattern matches {} locations - may be wrong!", 
                 pattern.count());
}
```
**Solution**: Make pattern more specific, or verify with debugger.

#### Memory Protection
```cpp
// Some memory regions are protected
DWORD oldProtect;
VirtualProtect(address, size, PAGE_EXECUTE_READWRITE, &oldProtect);
// ... modify memory ...
VirtualProtect(address, size, oldProtect, &oldProtect);
```

---

## Summary

RE4 Tweaks is a sophisticated game modification framework that:

1. **Injects** into game process via DLL proxy technique
2. **Scans** memory to find functions and data (resilient to updates)
3. **Hooks** functions to modify behavior at runtime
4. **Patches** memory to fix bugs and change constants
5. **Provides** a user-friendly configuration system
6. **Renders** an overlay UI for real-time adjustments

The modular design allows easy addition of new features, and the extensive use of reverse-engineered structures provides type-safe access to game internals. Configuration is stored in a human-readable INI file, and an in-game menu allows live tweaking without restarts (where possible).

For developers, the project serves as an excellent example of advanced Windows programming techniques including DLL injection, function hooking, memory manipulation, and game modding practices.
