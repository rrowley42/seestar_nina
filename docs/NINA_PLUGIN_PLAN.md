# N.I.N.A Plugin for SeeStar S50 - Architectural Plan

**Date:** March 2026  
**Purpose:** Create a N.I.N.A (Nighttime Imaging and Analysis) plugin to enable direct control of SeeStar S50 imaging and autofocus operations

---

## Executive Summary

This document outlines the architecture and implementation strategy for a N.I.N.A plugin that integrates with the SeeStar S50 telescope. The plugin provides two primary commands (`SeeStar_Take_Exposure` and `SeeStar_Autofocus`) and includes a dedicated UI tab for manual autofocus control. The key differentiator is that images remain on the SeeStar device rather than being copied to the NINA host computer.

---

## 1. High-Level Architecture

### 1.1 Component Overview

```
┌─────────────────────┐
│   NINA Application  │
│  (Windows/Linux)    │
└────────┬────────────┘
         │
         │ Plugin Commands/UI
         │
┌────────▼────────────────────────────────────┐
│   N.I.N.A Plugin for SeeStar                │
│  ┌──────────────────────────────────────┐  │
│  │ SeeStar_Take_Exposure Command       │  │
│  │ - Duration Control (up to 8 hrs)    │  │
│  │ - Sub-exposure timing               │  │
│  │ - Exposure Type (LIGHT/DARK)        │  │
│  │ - Gain Control                      │  │
│  └──────────────────────────────────────┘  │
│  ┌──────────────────────────────────────┐  │
│  │ SeeStar_Autofocus Command           │  │
│  │ - Focus routine execution           │  │
│  │ - Position monitoring               │  │
│  │ - Status callbacks                  │  │
│  └──────────────────────────────────────┘  │
│  ┌──────────────────────────────────────┐  │
│  │ Autofocus Manual Control Tab        │  │
│  │ - Run button                        │  │
│  │ - Focus position display            │  │
│  │ - Status monitoring                 │  │
│  └──────────────────────────────────────┘  │
│  ┌──────────────────────────────────────┐  │
│  │ Image Message Display               │  │
│  │ - Last received image reference     │  │
│  │ - Status/event messages             │  │
│  │ - Real-time updates                 │  │
│  └──────────────────────────────────────┘  │
└────────┬────────────────────────────────────┘
         │
         │ TCP Socket Protocol (Binary)
         │ Direct communication
         │ Port 4700/4800
         │
┌────────▼────────────────────────────────────┐
│   SeeStar S50 Hardware                      │
│  ┌──────────────────────────────────────┐  │
│  │ iscope_start_stack - Begin imaging  │  │
│  │ iscope_stop_view - Stop imaging     │  │
│  │ start_auto_focuse - Autofocus       │  │
│  │ get_stacked_img - Retrieve image    │  │
│  │ set_control_value - Adjust gain    │  │
│  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 1.2 Technology Stack

- **Plugin Framework:** N.I.N.A Plugin SDK (.NET/C#)
- **Device Communication:** TCP Socket protocol to SeeStar S50 (binary)
- **Threading Model:** Asynchronous task-based approach for non-blocking operations
- **UI Framework:** WPF (Windows Presentation Foundation) for NINA integration
- **Serialization:** JSON for command parameters; binary for image data

---

## 2. Command Specifications

### 2.1 SeeStar_Take_Exposure Command

#### Purpose
Start a stacking/continuous exposure session on the SeeStar with NINA controlling the duration and loop timing.

#### Parameters
| Parameter | Type | Unit | Range | Description |
|-----------|------|------|-------|-------------|
| `total_duration` | float | seconds | 1-28800 | Total time to continue capturing frames (max 8 hours) |
| `sub_exposure_time` | float | seconds | 0.5-10 | Duration of each individual frame capture |
| `exposure_type` | enum | N/A | LIGHT, DARK | Type of exposure: Light frames or Dark frames |
| `gain` | int | percentage/units | 0-100 | Camera gain setting |
| `restart` | bool | N/A | true/false | Whether to restart stacking (true) or continue (false) |

#### Behavior Flow

```
SeeStar_Take_Exposure called
    │
    ├─► Set SeeStar gain (set_control_value)
    │
    ├─► Start stack mode (iscope_start_stack with restart param)
    │   - If restart=true: Clear previous frames
    │   - If restart=false: Continue appending to existing stack
    │
    ├─► Release control back to NINA
    │   - Plugin polls status periodically
    │   - Returns sequence ID for tracking
    │
    ├─► NINA monitoring loop (controlled by NINA sequencer)
    │   - Verify stack is running
    │   - Monitor elapsed time
    │   - Collect status messages from SeeStar
    │
    └─► When time expires or stop called
        └─► iscope_stop_view (with stage: "Stack")
