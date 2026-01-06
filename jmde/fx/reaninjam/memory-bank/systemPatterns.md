# CLAP NINJAM Client - System Patterns

Architecture patterns, conventions, and implementation guidelines.

---

## 1. Threading Patterns

### 1.1 Thread Ownership

| Thread | Owner | Purpose |
|--------|-------|---------|
| Audio | Host (DAW) | Real-time audio processing |
| Run | Plugin | Network I/O, protocol handling |
| UI | Host (DAW) | GUI rendering, user input |

### 1.2 Lock Hierarchy

```
Audio thread:  NO LOCKS (reads atomics only)
     │
     ▼
Run thread:    state_mutex (never hold license_mutex while holding state_mutex)
     │
     ▼
UI thread:     state_mutex OR license_mutex (never both)
```

**Rule:** Never hold multiple locks. Never call into NJClient from audio thread except `AudioProc()`.

### 1.3 Atomic Access Patterns

```cpp
// UI thread writes, audio thread reads (MVP UI-touched only)
plugin->client->config_mastervolume.store(val, std::memory_order_relaxed);

// Audio thread reads
float vol = client->config_mastervolume.load(std::memory_order_relaxed);

// Run thread updates status, all threads read
cached_status.store(m_status, std::memory_order_release);
int status = cached_status.load(std::memory_order_acquire);

// Run/audio thread writes snapshot, UI reads
ui_snapshot.bpm.store(bpm, std::memory_order_relaxed);
float bpm_ui = ui_snapshot.bpm.load(std::memory_order_relaxed);
```

**Snapshot update point:** Refresh `ui_snapshot` in the Run thread after `NJClient::Run()` while holding `state_mutex`. Audio thread only writes VU fields.

**Memory ordering:**
- `relaxed` for config values (no ordering needed)
- `release/acquire` for status (establishes happens-before)

### 1.4 Event Queue Pattern (SPSC)

```cpp
// Run thread produces (never blocks)
if (!ui_queue.try_push(StatusChangedEvent{status, error})) {
    // Queue full - drop event (non-critical)
}

// UI thread consumes (drains all)
ui_queue.drain([&](UiEvent&& event) {
    std::visit([&](auto&& e) { handle(e); }, std::move(event));
});
```

### 1.5 License Callback Pattern (Dedicated Slot)

```cpp
// Run thread (in callback)
{
    std::lock_guard<std::mutex> lock(plugin->license_mutex);
    plugin->license_text = text;
}
plugin->license_pending.store(true, std::memory_order_release);

// Wait for response (with timeout)
std::unique_lock<std::mutex> lock(plugin->license_mutex);
plugin->license_cv.wait_for(lock, 60s, [&] {
    return plugin->license_response.load() != 0;
});

// UI thread
if (license_pending.load(std::memory_order_acquire)) {
    show_license_dialog();
}
// On accept/reject:
plugin->license_response.store(1, std::memory_order_release);
plugin->license_pending.store(false, std::memory_order_release);
plugin->license_cv.notify_one();
```

---

## 2. CLAP Plugin Patterns

### 2.1 Extension Discovery

```cpp
const void* plugin_get_extension(const clap_plugin_t* plugin, const char* id) {
    if (!strcmp(id, CLAP_EXT_AUDIO_PORTS)) return &s_audio_ports;
    if (!strcmp(id, CLAP_EXT_PARAMS)) return &s_params;
    if (!strcmp(id, CLAP_EXT_STATE)) return &s_state;
    if (!strcmp(id, CLAP_EXT_GUI)) return &s_gui;
    return nullptr;
}
```

### 2.2 Parameter ID Scheme

| ID | Name | Type |
|----|------|------|
| 0 | Master Volume | float 0.0-2.0 |
| 1 | Master Mute | bool (stepped) |
| 2 | Metronome Volume | float 0.0-2.0 |
| 3 | Metronome Mute | bool (stepped) |

**Future IDs:** Reserve 100+ for local channel, 1000+ for remote channels.

