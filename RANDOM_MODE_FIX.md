# Random Scenario Mode - Complete Fix

## Overview
Fixed the train simulator to support seamless continuous gameplay in Random Scenario mode, where players can advance to the next station without full game restart.

## Problem Statement
1. After reaching destination in Random mode, game would freeze
2. Unrealistic 0.3 km/h velocity jump when trying to accelerate  
3. Timer wouldn't count down after advancing
4. Score card wouldn't show
5. Doors sounds not playing correctly

## Solution Architecture

### Server Changes (`tasc/server.py`)

#### 1. Added `random_mode` flag (Line 205)
```python
self.random_mode = False  # Controls game-over behavior in Random Scenario mode
```
- Set in `setInitial` command when client sends `random_mode=true`
- Preserved throughout simulation lifecycle

#### 2. Modified Physics on Finish (Line 1351)
```python
if not self.random_mode:
    self.running = False
```
- In Random mode: `running` stays `True` so physics loop continues
- In Normal mode: `running` becomes `False` (existing behavior)
- This prevents the game freeze after reaching destination

#### 3. Fixed Command Application (Lines 675-690)
- Removed artificial velocity nudge (`st.v = 0.08`)
- Now initializes acceleration filter properly at v=0
- Smooth, natural acceleration from complete stop
- No more unrealistic 0.3 km/h jumps

#### 4. Enhanced `advanceStation` Handler (Lines 1597-1655)
- Checks `st.finished == True` before processing
- Performs soft-reset: clears command queue and state artifacts
- Preserves world coordinate (`sim.state.s`) for visual continuity
- Updates scenario: new distance (`L`), grade, friction (`mu`)
- Recomputes timer budget for new scenario
- Resets acceleration filter for clean start
- Sets `running = True` and clears `finished` flag
- Comprehensive debug logging

#### 5. `setInitial` Enhancement (Lines 1562-1589)
- Accepts `random_mode` parameter from client
- Sets `sim.random_mode = bool(random_mode)`
- Logs random mode setting in DEBUG

### Client Changes (`tasc/static/index.html`)

#### 1. Score Card Display (Lines 5396-5419)
- **ALWAYS show score card when game finishes** (both random and normal mode)
- Play doors open sound when finished
- In Random mode: don't hide the card, let user press Space to continue

#### 2. Space Key Handler (Lines 5100-5130)
- Detects Random mode + Finished state
- If true: calls `continueToNextStation()` instead of `startRun()`
- If false: calls `startRun()` for normal restart
- Clear logic flow

#### 3. `continueToNextStation()` Function (Lines 4907-4955)
- Hides score overlay
- Plays doors close sound
- Randomizes inputs if Random toggle ON
- Sends `advanceStation` command to server
- Clears obstacle state
- Shows feedback message

#### 4. `startRun()` Enhancement (Lines 5041-5046)
- Detects Random toggle state
- Sends `random_mode` flag with `setInitial` command
- Server uses this to determine end-of-game behavior

## Complete Gameplay Flow (Random Mode)

```
1. Player enables Random Scenario toggle
2. Player presses Start
   └─> setInitial sent with random_mode=true
   └─> Server sets sim.random_mode = True
   
3. Train physics loop starts normally
   └─> All acceleration/braking works naturally
   
4. Train reaches destination
   └─> step() sets st.finished = True
   └─> step() computes score
   └─> Since random_mode=True, sim.running stays True
   └─> Physics loop still running
   
5. Client receives finished=True
   └─> Score overlay shows
   └─> Doors open sound plays
   
6. Player presses Space
   └─> Space handler detects: Random mode + Finished
   └─> Calls continueToNextStation()
   └─> Plays doors close sound
   └─> Randomizes new inputs (distance, grade, mu)
   └─> Sends advanceStation command
   
7. Server receives advanceStation
   └─> Verifies st.finished == True
   └─> Soft-resets: clears queue, resets state
   └─> Preserves world position (sim.state.s)
   └─> Sets new track end (sim.scn.L = s + dist)
   └─> Updates grade and friction
   └─> Clears finished flag
   └─> Keeps running=True
   
8. Client hides score overlay
   └─> Clears obstacles
   └─> Shows feedback: "다음 역으로 진행합니다"
   
9. Physics loop continues with new scenario
   └─> Train at v=0, ready to accelerate
   └─> Next set of inputs ready
   └─> Timer counting down with new budget
   
10. Player presses P1 (accelerate)
    └─> Notch applied immediately (train is stopped)
    └─> Acceleration filter initialized
    └─> Train accelerates smoothly and naturally
    └─> No artificial jumps, realistic motion
    
11. Next station reached → Repeat from step 4
```