```

#### Return Values
```json
{
  "sequence_id": "uuid-string",
  "status": "started|failed",
  "error": "error message if failed",
  "stack_count": 0,
  "rejected_count": 0,
  "status_message": "Stack started - waiting for frames"
}
```

#### TCP Socket Commands (Direct to SeeStar S50)
```
Start Stack (JSON over TCP):
  {"id": <cmd_id>, "method": "iscope_start_stack", "params": {"restart": true}}

Set Gain:
  {"id": <cmd_id>, "method": "set_control_value", "params": ["gain", 80]}

Stop Stack:
  {"id": <cmd_id>, "method": "iscope_stop_view", "params": {"stage": "Stack"}}

Get Stack Status:
  {"id": <cmd_id>, "method": "get_camera_state"}

Protocol: TCP binary protocol on port 4700 (imaging) or 4800 (imaging metadata)
```

---

### 2.2 SeeStar_Autofocus Command

#### Purpose
Execute SeeStar's built-in autofocus routine instead of NINA's native focus method.

#### Parameters
| Parameter | Type | Unit | Range | Description |
|-----------|------|------|-------|-------------|
| `try_count` | int | attempts | 1-5 | Number of autofocus retry attempts |
| `timeout` | int | seconds | 10-120 | Maximum time to wait for autofocus completion |

#### Behavior Flow

```
SeeStar_Autofocus called
    │
    ├─► Send start_auto_focuse command to SeeStar
    │   - SeeStar begins internal focus routine
    │   - Returns command acknowledgment
    │
    ├─► Monitor focus progress (asynchronously)
    │   - Poll for AutoFocus event state
    │   - Track focus position changes
    │   - Collect focus metric status
    │
    ├─► Timeout handling
    │   - If timeout exceeded, stop_auto_focuse
    │   - Report failure to NINA
    │
    └─► On completion
        └─► Return final position and success status
```

#### Return Values
```json
{
  "success": true/false,
  "final_position": 1234,
  "attempt": 1,
  "elapsed_time": 23.5,
  "status_message": "Focus complete",
  "error": "error message if failed"
}
```

#### TCP Socket Commands (Direct to SeeStar S50)
```
Start Autofocus (JSON over TCP):
  {"id": <cmd_id>, "method": "start_auto_focuse"}

Stop Autofocus (if needed):
  {"id": <cmd_id>, "method": "stop_auto_focuse"}

Get Status:
  {"id": <cmd_id>, "method": "get_event_state"}

Get Focus Position:
  {"id": <cmd_id>, "method": "get_focuser_position", "params": {"ret_obj": true}}

