# N.I.N.A Plugin - Quick Reference & Visual Architecture

## System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        NINA Application (WPF)                            │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ Sequencer Engine                                                   │  │
│  │  • Execute automation sequences                                    │  │
│  │  • Call plugin commands: SeeStar_Take_Exposure, SeeStar_Autofocus │  │
│  │  • Monitor command progress                                        │  │
│  └────┬──────────────────────────────────────────────────────────────┘  │
│       │                                                                   │
│  ┌────▼──────────────────────────────────────────────────────────────┐  │
│  │ SeeStar Plugin for NINA (.NET/C#)                                 │  │
│  │                                                                    │  │
│  │  Commands:                  UI Components:                        │  │
│  │  ├─ SeeStar_Take_Exposure   ├─ Autofocus Manual Tab             │  │
│  │  ├─ SeeStar_Autofocus       │  ├─ Run/Stop buttons              │  │
│  │  └─ SeeStar_StopExposure    │  ├─ Focus position display        │  │
│  │                             │  ├─ Status messages               │  │
│  │  Services:                  │  └─ Progress bar                  │  │
│  │  ├─ StackingService         ├─ Message Display Panel            │  │
│  │  ├─ AutofocusService        └─ Configuration Settings           │  │
│  │  ├─ MessagePollingService                                        │  │
│  │  └─ SeeStar_API_Client      Background Threads:                 │  │
│  │                             ├─ Stack poll (500ms)               │  │
│  │ State Management:           ├─ Autofocus poll (200ms)           │  │
│  │  └─ StateManager            └─ Message poll (300ms)             │  │
│  │                                                                    │  │
│  └────┬──────────────────────────────────────────────────────────────┘  │
└───────┼─────────────────────────────────────────────────────────────────┘
        │
        │ TCP Socket Protocol (Binary JSON)
        │
┌───────▼─────────────────────────────────────────────────────────────────┐
│                      SeeStar S50 Hardware                                │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ TCP Ports:                                                          │  │
│  │  • Port 4700: Direct imaging commands & stack control              │  │
│  │  • Port 4800: Metadata & event information                         │  │
│  │                                                                    │  │
│  │ Imaging Engine       Autofocus Module       Storage               │  │
│  │  • Frame capture      • Focus routine       • Image stack          │  │
│  │  • Stack management   • Position tracking   • Metadata            │  │
│  │  • Real-time frames   • SNR calculation     • Logs                │  │
│  │  • Dark frame library • Temperature comp.   • Messages             │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Command Execution Flows

### SeeStar_Take_Exposure Flow

```
NINA Calls Command
    ↓
Plugin.SeeStar_Take_Exposure(duration=120s, sub_exp=2s, type=LIGHT, gain=75)
    ↓
┌─ Validation
│  ├─ Check parameters valid
│  ├─ Verify SeeStar is connected
│  └─ Verify not already stacking
│
├─ Send API calls to SeeStar ALP:
│  ├─ Set gain: {"method":"set_control_value","params":["gain",75]}
│  ├─ Start stack: {"method":"iscope_start_stack","params":{"restart":true}}
│  └─ Get initial status: {"method":"get_camera_state"}
│
├─ Create async polling task:
│  ├─ Poll every 500ms
│  ├─ Track frame count
│  ├─ Monitor for errors
│  └─ Duration timer countdown
│
├─ While duration_remaining > 0:
│  ├─ Every 500ms: Get stack status
│  ├─ Every 300ms: Get latest messages
│  ├─ Update UI with frame count
│  └─ If error detected: abort
│
├─ When duration expires or user stops:
│  ├─ Send stop: {"method":"iscope_stop_view","params":{"stage":"Stack"}}
│  ├─ Wait up to 10s for final update
│  ├─ Get final frame count
│  └─ Cleanup polling thread
│
└─ Return to NINA
    └─ ExposureResponse {sequence_id, final_count, success}
```

### SeeStar_Autofocus Flow

```
Manual Tab "Run Autofocus" clicked
    ↓
Plugin.SeeStar_Autofocus(try_count=1, timeout=120s)
    ↓
┌─ Validation & Setup
│  ├─ Disable Run button
│  ├─ Reset focus position display
│  └─ Clear messages
│
├─ Send API call:
│  └─ Start: {"method":"start_auto_focuse"}
│
├─ UI Update:
│  └─ Status: "Autofocus running..."
│
├─ Start polling thread:
│  ├─ Every 200ms get focus position
│  ├─ Every 300ms get status/messages
│  ├─ Update UI with current position
│  └─ Monitor timeout (120s)
│
├─ While autofocus_running AND timeout > 0:
│  ├─ Poll: {"method":"get_focuser_position","params":{"ret_obj":true}}
│  ├─ Display: Focus Position: 1234
│  ├─ Poll: {"method":"get_event_state"}
│  ├─ Check: AutoFocus state = "complete" ?
│  └─ Every 300ms display new messages
│
├─ On completion (success or timeout):
│  ├─ Stop polling thread
│  ├─ Send if needed: {"method":"stop_auto_focuse"}
│  ├─ Get final position
│  ├─ UI Update: Status: "Complete" / "Failed"
│  └─ Enable Run button
│
└─ Return result
    └─ AutofocusResponse {success, final_position, elapsed_time}
```

---

## Data Structures

### ExposureRequest
```csharp
public class ExposureRequest
{
    public double TotalDurationSeconds { get; set; }      // 1-28800 (up to 8 hours)
    public double SubExposureTimeSeconds { get; set; }    // 0.5-10
    public ExposureType Type { get; set; }                // LIGHT | DARK
    public int Gain { get; set; }                         // 0-100
    public bool Restart { get; set; }                     // true = new stack, false = continue
}
```

### ExposureResponse
```csharp
public class ExposureResponse
{
    public string SequenceId { get; set; }                // UUID for tracking
    public bool Success { get; set; }
    public int FinalFrameCount { get; set; }
    public int RejectedCount { get; set; }
    public string StatusMessage { get; set; }
    public string Error { get; set; }                     // null if success
}
```

### AutofocusRequest
```csharp
public class AutofocusRequest
{
    public int TryCount { get; set; }                     // 1-5 attempts
    public int TimeoutSeconds { get; set; }               // 10-120
}
```

### AutofocusResponse
```csharp
public class AutofocusResponse
{
    public bool Success { get; set; }
    public int FinalPosition { get; set; }
    public int Attempt { get; set; }
    public double ElapsedTimeSeconds { get; set; }
    public string StatusMessage { get; set; }
    public string Error { get; set; }                     // null if success
}
```

### StackStatus
```csharp
public class StackStatus
{
    public int FrameCount { get; set; }
    public int RejectedCount { get; set; }
    public DateTime LastFrameTime { get; set; }
    public double SnrMetric { get; set; }
    public string TargetName { get; set; }
    public bool IsRunning { get; set; }
}
```

### SeeStar_Message
```csharp
public class SeeStar_Message
{
    public DateTime Timestamp { get; set; }
    public string Source { get; set; }                    // stack | autofocus | system
    public string Level { get; set; }                     // info | warning | error
    public string Message { get; set; }
    public string Component { get; set; }                 // camera | focuser | telescope
}
```

---

## Timeline & Milestones

### Phase 1: Foundation (Weeks 1-3)
- [ ] Project setup & build pipeline
- [ ] SeeStar API client implementation
- [ ] Basic command stubs
- [ ] Unit tests for API client
- **Deliverable:** Testable HTTP client

### Phase 2: Core Commands (Weeks 4-5)
- [ ] SeeStar_Take_Exposure full implementation
- [ ] SeeStar_Autofocus full implementation  
- [ ] State machine implementation
- [ ] Integration tests with mock backend
- **Deliverable:** Commands work end-to-end

### Phase 3: UI Implementation (Weeks 6-7)
- [ ] Autofocus tab XAML design
- [ ] ViewModel implementation
- [ ] Message display panel
- [ ] Real-time updates & binding
- [ ] Plugin registration with NINA
- **Deliverable:** Manual autofocus tab functional

### Phase 4: Polish & Integration (Weeks 8-9)
- [ ] Configuration UI & settings
- [ ] Error handling & recovery flows
- [ ] Performance optimization
- [ ] Long-duration stability testing
- [ ] Documentation & examples
- **Deliverable:** Release candidate

### Phase 5: Testing & Release (Weeks 10-11)
- [ ] Integration testing with real SeeStar
- [ ] User acceptance testing
- [ ] Bug fixes from testing
- [ ] Package creation & signing
- [ ] Release notes
- **Deliverable:** v1.0 public release

---

## Key Integration Points

### With NINA Framework
```
1. ICommand Interface
   - Implement Execute() method
   - Return type: ExposureResponse / AutofocusResponse
   - Async execution with CancellationToken

2. IDockableTab Interface
   - Register Autofocus tab
   - Provide UserControl for display
   - Handle lifecycle events

3. IEquipment Interface
   - Register with equipment system
   - Connect/disconnect hooks
   - Status reporting

4. Sequencer Integration
   - Commands appear in sequencer steps
   - Parameters exposed as UI
   - Progress reporting
```

### With SeeStar Backend
```
1. TCP Socket Protocol (Primary)
   - JSON commands sent directly to SeeStar S50
   - Ports 4700 (imaging) / 4800 (metadata)
   - Binary protocol with JSON message payload

2. Image Streaming (Secondary)
   - Port same as primary
   - Binary image data streaming
   - Frame metadata via JSON
```

---

## Error Handling Strategy

### Timeout Scenarios
```
Exposure Start Timeout (30s):
  - Problem: iscope_start_stack never returns ACK
  - Action: Retry once, then fail with error message
  - User: "Failed to start stack - retrying in 2 seconds"

Exposure Running Timeout (total_duration + 30s):
  - Problem: Stack still running after user requested stop
  - Action: Force iscope_stop_view after 10 seconds
  - User: "Stack forced to stop - may lose recent frames"

Autofocus Timeout (timeout_seconds):
  - Problem: Focus routine not completing
  - Action: If try_count > 1, retry; else fail
  - User: "Autofocus timed out - retrying (attempt 2/N)"

Network Timeout (5 seconds per request):
  - Problem: No response from SeeStar backend
  - Action: Retry up to 3 times with exponential backoff
  - User: "Connection lost - retrying..." (shows in message panel)
```

### Error Recovery
```
Connection Lost During Exposure:
  1. Detect via failed status poll
  2. Attempt reconnect (3 tries)
  3. If reconnect succeeds: Resume polling
  4. If reconnect fails: Abort exposure, report error

Invalid Parameters:
  1. Validate all parameters before API call
  2. Show error in UI with valid ranges
  3. Prevent command execution

API Version Mismatch:
  1. Check version on plugin startup
  2. Log warning if incompatible version
  3. Continue but show upgrade notification
```

---

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Command Startup | < 500ms | From button click to stack start |
| Status Update Latency | < 1s | From SeeStar update to UI display |
| UI Thread Block | < 100ms | No visible UI stutter |
| CPU Overhead | < 5% | During polling operations (4 threads) |
| Memory Usage | < 50MB | Plugin footprint (plugin + cached data) |
| Network Bandwidth | < 100KB/min | Status + message polling |

---

## Configuration File Example

```toml
# ~/.nina/plugins/SeeStar/config.toml

[connection]
api_host = "192.168.1.100"
api_port = 5000
device_number = 1
connection_timeout_seconds = 10

[polling]
stack_status_interval_ms = 500
autofocus_position_interval_ms = 200
message_fetch_interval_ms = 300

[timeouts]
command_response_timeout_seconds = 30
autofocus_max_duration_seconds = 120
stack_stop_grace_period_seconds = 10
network_retry_timeout_seconds = 5

[defaults]
default_gain = 75
default_autofocus_retries = 1
sub_exposure_time_seconds = 2.0

[display]
message_buffer_size = 10
show_frame_count_live = true
show_snr_metric = true
show_focus_position_updates = true
```

---

## Testing Checklist

### Manual Testing Script
```
1. Plugin Installation
   □ Plugin appears in NINA Equipment
   □ Settings accessible
   □ Tab appears in Image window

2. Exposure Command
   □ Start exposure with valid parameters
   □ Frame count updates in real-time
   □ Stop exposure works
   □ Stop at expiration time works
   □ Concurrent messaging display works

3. Autofocus Manual
   □ Click Run button → starts autofocus
   □ Focus position updates in real-time
   □ Status message updates
   □ Completion detected
   □ Stop button works mid-operation

4. Error Handling
   □ SeeStar disconnect detected
   □ Recovery on reconnect
   □ Invalid parameters rejected
   □ Timeout handled gracefully
   □ Message buffer doesn't overflow

5. Long Duration
   □ 4+ hour exposure session stable
   □ No memory leak over time
   □ Background threads cleanup properly
   □ Repeated start/stop works
```

---

## Next Steps

1. **Review & Approval**
   - Stakeholder review of this plan
   - Feedback on architecture
   - Approval to proceed with Phase 1

2. **Setup Development Environment**
   - Clone N.I.N.A source
   - Setup plugin development template
   - Configure build pipeline

3. **Begin Phase 1**
   - Create project structure
   - Implement SeeStar API client
   - Write unit tests

4. **Continuous Integration**
   - Setup GitHub Actions
   - Automated builds on commit
   - Test result reporting

---

## References

- SeeStar S50: Binary protocol documentation
- TCP Socket: Ports 4700 (imaging) / 4800 (metadata)
- N.I.N.A SDK: https://github.com/USA-RedDragon/nina
- JSON Command Format: See API Reference section (16.1 in PLAN.md)

---

**Document Version:** 1.0
**Date:** March 20, 2026
**Status:** Ready for Review
