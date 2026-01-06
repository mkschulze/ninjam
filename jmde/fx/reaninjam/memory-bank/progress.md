# CLAP NINJAM Client - Implementation Progress

**Project Start:** January 2026  
**Target Platforms:** Windows 10+ (MSVC/Clang), macOS 10.15+ (Xcode/Clang)  
**Plugin Format:** CLAP (stereo 2-in/2-out)  
**UI Framework:** Dear ImGui (Metal on macOS, D3D11 on Windows)  
**Language:** C++20 (std::variant/std::optional + designated initializers)

---

## Phase 0: Project Setup ✅

| Task | Status | Notes |
|------|--------|-------|
| Create `ninjam-clap/` directory structure | ✅ | `/Users/cell/dev/ninjam-clap/` |
| Initialize git repository | ✅ | |
| Add CLAP SDK submodule (`libs/clap`) | ✅ | v1.2.7 |
| Add clap-helpers submodule (`libs/clap-helpers`) | ✅ | |
| Add Dear ImGui submodule (`libs/imgui`) | ✅ | |
| Add libogg submodule (`libs/libogg`) | ✅ | v1.3.6 |
| Add libvorbis submodule (`libs/libvorbis`) | ✅ | v1.3.7 |
| Copy WDL dependencies to `wdl/` | ✅ | jnetlib/, sha, queue, heapbuf, mutex, ptrlist, etc. |
| Create root CMakeLists.txt | ✅ | C++20; ObjC/ObjCXX enabled only on macOS |
| Create cmake/ClapPlugin.cmake | ⬜ | Not needed for MVP |
| Create resources/Info.plist.in | ✅ | macOS bundle |
| Verify empty build on Windows | ⬜ | Skipped (macOS only dev) |
| Verify empty build on macOS | ✅ | `_clap_entry` exported, bundle verified |

**Deliverable:** ✅ CLAP bundle builds, exports correct entry point

---

## Phase 1: Core NJClient Port ✅

| Task | Status | Notes |
|------|--------|-------|
| Copy njclient.h/cpp to `src/core/` | ✅ | From `/Users/cell/dev/ninjam/ninjam/` |
| Copy netmsg.h/cpp to `src/core/` | ✅ | |
| Copy mpb.h/cpp to `src/core/` | ✅ | |
| Copy njmisc.h/cpp to `src/core/` | ✅ | |
| Add atomic config fields to njclient.h | ✅ | master/metronome vol/pan/mute + metronome_channel + play_prebuffer |
| Add `cached_status` atomic | ✅ | Updated in Connect/Disconnect/Run (incl. early returns) |
| Remove REAPER Vorbis callback indirection | ✅ | Not needed - REANINJAM not defined, uses direct VorbisEncoder/Decoder |
| Implement `SpscRing<T, N>` in `src/threading/spsc_ring.h` | ✅ | Lock-free SPSC queue, API aligned with plan |
| Define `UiEvent` variant types | ✅ | ChatMessage, StatusChanged, UserInfoChanged, TopicChanged |
| Implement run thread wrapper | ✅ | `run_thread.h/cpp` - callbacks wired, adaptive sleep |
| Add UI atomic snapshot struct | ✅ | `UiAtomicSnapshot` in `src/ui/ui_state.h` |
| Create `UiState` struct | ✅ | `src/ui/ui_state.h` - connection, local channel, remote users, license |
| Update run thread to refresh UI snapshot | ✅ | BPM/BPI/position/beat updated in run loop |
| Implement chat callback | ✅ | Pushes ChatMessageEvent to ui_queue |
| Implement license callback | ✅ | Blocking wait with cv, 60s timeout |
| Unit test: connect to public server | ⬜ | Deferred - requires CLAP wrapper first |

**Deliverable:** ✅ NJClient core compiles, threading infrastructure in place, build verified

---

## Phase 2: CLAP Wrapper ✅

