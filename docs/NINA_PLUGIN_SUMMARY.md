# N.I.N.A Plugin for SeeStar S50 - Project Summary

**Status:** Planning Complete ✓  
**Created:** March 20, 2026  
**Target Implementation:** 10-12 weeks  
**Difficulty:** Advanced (Plugin Development + Real-time Async)

---

## Overview

This project creates a **N.I.N.A (Nighttime Imaging and Analysis) plugin** that integrates the **SeeStar S50 telescope** for advanced imaging and autofocus control. The plugin introduces two new commands and provides real-time manual autofocus control with live monitoring.

### Key Innovations

1. **Images Stay on SeeStar** - No large image transfers over network; NINA maintains metadata only
2. **Hybrid Control Model** - SeeStar operates autonomously; NINA controls duration via polling
3. **Real-Time Message Display** - Live status updates in NINA's interface while imaging
4. **Manual Autofocus Tab** - Dedicated UI for manual autofocus with focus position display

---

## Three Core Design Decisions

**Decision 1: Direct TCP Socket Protocol**
- Communication directly with SeeStar S50 (no intermediate server)
- Binary protocol on ports 4700/4800
- JSON commands with immediate responses
- Eliminates Python/HTTP layer

**Decision 2: 8-Hour Maximum Duration**
- Extended from 1 hour (3600s) to 8 hours (28,800s)
- Supports long observation sessions
- Hybrid control: SeeStar autonomous, NINA controls timing

**Decision 3: Keep Images on Device**
- No large image transfers to NINA computer
- Metadata/references only
- Network efficient

**Decision 4: Real-Time Status Display**
- Concurrent polling during active imaging
- Status messages shown in real-time
- No interference with frame capture

### 1. **NINA_PLUGIN_PLAN.md** (Comprehensive)
- 17 detailed sections covering all aspects
- System architecture diagrams
- Data flow diagrams
- Complete API reference
- Implementation phases (5 phases, 10-12 weeks)
- Error handling strategy
- Performance targets
- Testing checklist

**Use this for:** Understanding the full architecture and long-term strategy

---

### 2. **NINA_PLUGIN_QUICK_REF.md** (Reference)
- Visual system architecture
- Command execution flows (state diagrams)
- Data structure definitions
- Configuration examples
- Timeline overview
- Integration points with NINA
- Error handling patterns

**Use this for:** Quick lookups and understanding specific components

---

### 3. **NINA_PLUGIN_IMPLEMENTATION.md** (Development)
- Detailed project structure (directory tree)
- Complete C# class implementations
  - `SeeStar_APIClient.cs` (full code)
  - `StackingService.cs` (full code)
  - `AutofocusService.cs` (full code)
  - `SeeStar_TakeExposureCommand.cs` (full code)
  - `AutofocusTabViewModel.cs` (full code)
  - XAML UI markup for tab
- Model class definitions
- Unit test examples
- Build configuration (csproj)
- CI/CD pipeline (GitHub Actions)

**Use this for:** Actual development—copy/paste ready code patterns

---

## Core Design Decisions

### Decision 1: Keep Images on SeeStar
**Why:** Network efficiency and speed  
**How:** Plugin maintains image metadata; `get_stacked_img` called on-demand only  
**Tradeoff:** NINA cannot display live preview during exposure

---

### Decision 2: Hybrid Control Model (Release & Monitor)
**Why:** SeeStar autonomously acquires frames; NINA controls timing  
**How:** Start stack, release control, poll status every 500ms, stop when duration expires  
**Tradeoff:** NINA must implement timing logic; cannot control frame-by-frame

---

### Decision 3: Concurrent Polling (3 Background Threads)
**Why:** Multiple asynchronous operations happening simultaneously  
- Stack status polling (500ms)
- Autofocus monitoring (200ms)
- Message retrieval (300ms)
  
**Solution:** Dedicated threads + `async/await` + CancellationToken  
**Tradeoff:** Complexity; must handle thread safety carefully

---

### Decision 4: Real-Time Message Display During Imaging
**Question:** "Can we fetch messages while SeeStar is imaging?"  
**Answer:** YES! Message polling is lightweight and doesn't interfere  
**Implementation:** Separate HTTP client thread polling every 300ms

---

## Two Core Commands

### Command 1: SeeStar_Take_Exposure