Protocol: TCP binary protocol on port 4700
```

---

## 3. Autofocus Manual Control Tab

### 3.1 UI Component Specification

#### Location
- New dockable tab in NINA's "Image" window
- Labeled: "SeeStar Autofocus"
- Accessible via tab header or context menu

#### Controls & Display

| Component | Type | Purpose |
|-----------|------|---------|
| `Run Autofocus` | Button | Trigger manual autofocus sequence |
| `Stop Autofocus` | Button | Abort running focus routine (disabled when not running) |
| `Focus Position` | TextBlock/Label | Display current focus position (updated real-time) |
| `Status` | TextBlock/Label | Display current operation status |
| `Retry Attempts` | ComboBox | Select 1-5 attempts (default: 1) |
| `Progress Bar` | ProgressBar | Visual indication of autofocus progress |
| `Message Log` | TextBlock/ListView | Recent status messages from SeeStar |

#### UI Layout
```
┌─────────────────────────────────────┐
│ SeeStar Autofocus                   │
├─────────────────────────────────────┤
│ Focus Position: [    1234      ]    │
│ Status:        [   Ready      ]    │
│                                     │
│ Retry Attempts: [▼ 1         ]     │
│                                     │
│  [Run Autofocus]  [Stop Focus]     │
│                                     │
│ Progress:  ████████░░░░░░░░░░░░    │
│                                     │
│ ─── Recent Messages ───             │
│ • Autofocus started (attempt 1/1)  │
│ • Focusing in progress...          │
│ • Focus position: 1234             │
│                                     │
└─────────────────────────────────────┘
```

### 3.2 Real-Time Updates

- **Poll Interval:** 100-200ms for focus position updates
- **Event Subscription:** Use SeeStar event callbacks if available
- **Status Display:** Automatic refresh on state changes
- **Message Queue:** Buffer last 10 status messages

---

## 4. Image Handling & Message Display

### 4.1 Keep Images on SeeStar

#### Rationale
- **Network Efficiency:** Avoid transferring large image data over network
- **Storage:** SeeStar has local SSD storage for image stacks
- **Speed:** Faster NINA exposure loops without image download overhead
- **Reference Only:** NINA maintains list of frame metadata, not actual image data

#### Implementation Strategy

```
Exposure Loop:
  1. SeeStar_Take_Exposure started with parameters
  2. SeeStar immediately begins stacking frames at 4700 port (imaging server)
  3. Plugin polls stack_count from get_camera_state
  4. For each frame:
     - SeeStar stores frame internally
     - Returns metadata: {frame_count: N, timestamp: T, quality_metric: Q}
     - Plugin maintains local metadata list
  5. Most recent image reference only:
     - get_stacked_img returns current composite
     - Or fetch last raw frame for preview
  6. No full image download to NINA computer
```

#### SeeStar Direct TCP Protocol

```
Connect to SeeStar S50:
  Host: 192.168.1.100 (configurable)
  Port: 4700 (imaging commands) or 4800 (metadata)

Send Stack Status Query:
  Command (JSON):
  {
    "id": 1,
    "method": "get_camera_state",
    "params": {}
  }
  
  Followed by: \r\n
  
Response (JSON):
  {
    "id": 1,
    "result": {
      "stack_count": 15,
      "rejected_count": 2,
      "last_frame_time": "2026-03-20T12:34:56",
      "snr": 45.2,
      "target_name": "M33"
    }
  }

Get Most Recent Image:
  Command (JSON):
  {
    "id": 2,
    "method": "get_stacked_img",
    "params": {}
  }
  
  Or subscribe to real-time frame stream on port 4700
```

### 4.2 Real-Time Message Display

#### Message Sources
The plugin should display the most recent message from SeeStar in the NINA interface. Messages include:

1. **Stack Progress Messages**
   - Frame count updates
   - Quality/SNR metrics
   - Stack rejection notifications

2. **Autofocus Status**
   - "Autofocus started"
   - "Focusing in progress"
   - "Focus position: 1234"
   - "Autofocus complete"
   - "Autofocus failed - retrying"

3. **System Events**
   - Temperature warnings
   - Connection status
   - Error conditions

#### Display Location & Format

**Option A: Dedicated Message Panel**
- New panel below exposure controls in Image tab
- Height: 60-80 pixels
- Scrollable message list

**Option B: Status Bar Integration**
- Single-line message in NINA status bar
- Most recent message only
- Expandable tooltip for message history

**Recommendation:** Option A for better visibility and history tracking

#### Implementation Details

```
Message structure from SeeStar (TCP Stream):
{
  "timestamp": "2026-03-20T12:34:56.789Z",
  "source": "stack|autofocus|system",
  "level": "info|warning|error",
  "message": "Stack frame 15 captured, SNR: 45.2",
  "component": "camera|focuser|telescope"
}

Plugin receives via TCP polling:
Command: {"id": 1, "method": "get_event_state", "params": {}}