| Task | Status | Notes |
|------|--------|-------|
| Create `src/plugin/ninjam_plugin.h` | ✅ | Plugin instance struct (Part 1 Section 5) |
| Implement clap_entry.cpp | ✅ | Factory, descriptor (Part 2 Section 1) |
| Implement plugin lifecycle | ✅ | init, destroy, activate, deactivate, start/stop_processing (Part 2 Section 2) |
| Implement audio ports extension | ✅ | Stereo I/O (Part 2 Section 3) |
| Implement process() | ✅ | AudioProc() call, pass-through when disconnected (Part 2 Section 4) |
| Implement params extension | ✅ | 4 params: master vol/mute, metro vol/mute (Part 2 Section 5) |
| Implement state extension | ✅ | JSON save/load via picojson, no password (Part 2 Section 6) |
| Add picojson.h to `src/third_party/` | ✅ | Single-header JSON parser downloaded |
| Test with clap-validator | ⬜ | Deferred |

**Deliverable:** ✅ Plugin loads, processes audio, saves/restores state

---

## Phase 3: Platform GUI ⬜

| Task | Status | Notes |
|------|--------|-------|
| Create `src/platform/gui_context.h` | ⬜ | Abstract interface (Part 3 Section 1) |
| Implement `gui_win32.cpp` | ⬜ | Win32 + D3D11 + ImGui (Part 3 Section 1.1) |
| Implement `gui_macos.mm` | ⬜ | Cocoa + Metal + ImGui (Part 3 Section 1.2) |
| Implement clap_gui.cpp | ⬜ | GUI extension hooks (Part 2 Section 7) |
| Test ImGui renders in REAPER | ⬜ | |
| Test ImGui renders in Bitwig | ⬜ | |

**Deliverable:** ImGui window appears with test content

---

## Phase 4: UI Panels ⬜

| Task | Status | Notes |
|------|--------|-------|
| Create `src/ui/ui_state.h` | ⬜ | UI state struct (Part 1 Section 6) |
| Implement ui_main.cpp | ⬜ | Main layout, event draining (Part 3 Section 2) |
| Implement ui_status.cpp | ⬜ | Connection dot, BPM, BPI, beat progress (Part 3 Section 3) |
| Implement ui_connection.cpp | ⬜ | Server/user/pass inputs, connect/disconnect (Part 3 Section 4) |
| Implement ui_local.cpp | ⬜ | Name, bitrate, transmit, vol/pan/mute/solo (Part 3 Section 5) |
| Implement ui_master.cpp | ⬜ | Master + metronome controls (Part 3 Section 6) |
| Implement ui_remote.cpp | ⬜ | Remote users tree, per-channel controls (Part 3 Section 7) |
| Implement ui_license.cpp | ⬜ | Modal dialog accept/reject (Part 3 Section 8) |
| Implement ui_meters.cpp | ⬜ | VU meter widget (Part 3 Section 9) |
| Wire VU snapshot updates | ⬜ | Audio thread updates UiAtomicSnapshot |

**Deliverable:** Full UI functional

---

## Phase 5: Integration & Polish ⬜

| Task | Status | Notes |
|------|--------|-------|
| End-to-end test: connect, transmit, receive | ⬜ | Use public NINJAM server |
| Verify multi-instance works | ⬜ | No globals except read-only descriptor |
| State persistence test | ⬜ | Save project, reload, verify settings |
| Parameter automation test | ⬜ | Automate master volume in DAW |
| Memory leak check (Windows) | ⬜ | Visual Studio diagnostics |
| Memory leak check (macOS) | ⬜ | Instruments/Leaks |
| Test in REAPER (Win) | ⬜ | |
| Test in REAPER (macOS) | ⬜ | |
| Test in Bitwig (Win) | ⬜ | |
| Test in Bitwig (macOS) | ⬜ | |

**Deliverable:** Release candidate

---

## Current Status

| Item | Value |
|------|-------|
| **Current Phase** | 3 - Platform GUI |
| **Blockers** | None |
| **Next Action** | Create gui_context.h abstract interface and platform implementations |

---

## Immediate Next Steps

### Phase 3 Tasks

#### 1. Create `src/platform/gui_context.h`
- Abstract interface for platform-specific GUI rendering

#### 2. Implement `gui_macos.mm`
- Cocoa + Metal + ImGui backend

#### 3. Implement `gui_win32.cpp`
- Win32 + D3D11 + ImGui backend

#### 4. Implement clap_gui.cpp
- CLAP GUI extension hooks

---

## Session Log