### 2.3 State Serialization

```cpp
// Always include version for migration
{
    "version": 1,
    "server": "...",
    "username": "...",
    // NO password - security
    "master": { "volume": 1.0, "mute": false },
    "metronome": { "volume": 0.5, "mute": false },
    "localChannel": { "name": "...", "transmit": true, "bitrate": 64 }
}
```

### 2.4 Audio Processing Pattern

```cpp
clap_process_status plugin_process(const clap_plugin_t* plugin,
                                    const clap_process_t* process) {
    auto* p = static_cast<NinjamPlugin*>(plugin->plugin_data);

    // 1. Sync params from host events
    sync_params_from_events(p, process->in_events);

    // 2. Get I/O pointers
    float* in[2] = { process->audio_inputs[0].data32[0],
                     process->audio_inputs[0].data32[1] };
    float* out[2] = { process->audio_outputs[0].data32[0],
                      process->audio_outputs[0].data32[1] };

    // 3. Check connection status (lock-free)
    int status = p->client->cached_status.load(std::memory_order_acquire);

    if (status == NJC_STATUS_OK) {
        // is_playing/is_seek/cursor_pos extracted from transport (omitted)
        // 4. Process through NJClient (no locks)
        const bool just_monitor = !is_playing;
    p->client->AudioProc(in, 2, out, 2, process->frames_count,
                         static_cast<int>(p->sample_rate),
                         just_monitor, is_playing, is_seek, cursor_pos);
    } else {
        // 5. Pass-through when disconnected
        memcpy(out[0], in[0], process->frames_count * sizeof(float));
        memcpy(out[1], in[1], process->frames_count * sizeof(float));
    }

    return CLAP_PROCESS_CONTINUE;
}
```

---

## 3. UI Patterns

### 3.1 ImGui Window Setup

```cpp
// Fill entire plugin area
ImGuiViewport* viewport = ImGui::GetMainViewport();
ImGui::SetNextWindowPos(viewport->Pos);
ImGui::SetNextWindowSize(viewport->Size);

ImGuiWindowFlags flags =
    ImGuiWindowFlags_NoTitleBar |
    ImGuiWindowFlags_NoResize |
    ImGuiWindowFlags_NoMove |
    ImGuiWindowFlags_NoCollapse;

ImGui::Begin("NINJAM", nullptr, flags);
// ... content ...
ImGui::End();
```

### 3.2 Collapsible Panels

```cpp
if (ImGui::CollapsingHeader("Connection", ImGuiTreeNodeFlags_DefaultOpen)) {
    ImGui::Indent();
    // ... panel content ...
    ImGui::Unindent();
}
```

### 3.3 Parameter Binding

```cpp
// Read atomic, modify, write back
float vol = plugin->param_master_volume.load(std::memory_order_relaxed);
ImGui::SetNextItemWidth(200);
if (ImGui::SliderFloat("Master Volume", &vol, 0.0f, 2.0f)) {
    plugin->param_master_volume.store(vol, std::memory_order_relaxed);
    plugin->client->config_mastervolume.store(vol, std::memory_order_relaxed);
}
```

### 3.4 NJClient API Calls from UI

```cpp
// Always take state_mutex for NJClient API (except AudioProc)
if (ImGui::Button("Connect")) {
    std::lock_guard<std::mutex> lock(plugin->state_mutex);
    plugin->client->Connect(server, user, pass);
}
```

### 3.5 VU Meter Colors

| Level | Color | RGB |
|-------|-------|-----|
| < 60% | Green | (50, 200, 50) |
| 60-85% | Yellow | (200, 200, 50) |
| > 85% | Red | (200, 50, 50) |

---

## 4. Naming Conventions

### 4.1 Files

| Pattern | Example |
|---------|---------|
| CLAP extensions | `clap_audio.cpp`, `clap_params.cpp` |
| UI panels | `ui_status.cpp`, `ui_connection.cpp` |
| Platform code | `gui_win32.cpp`, `gui_macos.mm` |
| Core (modified) | `njclient.cpp`, `netmsg.cpp` |