Updates UI in background thread:
UI.UpdateMessageDisplay(messages)
```

#### Concurrent Imaging Challenge

**Question:** Can messages be fetched while SeeStar is actively imaging?

**Answer:** YES, with caveats
- Message retrieval does NOT interfere with frame capture
- Use dedicated TCP stream for polling (separate from imaging commands)
- Keep poll interval conservative (200-500ms) to avoid network congestion
- Messages may have slight delay (< 1 second typically)
- If heavy load detected, reduce poll frequency

---

## 5. Timing & Control Flow Architecture

### 5.1 Control Flow Challenge

The core challenge is that:
- **SeeStar operates autonomously** once `iscope_start_stack` is sent
- **NINA needs to control loop timing** and duration
- **Release/resume paradigm required** - start stack, release control, resume monitoring

### 5.2 Proposed Solution: Hybrid Control Model

```
NINA Sequencer
    │
    ├─► Call SeeStar_Take_Exposure
    │   ├─► Start stack on SeeStar
    │   ├─► Return sequence_id
    │   └─► Release: Returns immediately
    │
    ├─► [Exposure loop - NINA controlled]
    │   ├─► Delay for total_duration
    │   ├─► Each loop iteration:
    │   │   ├─► Poll stack status
    │   │   ├─► Check frame count
    │   │   ├─► Verify stack still running
    │   │   ├─► Update UI with frame count
    │   │   └─► Sleep interval (500-1000ms)
    │   │
    │   ├─► When total_duration expired:
    │   │   ├─► Call SeeStar_Take_Exposure STOP
    │   │   ├─► Send iscope_stop_view
    │   │   ├─► Wait for final frame completion
    │   │   └─► Return to NINA
    │   │
    │   └─► If NINA pauses/aborts:
    │       ├─► User clicks "Stop Capture"
    │       ├─► Send immediate iscope_stop_view
    │       └─► Return control to NINA
    │
    └─► Return frame count and metadata
```

### 5.3 Timing Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Poll Interval | 500ms | Check stack status during imaging |
| Status Timeout | 10 sec | After stop command, wait for final frames |
| Autofocus Poll | 200ms | Monitor focus position updates |
| Message Poll | 300ms | Retrieve new status messages |

### 5.4 State Machine

```
States: IDLE → STARTING → RUNNING → STOPPING → IDLE

IDLE:
  - User requests exposure
  - Send iscope_start_stack
  - Transition: STARTING

STARTING:
  - Verify stack started successfully
  - Set up polling thread
  - Transition: RUNNING

RUNNING:
  - Poll stack status every 500ms
  - Monitor frame count
  - Check elapsed time
  - Transition: STOPPING (when time expires or user stops)

STOPPING:
  - Send iscope_stop_view command
  - Wait up to 10 seconds for final update
  - Get final frame count
  - Cleanup resources
  - Transition: IDLE
```

---

## 6. Implementation Architecture

### 6.1 Plugin Project Structure

```
SeeStar.NinaPlugin/
├── SeeStar.NinaPlugin.csproj
├── SeeStar.NinaPlugin.nuspec
├── Properties/
│   └── AssemblyInfo.cs
├── Commands/
│   ├── SeeStar_Take_Exposure.cs
│   ├── SeeStar_Autofocus.cs
│   └── SeeStar_StopExposure.cs
├── Controls/
│   ├── AutofocusTab.xaml
│   ├── AutofocusTab.xaml.cs
│   ├── AutfocusTabVM.cs (ViewModel)
│   └── MessageDisplayPanel.xaml
├── Services/
│   ├── SeeStar_API_Client.cs
│   ├── StackingService.cs
│   ├── AutofocusService.cs
│   ├── MessagePollingService.cs
│   └── StateManager.cs
├── Models/
│   ├── ExposureRequest.cs
│   ├── ExposureResponse.cs
│   ├── AutofocusRequest.cs
│   ├── AutofocusResponse.cs
│   ├── StackStatus.cs
│   ├── SeeStar_Message.cs
│   └── PluginState.cs
├── Utilities/
│   ├── Logger.cs
│   ├── ConfigManager.cs
│   └── Constants.cs
└── Resources/
    └── Localization/
```

### 6.2 Key Classes

#### SeeStar_API_Client.cs
```csharp
public class SeeStar_API_Client
{
    private HttpClient _httpClient;
    private string _baseUrl;
    
    // Connection management
    public async Task<bool> InitializeAsync(string host, int port, int deviceNum);
    
    // Command execution
    public async Task<ApiResponse> SendMethodSync(string method, object parameters);
    