```
Input Parameters:
  • total_duration: 1-28800 seconds (up to 8 hours of imaging)
  • sub_exposure_time: 0.5-10 seconds (per-frame duration)
  • exposure_type: LIGHT or DARK
  • gain: 0-100 (camera gain setting)
  • restart: true (new stack) or false (continue existing)

Behavior:
  1. Send iscope_start_stack to SeeStar
  2. Return immediately with sequence_id
  3. NINA calls this, then sequencer continues looping
  4. Plugin polls stack status every 500ms in background
  5. When total_duration expires, NINA calls stop
  6. Return final frame count to NINA

Output:
  {
    sequence_id: "uuid",
    success: true,
    final_frame_count: 45,
    rejected_count: 2,
    status_message: "Stack complete"
  }
```

**Key Insight:** Non-blocking. Once started, control returns immediately.

---

### Command 2: SeeStar_Autofocus

```
Input Parameters:
  • try_count: 1-5 (retry attempts)
  • timeout: 10-120 seconds

Behavior:
  1. Send start_auto_focuse to SeeStar
  2. Poll focus position every 200ms
  3. Poll autofocus status every poll cycle
  4. When complete (or timeout), return
  5. If failed and try_count > 1, retry

Output:
  {
    success: true,
    final_position: 1234,
    attempt: 1,
    elapsed_time: 23.5,
    status_message: "Focus complete"
  }
```

**Key Insight:** Blocks until complete. Called from manual tab or sequencer.

---

## UI Components

### Autofocus Manual Control Tab

```
┌────────────────────────────┐
│ SeeStar Autofocus          │
├────────────────────────────┤
│                            │
│ Focus Position     1234    │
│ Status             Ready   │
│                            │
│ Retry Attempts [∨ 1     ]  │
│                            │
│ [Run Autofocus] [Stop]    │
│                            │
│ Progress: ████████░░░░     │
│                            │
│ ─ Recent Messages ─        │
│ • Autofocus started (1/1) │
│ • Focusing in progress... │
│ • Position: 1234          │
│                            │
└────────────────────────────┘
```

---

## Implementation Timeline

| Phase | Duration | Deliverable | Criteria |
|-------|----------|-------------|----------|
| 1: Foundation | 2-3 wks | API Client + Tests | HTTP communication works |
| 2: Core Commands | 2 wks | Both commands functional | Commands start/stop/monitor successfully |
| 3: UI & Manual Tab | 2 wks | Autofocus tab + messages | Tab displays real-time data |
| 4: Advanced Features | 2 wks | Config UI + optimization | Concurrent polling stable |
| 5: Testing & Release | 2-3 wks | Package + docs | 4+ hour sessions stable, release ready |

**Total: 10-12 weeks**

---

## Technology Stack

| Component | Tech | Notes |
|-----------|------|-------|
| Plugin Framework | .NET Framework 4.7.2 or .NET 6.0 | N.I.N.A SDK compatible |
| UI | WPF (XAML) | NINA standard |
| Communication | TCP Socket (Binary) | Direct to SeeStar S50 |
| Commands | ICommand (NINA) | Pattern for sequencer integration |
| Async | async/await + Threading | For non-blocking operations |
| Testing | xUnit + Moq | Unit test patterns provided |
| CI/CD | GitHub Actions | Automated builds on push |

---

## Critical Success Factors

### 1. Robust Error Handling
- Network disconnections during exposure
- Timeouts on any API call
- SeeStar firmware incompatibility
- Recovery without losing data

### 2. Non-Blocking UI
- All background operations async
- Proper thread synchronization
- UI updates via Dispatcher
- No 100% CPU polling

### 3. Long Duration Stability
- 4+ hour exposure sessions must work
- No memory leaks
- Threads cleanup properly
- Polling rates optimized

### 4. Message Display Reliability
- Concurrent image capture + message polling
- Sub-1-second message latency acceptable
- Buffer doesn't overflow (circular queue)