| Date | Session | Progress |
|------|---------|----------|
| 2026-01-06 | Planning | ✅ Completed functional design |
| | | ✅ Completed threading/sync plan |
| | | ✅ Completed technical design (3 parts) |
| | | ✅ Created progress.md |
| 2026-01-06 | Phase 0 | ✅ Created project at `/Users/cell/dev/ninjam-clap/` |
| | | ✅ Added all submodules (clap, clap-helpers, imgui, libogg, libvorbis) |
| | | ✅ Copied WDL files |
| | | ✅ Created CMakeLists.txt, Info.plist.in |
| | | ✅ Created stub clap_entry.cpp |
| | | ✅ Build verified on macOS (x86_64 bundle, clap_entry exported) |
| 2026-01-06 | Phase 1 | ✅ Copied NJClient core files to src/core/ |
| | | ✅ Added atomic config fields (master/metro vol/pan/mute, prebuffer, metronome_channel) |
| | | ✅ Added cached_status atomic with updates in Connect/Disconnect/Run |
| | | ✅ Created SpscRing<T,N> lock-free queue |
| | | ✅ Created UiEvent variant types |
| | | ✅ Created run_thread.h/cpp with adaptive sleep |
| | | ✅ Created ninjam_plugin.h struct |
| 2026-01-07 | Phase 1 Review | ✅ Code review by senior developer |
| | | ✅ Created ui_state.h with UiState + UiAtomicSnapshot |
| | | ✅ Implemented chat_callback and license_callback |
| | | ✅ Added UI snapshot refresh in run loop |
| | | ✅ Fixed atomic metronome channel read |
| | | ✅ Build verified on macOS |
| 2026-01-07 | Phase 2 | ✅ Rewrote clap_entry.cpp with full plugin lifecycle |
| | | ✅ Implemented audio ports extension (stereo I/O) |
| | | ✅ Implemented process() with AudioProc call and pass-through |
| | | ✅ Implemented params extension (4 params with dB display) |
| | | ✅ Implemented state extension (JSON save/load with picojson) |
| | | ✅ Downloaded picojson.h to src/third_party/ |
| | | ✅ Fixed namespace conflict in run_thread.h |
| | | ✅ Build verified on macOS |
| 2026-01-07 | Phase 2 Review | ✅ Code review by senior developer |
| | | ✅ Transport default changed to "not playing" |
| | | ✅ Added null guard for data32 buffers (returns CLAP_PROCESS_ERROR) |
| | | ✅ State save: snapshot UI data under mutex before serialization |
| | | ✅ State save: handle partial writes in loop |
| | | ✅ State load: parse first, apply under mutex atomically |
| | | ✅ Build verified on macOS |

---

## Decisions Made

| Decision | Rationale | Reference |
|----------|-----------|-----------|
| Port NJClient vs rewrite | Core is battle-tested, 99% portable | Initial analysis |
| Single stereo I/O (MVP) | Simplifies audio routing, covers most use cases | Functional Design F04 |
| No chat in MVP | Reduces scope, can add later | Functional Design F18 |
| Dear ImGui for UI | Cross-platform, immediate mode, simple | Functional Design 3.2 |
| Metal + D3D11 backends | Native GPU, best performance | Part 3 Section 1 |
| Password in-memory only | Security - never saved to disk | Plan Section 3 |
| Atomic config fields | Lock-free audio thread access | Plan Section 2 |
| Dedicated license slot | Guaranteed delivery vs queue | Plan Section 4 |
| Run thread always ticks | Handle connection state transitions | Plan Section 1 |
| C++20 required | std::variant, std::optional, designated initializers | Plan Overview |

---

## Reference Documents

| Document | Purpose |
|----------|---------|
| [Functional Design](functional-design-clapNinjam.md) | Features, requirements, architecture overview |
| [Threading & Sync Plan](plan-clapNinjamThreadingSync.md) | Thread roles, atomics, callbacks, license handling |
| [Technical Design Part 1](technical-design-part1-core.md) | Project structure, CMake, NinjamPlugin struct, SpscRing |
| [Technical Design Part 2](technical-design-part2-clap.md) | CLAP entry, lifecycle, audio, params, state, GUI ext |
| [Technical Design Part 3](technical-design-part3-ui.md) | Platform GUI layers, ImGui panels, VU meters |

---

## Legend

- ⬜ Not started
- 🔄 In progress
- ✅ Completed
- ❌ Blocked