## Key Improvements

| Issue | Solution | Result |
|-------|----------|--------|
| Game freeze after advance | Keep `running=True` in random mode | Physics continue smoothly |
| 0.3 km/h jump | Remove velocity nudge, init filter properly | Natural acceleration from 0 |
| Timer frozen | Timer enabled after advanceStation | Countdown works continuously |
| Score card not showing | Always show on finish, regardless of mode | Visual feedback present |
| Doors sounds missing | Play on finish, play on continue | Audio cues correct |
| Physics unresponsive | Initialize acceleration filter at v=0 | Immediate responsive input |

## Testing Checklist

- [ ] Start game with Random toggle ON
- [ ] Drive train to destination
- [ ] Verify score card pops up
- [ ] Verify doors open sound plays
- [ ] Press Space
- [ ] Verify doors close sound plays
- [ ] Verify score card disappears
- [ ] Verify new random values loaded (distance, grade shown)
- [ ] Verify train at 0 km/h
- [ ] Press P1 notch
- [ ] Verify smooth, natural acceleration (NO jump to 0.3 km/h)
- [ ] Verify timer counting down
- [ ] Drive normally to destination
- [ ] Repeat 5+ times for continuous loop
- [ ] Verify no accumulated bugs/state corruption

## Technical Notes

### Why Keep `running=True` in Random Mode?
- Allows physics loop (`sim_loop()`) to continue calling `step()`
- Prevents accumulated bugs from `running=False` → `running=True` transitions
- Timer can decrement and game state updates smoothly

### Why Soft-Reset Instead of Full Reset?
- Preserves world coordinate for visual continuity  
- Rails and terrain don't "jump"
- Player feels seamless transition to next station
- Eliminates jarring scene resets

### Why Initialize Acceleration Filter?
- At v=0, physics need initial condition to work properly
- Without initialization, first time step produces no acceleration
- By setting `_a_cmd_filt = power_accel`, ensures smooth response
- Natural physics from first moment of input

## File Modifications

### `tasc/server.py`
- Line 205: Added `self.random_mode` flag
- Lines 675-690: Fixed command application (removed velocity nudge)
- Line 1351: Conditional `running = False` based on `random_mode`
- Lines 1562-1589: Enhanced `setInitial` to accept and set `random_mode`
- Lines 1597-1655: Enhanced `advanceStation` handler with proper reset and debug

### `tasc/static/index.html`
- Lines 5041-5046: Send `random_mode` flag in `startRun()`
- Lines 5100-5130: Fixed Space key handler for Random mode
- Lines 5396-5419: Always show score card + play doors open sound
- Lines 4907-4955: Verified `continueToNextStation()` logic

## Debugging Tips

Enable DEBUG mode in server:
```python
DEBUG = True  # Line 14 in server.py
```

Watch for these log messages:
- `[ADVANCE] Starting soft reset:` - advance command received
- `[ADVANCE] Completed:` - advance successful
- `[APPLY_CMD] Forward notch at v=0:` - acceleration filter initialized
- `Simulation finished:` - game reached destination

## Future Improvements

1. Add visual indication of infinite loop mode
2. Add statistics tracking across multiple stations
3. Add difficulty scaling (harder future stations)
4. Add achievements for consecutive perfect runs
5. Add music/theme variations for each station