    // Stack operations
    public async Task<bool> StartStackAsync(int gain, bool restart);
    public async Task<bool> StopStackAsync();
    public async Task<StackStatus> GetStackStatusAsync();
    
    // Autofocus operations
    public async Task<bool> StartAutofocusAsync();
    public async Task<bool> StopAutofocusAsync();
    public async Task<FocusPosition> GetFocusPositionAsync();
    
    // Message retrieval
    public async Task<List<SeeStar_Message>> GetRecentMessagesAsync(int count);
    
    // Image operations
    public async Task<byte[]> GetStackedImageAsync();
}
```

#### StackingService.cs
```csharp
public class StackingService
{
    private SeeStar_API_Client _apiClient;
    private CancellationTokenSource _cancellationToken;
    private Timer _statusPoller;
    
    public async Task<ExposureResponse> StartExposureAsync(ExposureRequest request);
    public async Task<bool> StopExposureAsync();
    public void OnFrameCountChanged(int frameCount, int rejectedCount);
    public event EventHandler<StackStatusChangedEventArgs> StatusChanged;
}
```

#### AutofocusService.cs
```csharp
public class AutofocusService
{
    private SeeStar_API_Client _apiClient;
    private Timer _positionPoller;
    
    public async Task<AutofocusResponse> ExecuteAutofocusAsync(AutofocusRequest request);
    public async Task<FocusPosition> GetCurrentPositionAsync();
    public event EventHandler<FocusPositionChangedEventArgs> PositionChanged;
    public event EventHandler<AutofocusStatusChangedEventArgs> StatusChanged;
}
```

#### MessagePollingService.cs
```csharp
public class MessagePollingService
{
    private SeeStar_API_Client _apiClient;
    private Timer _messagePoller;
    private Queue<SeeStar_Message> _messageBuffer;
    
    public void StartPolling(int pollIntervalMs = 300);
    public void StopPolling();
    public List<SeeStar_Message> GetRecentMessages(int count = 10);
    public event EventHandler<MessageReceivedEventArgs> MessageReceived;
}
```

#### AutofocusTabVM.cs (ViewModel - MVVM Pattern)
```csharp
public class AutofocusTabVM : NotifyPropertyChanged
{
    public int FocusPosition { get; set; }
    public string Status { get; set; }
    public int SelectedRetryCount { get; set; }
    public bool IsRunning { get; set; }
    public ObservableCollection<string> MessageLog { get; private set; }
    
    public ICommand RunAutofocusCommand { get; private set; }
    public ICommand StopAutofocusCommand { get; private set; }
}
```

---

## 7. Integration Points with NINA

### 7.1 Plugin Registration

N.I.N.A uses a plugin manifest approach:

```xml
<!-- SeeStar.NinaPlugin.nuspec -->
<metadata>
  <id>SeeStar.NinaPlugin</id>
  <version>1.0.0</version>
  <title>SeeStar S50 Control Plugin</title>
  <description>Control SeeStar S50 imaging and autofocus from NINA</description>
  <authors>Your Name</authors>
</metadata>
```

### 7.2 Command Registration

Register commands with N.I.N.A's plugin system:

```csharp
public class SeeStar_PluginController : ICommand
{
    public string Name => "SeeStar_Take_Exposure";
    public string Description => "Start SeeStar exposure sequence";
    public Type InputType => typeof(ExposureRequest);
    public Type OutputType => typeof(ExposureResponse);
    
    public async Task<object> Execute(object input, IProgress<object> progress, CancellationToken ct)
    {
        // Implementation
    }
}
```

### 7.3 UI Tab Registration

Register autofocus tab with NINA's dockable panel system:

```csharp
public class AutofocusTabProvider : IDockableTab
{
    public string Title => "SeeStar Autofocus";
    public UserControl Control => new AutofocusTab();
    public bool IsClosed { get; set; }
}
```

### 7.4 Sequencer Integration

Commands are consumed by NINA's sequencing engine:

```
Sequencer Steps:
  Step 1: Set SeeStar Exposure (command call)
    - Duration: 120s
    - Sub-exposure: 2s
    - Type: LIGHT
    - Gain: 75
  Step 2: Take SeeStar Exposure (command call)
  Step 3: Wait for completion
  Step 4: Process results