### 4.2 Classes/Structs

```cpp
struct NinjamPlugin;     // Main plugin instance
struct UiState;          // UI-only state
struct RemoteUser;       // Data struct
struct RemoteChannel;    // Data struct

class GuiContext;        // Abstract interface
class GuiContextWin32;   // Platform implementation
```

### 4.3 Functions

```cpp
// CLAP callbacks (C linkage pattern)
static bool plugin_init(const clap_plugin_t* plugin);
static void plugin_destroy(const clap_plugin_t* plugin);

// UI render functions
void ui_render_frame(NinjamPlugin* plugin);
void ui_render_status_bar(NinjamPlugin* plugin);

// Helper functions
static void refresh_remote_users(NinjamPlugin* plugin);
static ImU32 get_vu_color(float level);
```

### 4.4 Variables

```cpp
// Atomics
std::atomic<float> config_mastervolume;
std::atomic<int> cached_status;
UiAtomicSnapshot ui_snapshot;
std::atomic<bool> license_pending;

// Mutexes
std::mutex state_mutex;
std::mutex license_mutex;

// UI state
char server_input[256];
bool local_transmit;
float local_volume;
```

---

## 5. Error Handling

### 5.1 Connection Errors

```cpp
// Callback pushes error to UI
void on_disconnect(NinjamPlugin* p, int code) {
    std::string msg = get_error_message(code);
    p->ui_queue.try_push(StatusChangedEvent{NJC_STATUS_DISCONNECTED, msg});
}

// UI displays error
if (!state.connection_error.empty()) {
    ImGui::TextColored(ImVec4(1,0.3f,0.3f,1), "%s", state.connection_error.c_str());
}
```

### 5.2 Graceful Degradation

| Failure | Behavior |
|---------|----------|
| Queue full | Drop event (non-critical data) |
| License timeout | Auto-reject after 60s |
| Network error | Disconnect, show error, allow reconnect |
| Invalid state | Log and continue (don't crash) |

---

## 6. Build Patterns

### 6.1 Platform Detection

```cmake
if(WIN32)
    target_sources(plugin PRIVATE src/platform/gui_win32.cpp)
    target_link_libraries(plugin PRIVATE d3d11 dxgi ws2_32)
elseif(APPLE)
    target_sources(plugin PRIVATE src/platform/gui_macos.mm)
    target_link_libraries(plugin PRIVATE
        "-framework Metal"
        "-framework MetalKit"
        "-framework Cocoa")
endif()
```

### 6.2 CLAP Bundle Structure

```
# Windows
NINJAM.clap/
└── NINJAM.dll

# macOS
NINJAM.clap/
├── Contents/
│   ├── Info.plist
│   └── MacOS/
│       └── NINJAM
```

---

## 7. Testing Patterns

### 7.1 clap-validator

```bash
clap-validator build/NINJAM.clap
```

Must pass all checks before release.

### 7.2 Multi-Instance Test

1. Load plugin on two tracks
2. Connect both to same server
3. Verify independent state
4. Verify no crashes/hangs

### 7.3 State Persistence Test

1. Configure plugin (connect, set levels)
2. Save DAW project
3. Close and reopen project
4. Verify all settings restored (except password)

---

## 8. Anti-Patterns (What NOT to Do)

| ❌ Don't | ✅ Do Instead |
|----------|---------------|
| Lock mutex in audio thread | Use atomics for audio-accessed data |
| Block on UI response while holding `state_mutex` | Release `state_mutex`, then wait with timeout |
| Save password to state | Keep in memory only, clear on disconnect |
| Use globals for instance data | Store in NinjamPlugin struct |
| Call NJClient API from audio thread | Only call AudioProc() |
| Hold multiple locks | Single lock at a time |
| Allocate memory in audio thread | Pre-allocate buffers |
| Use std::shared_ptr across threads | Use raw pointers with clear ownership |