### 5. Clear State Transitions
- State machine handles all scenarios
- Commands respect state (can't start twice)
- Graceful error recovery

---

## Known Limitations (v1.0)

1. ❌ **No live preview** - Images stay on SeeStar
2. ❌ **Single device only** - Can't control multiple SeeStar units
3. ❌ **Gain set at start** - Cannot adjust gain mid-exposure
4. ❌ **NINA loops independently** - SeeStar operates autonomously once started
5. ❌ **Message delay** - 200-500ms lag acceptable but not real-time

---

## Future Enhancements

- ✓ Multi-device support (multiple SeeStar units)
- ✓ Selective image transfer (first/last/best frame)
- ✓ Real-time histogram display
- ✓ Advanced SNR analysis
- ✓ Temperature integration
- ✓ Plate solving integration
- ✓ Predictive autofocus based on temperature

---

## Key SeeStar API Calls Used

From existing `seestar_device.py`:

```python
# Start imaging
{"method":"iscope_start_stack", "params":{"restart":true}}

# Stop imaging  
{"method":"iscope_stop_view", "params":{"stage":"Stack"}}

# Set gain
{"method":"set_control_value", "params":["gain", 75]}

# Get status
{"method":"get_camera_state"}

# Start autofocus
{"method":"start_auto_focuse"}

# Get focus position
{"method":"get_focuser_position", "params":{"ret_obj":true}}

# Get status events
{"method":"get_event_state"}

# Get stacked image
{"method":"get_stacked_img"}
```

All calls go through:  
`PUT http://seestars:4700 (TCP Socket Protocol)`

Direct TCP socket commands with JSON parameters sent to SeeStar S50

---

## File Locations

All planning documents are in the workspace root:

```
seestar_alp/
├── NINA_PLUGIN_PLAN.md              ← Main comprehensive plan (17 sections)
├── NINA_PLUGIN_QUICK_REF.md         ← Visual guide & quick reference
├── NINA_PLUGIN_IMPLEMENTATION.md    ← Code examples & project structure
├── NINA_PLUGIN_SUMMARY.md           ← This file
└── [plugin project will go here when created]
```

---

## How to Get Started

### Step 1: Review the Plans (Asynchronous)
- [ ] Read NINA_PLUGIN_PLAN.md sections 1-5 (architecture)
- [ ] Review NINA_PLUGIN_QUICK_REF.md diagrams
- [ ] Check NINA_PLUGIN_IMPLEMENTATION.md for code patterns

### Step 2: Setup Development Environment
- [ ] Clone N.I.N.A source code
- [ ] Create new plugin project from template
- [ ] Setup build pipeline

### Step 3: Begin Phase 1
- [ ] Create project structure (see IMPLEMENTATION.md)
- [ ] Copy SeeStar_APIClient.cs code
- [ ] Implement HTTP communication
- [ ] Write unit tests

### Step 4: Iterative Development
- [ ] Phase 1 → Core commands → UI → Testing → Release
- [ ] Each phase has acceptance criteria (see PLAN.md)

### Step 5: Testing with Real Hardware
- [ ] Integration testing with live SeeStar
- [ ] 4+ hour stress testing
- [ ] Error scenario validation

---

## Questions to Revisit

During development, you may need to revisit these:

1. **Image Data Handling**
   - NINA sequencer expects images; how to handle metadata-only response?
   - Solution: Custom ResultData class with metadata + reference

2. **Gain Adjustment**
   - Can we change gain after exposure starts?
   - Answer: Probably not; set before start (limitation documented)

3. **Exposure Type (LIGHT/DARK)**
   - How does SeeStar distinguish? Via UI setting or parameter?
   - Answer: Check existing code in `action_set_exposure`

4. **Focus Metric**
   - What metric does SeeStar use for quality?
   - Answer: SNR available in `get_camera_state`

5. **Message Rate**
   - How many messages per minute from SeeStar?
   - Answer: Varies; design for up to 100/minute

---

## Success Metrics

✓ **Phase 1 Complete:** API client fully tested, HTTP communication works  
✓ **Phase 2 Complete:** Both commands callable and functional  
✓ **Phase 3 Complete:** Autofocus tab usable with real-time updates  
✓ **Phase 4 Complete:** 1+ hour stability proven, no memory leaks  
✓ **Phase 5 Complete:** 4+ hour sessions stable, release ready  

**Final Success:** Plugin ships, users report successful imaging sessions with SeeStar from NINA

---

## References & Resources

- **SeeStar Code:** `device/seestar_device.py` (2500+ lines reference)
- **API Examples:** `bruno/Seestar Alpaca API/` (Bruno API client files)
- **N.I.N.A SDK:** https://github.com/USA-RedDragon/nina
- **Async Patterns:** Microsoft docs on async/await for .NET
- **Plugin Examples:** NINA GitHub plugin repository

---

## Contact & Questions

For clarification on any plan sections:
1. Reference the appropriate document section
2. Check IMPLEMENTATION.md for code examples
3. Review SeeStar source: `device/seestar_device.py`
4. Refer to Bruno API files for endpoint specifics

---

## Next Action

**✓ Planning Complete**  
**→ Ready for Phase 1: Foundation**

When ready to begin development:
1. Create project from IMPLEMENTATION.md structure
2. Start with SeeStar_APIClient.cs (first 400 lines of code)
3. Write HTTP tests against mock backend
4. Proceed to Phase 2

---

**Document Version:** 1.0  
**Status:** ✓ Complete & Ready for Development  
**Created:** March 20, 2026