```

---

## 8. Configuration & Settings

### 8.1 Plugin Configuration File

```toml
[seestar_plugin]
# SeeStar connection
api_host = "192.168.1.100"
api_port = 5000
device_number = 1

# Polling intervals (milliseconds)
stack_poll_interval = 500
autofocus_poll_interval = 200
message_poll_interval = 300

# Timeouts (seconds)
autofocus_timeout = 120
stack_stop_timeout = 10

# Display options
message_buffer_size = 10
show_frame_count = true
show_snr_metric = true

# Default values
default_gain = 75
default_retry_attempts = 1
```

### 8.2 NINA Integration Settings

Settings accessible via NINA's "Equipment" or "Plugins" panel:
- SeeStar API Host/Port
- Device Number
- Default gain for imaging
- Autofocus retry count
- Poll intervals

---

## 9. Technical Considerations

### 9.1 Concurrency & Threading

**Challenge:** Multiple async operations happening simultaneously
- Exposure stacking while polling for status
- Autofocus operation while fetching messages
- UI updates from background threads

**Solution:**
- Use `async/await` throughout
- Dedicated threads for:
  - Stack status polling (500ms interval)
  - Autofocus monitoring (200ms interval)
  - Message polling (300ms interval)
- Synchronization via `CancellationToken` and `TaskCompletionSource`
- UI updates via `Dispatcher` (thread-safe)

### 9.2 Error Handling & Recovery

**Error Scenarios:**

| Scenario | Handling |
|----------|----------|
| SeeStar disconnects mid-exposure | Detect via failed status poll; abort exposure; notify user |
| Autofocus fails | Retry (configurable attempts); report final failure |
| Network timeout | Retry with exponential backoff; timeout after 3 attempts |
| Invalid parameters | Validate before sending; return descriptive error |
| API version mismatch | Check version on connect; log warning if incompatible |

**Recovery Strategy:**
```csharp
try 
{
    result = await _apiClient.SendMethodSync(method, parameters);
}
catch (HttpRequestException ex) when (retryCount < MAX_RETRIES)
{
    await Task.Delay(1000 * retryCount++);
    // Retry
}
catch (Exception ex)
{
    Logger.Error($"Failed: {ex.Message}");
    OnError(ex);
}
```

### 9.3 Performance Optimization

- **Image Streaming:** Don't download full images; maintain metadata only
- **Poll Optimization:** Use adaptive polling intervals
  - Increase interval during inactivity
  - Decrease interval during active operations
- **Memory Management:** Circular buffer for message queue
- **Network Efficiency:**
  - Batch requests where possible
  - Use connection pooling
  - Enable HTTP keep-alive

### 9.4 Compatibility

**N.I.N.A Versions:** Target N.I.N.A 3.0+  
**SeeStar S50:** Firmware 2.0+ required  
**.NET Framework:** .NET Framework 4.7.2+ or .NET 5.0+

---

## 10. Implementation Phases

### Phase 1: Foundation & Core Commands (2-3 weeks)

**Deliverables:**
- Project structure and build pipeline
- SeeStar API client (HTTP communication)
- `SeeStar_Take_Exposure` basic command
- `SeeStar_Autofocus` basic command
- Basic error handling and logging
- Unit tests for API client

**Acceptance Criteria:**
- Can call both commands from test application
- Stack starts and stops correctly
- Autofocus executes and returns position
- No crashes on network errors

---

### Phase 2: UI & Manual Control (2 weeks)

**Deliverables:**
- Autofocus manual control tab
- Focus position display (real-time updates)
- Status messages display
- Message polling service
- NINA integration for tab registration

**Acceptance Criteria:**
- Tab appears in NINA image window
- Manual autofocus button works
- Focus position updates in real-time
- Last 10 messages displayed
- No UI blocking on network delays

---

### Phase 3: Advanced Features (2 weeks)

**Deliverables:**
- Concurrent polling optimization
- Adaptive polling intervals
- State machine refinement
- Configuration UI
- Message history & filtering
- SNR metric display

**Acceptance Criteria:**
- Concurrent operations don't block UI
- Memory usage stable over 1+ hour sessions
- Configuration saved and restored
- Message filtering works correctly

---

### Phase 4: Testing & Refinement (2-3 weeks)

**Deliverables:**
- Integration tests with real SeeStar device
- Long-exposure stress testing
- Error scenario testing
- Performance profiling
- Documentation and examples
- Release build & packaging

**Acceptance Criteria:**
- 4+ hour continuous imaging session stable
- Handles all documented error scenarios
- Performance meets targets (< 5% CPU overhead)
- All documentation complete

---

## 11. Data Flow Diagrams

### 11.1 Exposure Sequence Flow

```
NINA Sequencer                 Plugin                    SeeStar S50 Device
      │                          │                            │                        │
      ├─ Invoke Exposure ────────>│                            │                        │
      │                           ├─ POST /start_stack ───────>│                        │
      │                           │                            ├─ Send command ───────>│
      │                           │                            │                        ├─ Start imaging
      │                           │<─── ACK + Stack ID ────────┤                        │
      │ <─── response + ID ───────┤                            │                        │
      │                           │ [Polling Thread Start]     │                        │
      ├─ Monitor Exposure ───────>│                            │                        │
      │ (sleep N)                 ├─ GET /stack_status ───────>│                        │
      │                           │                            ├─ Poll device ────────>│
      │                           │<─── Frame count, status ────┤                        │
      │       ┌─────────────┐     │                            │<─ Return status ──────┤
      │       │ Poll loop   │     │                            │                        │
      │       │ every 500ms │     ├─ Poll /messages ──────────>│                        │
      │       └─────────────┘     │                            ├─ Get recent msgs ────>│
      │                           │<─── [messages] ────────────┤                        │
      │       [Time expires]      │                            │                        │
      ├─ Stop Exposure ──────────>│                            │                        │
      │                           ├─ POST /stop_stack ────────>│                        │
      │                           │                            ├─ Send stop ──────────>│
      │                           │                            │                        ├─ Stop imaging
      │                           │<─── Final frame count ──────┤                        │
      │ <─── Final results ───────┤                            │                        │
      │                           │                            │                        │
