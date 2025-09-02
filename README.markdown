# idTech4-DSP-Audio: Enhanced Audio Mod for idTech 4 (Dhewm3)

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Build Status](https://img.shields.io/badge/build-Alpha-brightgreen.svg)](https://github.com/dhewm/dhewm3)
[![Contributors](https://img.shields.io/badge/contributors-1-orange.svg)](https://github.com/yourusername/idTech4-DSP-Audio/graphs/contributors)

## Project Overview

This mod extends the idTech 4 engine (via the Dhewm3 port) to support hardware-accelerated audio processing using a USB-connected Digital Signal Processor (DSP, e.g., Texas Instruments TMS320C674x). It introduces real-time ray-traced audio effects (e.g., reverb, occlusion, distortion) and dynamic audio mode zones defined by in-game entities (`func_audio_mode`). Designed for single-player horror/cyberpunk mods like *Petabyte Madness*, it enhances immersion with spatial audio and dynamic effects while maintaining compatibility with idTech4’s sound system (e.g., OpenAL fallback). The mod is modder-friendly, with editor integration, debug tools, and extensive configuration via cVARs.

Key features include:
- DSP offload for low-latency audio processing (<20ms target).
- Entity-based audio modes for location-specific effects (e.g., combat distortion, lab reverb).
- Smooth blending of modes in overlapping zones.
- Hot-plug USB support and robust error handling.
- Debug tools and cVARs for modders and players.

**Note**: This is an alpha release (v1.0). Test thoroughly in large maps and profile performance (e.g., `com_showFPS 1`). Multiplayer is untested; focus on single-player.

## Table of Contents
- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
- [Architecture Overview](#architecture-overview)
- [idTech4 Modifications](#idtech4-modifications)
- [DSP Implementation](#dsp-implementation)
- [Optimization and Edge Cases](#optimization-and-edge-cases)
- [cVARs](#cvars)
- [Debug Tools](#debug-tools)
- [Contributing](#contributing)
- [License](#license)
- [Resources and References](#resources-and-references)

## Features
- **Hardware DSP Audio**:
  - Offloads ray tracing, convolution, and effects (e.g., reverb, distortion) to a USB DSP (TMS320C674x or similar, VID:PID 0x0403:0x6010).
  - Processes up to 1000 stochastic rays per sound source with 5 bounces for realistic occlusion/diffraction.
  - Uses TI DSPLIB for efficient FIR filtering (convolution).
- **Audio Mode Entities**:
  - `func_audio_mode` entities define spatial zones with modes (0-10, e.g., 0=explore, 1=combat) and radii.
  - Smooth blending (lerp) between overlapping zones based on distance; supports step transitions.
  - Scriptable events (`onPlayerEnter`, `onPlayerExit`) for custom behaviors (e.g., play sound cues).
- **Robust Integration**:
  - Hot-plug support: Reconnects DSP every 5s if unplugged (via `s_dspReconnectInterval`).
  - Fallback to OpenAL if DSP unavailable (`s_useDSP 0`).
  - Batched USB transfers reduce overhead (e.g., single buffer for multiple emitters).
- **Modder QoL**:
  - Editor-friendly: Place zones in D3Edit, set keys (e.g., `"mode" "1" "radius" "512"`).
  - Debug: `audioModesDebug 2` draws red spheres; `listAudioModes` dumps zone info.
  - Extensible: Define mode-specific FX in DSP firmware; script events for dynamic effects.
- **Player UX**:
  - Smooth mode transitions (tunable via `s_audioModeBlendTime`).
  - Accessibility: Toggle zones (`s_audioModesEnabled 0`) for reduced audio complexity.
  - Audio cues on zone entry (e.g., faint sting via `snd_mode_enter`).

## Installation

### Prerequisites
- **Engine**: Dhewm3 (modern idTech4 port): `git clone https://github.com/dhewm/dhewm3`.
- **DSP Hardware**: TMS320C674x-based PCB (e.g., OMAP-L138) with USB interface.
- **Dependencies**:
  - libusb-1.0 (`sudo apt install libusb-1.0-0-dev` on Ubuntu; `vcpkg install libusb` on Windows).
  - C++11+ compiler (e.g., GCC, MSVC).
  - Code Composer Studio (CCS) for DSP firmware (download from TI website).
  - TI DSPLIB for convolution functions.
- **Tools**: CMake for Dhewm3 build; D3Edit for map editing.

### Building Dhewm3 with Mod
1. Clone Dhewm3: `git clone https://github.com/dhewm/dhewm3`.
2. Copy mod files to Dhewm3 source:
   - Sound: `neo/sound/snd_dsp.h`, `neo/sound/snd_dsp.cpp`, `neo/sound/snd_system.cpp`, `neo/sound/snd_world.cpp`.
   - Game: `neo/game/AudioModeEntity.h`, `neo/game/AudioModeEntity.cpp`, `neo/game/AudioModeManager.h`, `neo/game/AudioModeManager.cpp`.
   - Defs: `def/audio_modes.def`.
3. Update `neo/CMakeLists.txt`:
   - Add to game sources: `game/AudioModeEntity.cpp`, `game/AudioModeManager.cpp`.
   - Add to sound sources: `sound/snd_dsp.cpp`.
   - Link libusb: `target_link_libraries(dhewm3 PRIVATE usb-1.0)`.
4. Build: `mkdir build && cd build && cmake .. && make -j4`.
5. Install: Copy `build/dhewm3` to game directory.

### DSP Firmware Build
1. Create a CCS project for TMS320C674x.
2. Implement firmware (see [DSP Implementation](#dsp-implementation)).
3. Link TI DSPLIB and USB stack.
4. Flash to DSP via CCS.

## Usage

### For Players
1. Connect DSP via USB (ensure drivers installed).
2. Launch game: `./dhewm3 +set s_useDSP 1`.
3. Adjust settings in console or cfg:
   - `s_audioModesEnabled 0`: Disable zones for simpler audio.
   - `s_audioModeBlendTime 1.0`: Slower transitions.
4. Hear dynamic effects (e.g., reverb in large rooms, distortion in combat zones).

If DSP disconnects, audio falls back to software (logged in console).

### For Modders
1. **Place Zones**: In D3Edit, spawn `func_audio_mode`. Set keys:
   - `"mode" "1"` (combat FX), `"radius" "512"`, `"priority" "1"`, `"blend_type" "0"` (lerp).
2. **Debug**:
   - `audioModesDebug 2`: Draws red spheres for active zones.
   - `listAudioModes`: Lists zone details (name, mode, radius, position).
3. **Extend**:
   - Define mode FX in DSP firmware (e.g., mode 1 = high distortion).
   - Add scripts in map .script, e.g.:
     ```
     void onPlayerEnter(entity zone) {
         sys.println("Entered mode " + zone.getKey("mode"));
         sys.playSound("snd_custom_cue");
     }
     ```
4. **Tune**: Use cVARs (e.g., `s_audioMaxZones 10` for smaller maps).

## Architecture Overview

The mod hooks into idTech4’s sound system (`idSoundSystemLocal`) and offloads processing to a DSP via libusb. Audio mode entities (`func_audio_mode`) define zones that apply mode-specific effects to nearby emitters. The `idAudioModeManager` class handles spatial queries and blending.

### Data Flow
```mermaid
graph TD
    A[idSoundWorldLocal] -->|Listener/Emitter Data| B[idAudioModeManager]
    B -->|Mode IDs| C[idSoundHardware_DSP]
    C -->|Scene Data, Audio| D[libusb Driver]
    D -->|Bulk Transfer| E[TMS320C674x DSP]
    E -->|Processed Audio| D --> C --> A
    subgraph "Game"
        A --> F[MixLoop: Batch Emitters]
        B -->|Spatial Query| G[gameLocal.clip]
    end
    subgraph "DSP"
        E --> H[Stochastic Ray Tracing]
        H --> I[Convolution via DSPLIB]
    end
```

## idTech4 Modifications
- **snd_dsp.h/cpp**: Implements `idSoundHardware_DSP`, handling USB comms, threading, and buffer queues with `std::unique_ptr` for leak-free memory.
- **AudioModeEntity.h/cpp**: Defines `func_audio_mode` entities with scriptable events and player entry cues.
- **AudioModeManager.h/cpp**: Manages zone queries (using `gameLocal.clip`) and distance-based mode blending.
- **snd_system.cpp**: Adds cVARs, hot-plug detection in `AsyncUpdate`.
- **snd_world.cpp**: Integrates manager, batches emitter data for DSP.
- **def/audio_modes.def**: Declares entity with editor-friendly keys.

## DSP Implementation
- **Target**: TMS320C674x with floating-point support.
- **Tasks**:
  - Parse scene data (listener/emitter positions, modes).
  - Stochastic ray tracing (up to 1000 rays, 5 bounces).
  - Convolution using DSPLIB’s `DSPF_sp_fir_gen`.
- **USB**: TI USB stack for bulk endpoints (OUT for data, IN for results).
- **Optimizations**:
  - EDMA for buffer copies.
  - Fixed-point (Q15) or inline assembly (e.g., `MPYSP`) if needed.
- **Pseudo-Code**:
  ```c
  void main() {
      init_usb();
      while (1) {
          receive_scene_data(&listener, &emitters, &modes);
          float ir[1024];
          compute_ir(ir, 1024, listener, emitters); // Stochastic rays
          convolve_audio(input, ir, output, 4096, 1024);
          send_usb(output);
      }
  }
  ```

## Optimization and Edge Cases
- **Optimizations**:
  - Spatial queries use `gameLocal.clip` for O(log n) performance.
  - Batched USB transfers (one per frame vs. per emitter).
  - Cap zones (`s_audioMaxZones`) to limit overhead.
  - Cache IRs for static emitters on DSP.
- **Edge Cases**:
  - DSP disconnect: Hot-plug retries; fallback to OpenAL.
  - No emitters/zones: Skip DSP calls, use default mode.
  - Overlapping zones: Lerp or step blending; priority resolves ties.
  - Large maps: Cap zones, tune `s_audioModeSearchRadius`.

## cVARs
Key configuration variables (see `snd_system.cpp`):
- `s_useDSP` (bool, 0): Enable DSP offload.
- `s_dspLogLevel` (int, 0): Logging (0=errors, 1=init/shutdown, 2=verbose).
- `s_dspReconnectInterval` (int, 5000): Hot-plug poll (ms).
- `s_audioModesEnabled` (bool, 1): Enable zones.
- `s_audioModeBlendTime` (float, 0.5): Blend duration (s).
- `s_audioModeSearchRadius` (float, 1024): Zone search distance.
- `s_audioModeDefault` (int, 0): Fallback mode.
- `s_audioMaxZones` (int, 20): Max zones per update.

## Debug Tools
- `audioModesDebug [0/1/2]`: 0=off, 1=list zones, 2=draw red spheres (active zones).
- `listAudioModes`: Dumps zone details (name, mode, radius, position).
- Profile with `com_showFPS 1` for performance.

## Contributing
Fork, create a feature branch, and submit PRs. Guidelines:
- Follow idTech4 style (e.g., `id*` prefixes, no exceptions).
- Test on Dhewm3; include unit tests for DSP math.
- Report bugs via GitHub Issues.

## License
GNU General Public License v3.0 (GPL-3.0). See [LICENSE](LICENSE).

## Resources and References
- **Dhewm3**: [github.com/dhewm/dhewm3](https://github.com/dhewm/dhewm3).
- **TI Documentation**: TMS320C674x DSP Reference (SPRUFE8B), DSPLIB.
- **idTech4**: [Fabien Sanglard’s Doom 3 Code Review](https://fabiensanglard.net/doom3/).
- **libusb**: [libusb.info](https://libusb.info/).
- **Audio Ray Tracing**: Inspired by US Patent 8139780B2, vaRays concepts.

For support, open an issue or contact the maintainer.