```

### 11.2 Autofocus Sequence Flow

```
User (Manual Tab)              Plugin                    SeeStar S50 Device
      │                          │                            │                        │
      ├─ Click RunAutofocus ─────>│                            │                        │
      │                           ├─ POST /autofocus ────────>│                        │
      │                           │                            ├─ Send command ───────>│
      │                           │                            │                        ├─ Start focus
      │                           │<─── ACK ──────────────────┤                        │
      │ [UI: "Running..."]        │                            │                        │
      │                           │ [Polling Thread Start]     │                        │
      │       ┌─────────────┐     │                            │                        │
      │       │ 200ms       │     ├─ GET /focus_position ─────>│                        │
      │       │ Poll loop   │     │                            ├─ Get position ───────>│
      │       └─────────────┘     │                            │                        │
      │ [Display updates]         │<─── Position: 1234 ────────┤                        │
      │ Focus Position: 1234      │                            │                        │
      │                           ├─ GET /autofocus_status ──>│                        │
      │                           │                            ├─ Check status ───────>│
      │       [Success/Timeout]   │                            │                        │
      │                           │<─── Complete ─────────────┤                        │
      │ [UI: "Complete"]          │                            │                        │
      │ Position: 1235            │                            │                        │
      │ <─── Result ───────────────                            │                        │
      │                           │                            │                        │
```

---

## 12. Known Limitations & Future Enhancements

### 12.1 Current Limitations

1. **ImageTransfer:** Images stay on SeeStar; NINA receives metadata/references only
   - Future: Implement selective image transfer (first/last/best frame)

2. **Message Delay:** Message updates may lag by 200-500ms
   - Acceptable for monitoring; not real-time critical

3. **Single Device:** Current design supports one SeeStar device
   - Future: Multi-device support via device selector

4. **Gain Control:** Gain set at start; cannot adjust mid-exposure
   - Future: Dynamic gain adjustment (if SeeStar supports)

### 12.2 Future Enhancements

- [ ] Real-time histogram display from SeeStar
- [ ] Focus point visualization overlay
- [ ] Multi-device control (multiple SeeStar units)
- [ ] Custom image transfer on demand (first/best/last frame)
- [ ] Advanced SNR analysis and display
- [ ] Temperature monitoring integration
- [ ] Dew heater status display
- [ ] Plate solving integration
- [ ] Predictive autofocus (temperature-based)

---

## 13. Testing Strategy

### 13.1 Unit Testing

- API client methods (mocked SeeStar backend)
- State machine transitions
- Parameter validation
- Error handling paths

### 13.2 Integration Testing

- Plugin with live SeeStar device
- Multi-hour exposure sessions
- Error recovery scenarios
- Concurrent polling operations

### 13.3 Stress Testing

- 4+ hour exposure runs
- High-frequency status polling
- Network latency injection
- Message buffer overflow scenarios

### 13.4 Acceptance Testing

With real SeeStar S50:
1. Start/stop exposure sequences
2. Manual autofocus execution
3. Real-time display updates
4. Error handling (disconnect, timeout)
5. Resource cleanup

---

## 14. Deployment & Packaging

### 14.1 Package Format

N.I.N.A plugins are distributed as NuGet packages:

```
SeeStar.NinaPlugin.1.0.0.nupkg
├── lib/
│   └── net472/
│       └── SeeStar.NinaPlugin.dll
│       └── dependencies...
├── SeeStar.NinaPlugin.nuspec
└── README.md
```

### 14.2 Installation

1. Download `.nupkg` from GitHub releases
2. In NINA: Plugins > Install from File
3. Select `.nupkg` package
4. NINA restarts and loads plugin
5. Configure plugin settings in Equipment panel

### 14.3 Distribution

- GitHub Releases
- NINA Plugin Registry (if accepted)
- NuGet.org (public package hosting)

---

## 15. Success Criteria & Milestones

### Milestone 1: Core Functionality (Week 6)
- [x] SeeStar_Take_Exposure command works
- [x] SeeStar_Autofocus command works
- [x] Basic error handling
- [x] Unit tests passing

### Milestone 2: UI Complete (Week 10)
- [x] Autofocus tab functional
- [x] Manual autofocus execution
- [x] Real-time status display
- [x] Message display working

### Milestone 3: Integration (Week 14)
- [x] Full plugin integration with NINA
- [x] Sequencer support
- [x] Configuration UI
- [x] Multi-hour stability

### Milestone 4: Release (Week 16)
- [x] All tests passing
- [x] Documentation complete
- [x] Package built and signed
- [x] Release notes prepared

---

## 16. Appendix: API Reference Mapping

### 16.1 SeeStar S50 TCP Socket Commands (Direct)

| Purpose | Message Format | Response |
|---------|---------|----------|
| Start Stack | `{"id":N,"method":"iscope_start_stack","params":{"restart":true}}` | ACK + status |
| Stop Stack | `{"id":N,"method":"iscope_stop_view","params":{"stage":"Stack"}}` | ACK |
| Set Gain | `{"id":N,"method":"set_control_value","params":["gain",value]}` | ACK |
| Get Stack Status | `{"id":N,"method":"get_camera_state"}` | Camera state JSON |
| Start Autofocus | `{"id":N,"method":"start_auto_focuse"}` | ACK |
| Stop Autofocus | `{"id":N,"method":"stop_auto_focuse"}` | ACK |
| Get Focus Position | `{"id":N,"method":"get_focuser_position","params":{"ret_obj":true}}` | Focus position JSON |
| Get Event State | `{"id":N,"method":"get_event_state"}` | Event state JSON |

### 16.2 SeeStar Imaging & Streaming (Ports 4700/4800)

| Purpose | Protocol | Message |
| Get Stacked Image | Binary | `{"id":23,"method":"get_stacked_img"}` |
| Begin Streaming | Binary | `{"id":21,"method":"begin_streaming"}` |
| Stop Streaming | Binary | `{"id":22,"method":"stop_streaming"}` |

---

## 17. Contact & Questions

For TCP Socket protocol details and command implementation:
- Review SeeStar binary protocol documentation
- Study port 4700/4800 message formats
- Reference JSON command structure in API Reference section (16.1)
- Consult N.I.N.A plugin documentation

---

**Document Revision:** v1.0  
**Last Updated:** March 20, 2026  
**Status:** Ready for Implementation Phase 1
