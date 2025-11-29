# BVE Sound Engine - Debug Guide

## Recent Improvements

The BVE Sound Engine has been enhanced with comprehensive debugging output to help diagnose audio playback issues.

### What Changed

1. **Enhanced Debug Logging in `ensurePlaying()`**
   - Now logs volume, frequency, speed, and mode every 500ms (rate-limited to avoid spam)
   - Logs when each WAV starts playing
   - Shows exact values being used for audio ramping

2. **Better Error Reporting in `update()`**
   - Added early return when speed < 0.5 (engine idle)
   - Improved warning when no WAV names found
   - Shows first 5 available speeds for diagnostics

3. **Initialization State Tracking**
   - Added `lastDebugLog` instance variable to rate-limit output
   - Prevents console spam while still providing diagnostic info

4. **Critical Bug Fixes (Previous Session)**
   - CSV metadata row handling (skip "Bvets Motor Noise Table 0.01" header)
   - Async initialization in update loop (removed broken re-init check)
   - Parameter passing bugs in fadeOutTime
   - Missing error handling in source creation
   - Unbounded audio values (clamped gain to [0,1], frequency to [0.5, 2.0])

## Testing Steps

### 1. Start a Run
- Open the simulator
- Load E233 scenario
- Start a run (press the start button)

### 2. Check Browser Console
Press `F12` to open Developer Tools, go to Console tab

**Look for this on startup:**
```
BVE CSV tables loaded successfully
BVE loaded 33 WAV buffers
✓ BVE Sound Engine fully initialized and ready
```

### 3. Accelerate and Listen
- Increase power notch (press `A` or use mouse)
- **Listen carefully** for motor sounds - should gradually get louder and higher-pitched
- Watch console for:
```
BVE: MotorP1_Oflat.wav @ speed 5.0 = vol: 0.35, freq: 1.02, mode: power
BVE: Started MotorP1_Oflat.wav
```

### 4. Test Each Mode

**Power Mode (Acceleration):**
- Sounds should start quiet and get louder as speed increases
- Frequency (pitch) should increase slightly
- Multiple motor sounds layer together

**Brake Mode:**
- Notch the brake (press `Z` or use mouse)
- Brake sounds should take over smoothly
- Should be different WAV files than power mode

**Coast Mode:**
- Release all controls
- Sounds should fade out gradually
- Nothing in console (coast = silence)

## Expected Audio Files

The system is loading these 33 WAV files from `/static/E233/OriginalData/Sound/Motor/`:

**POWER SOUNDS (11 files):**
- MotorP1_Oflat.wav, MotorP1_Sub.wav, MotorP1-P2_L.wav, MotorP1-P2_H.wav
- MotorP2_Oflat.wav, MotorP2_Sub.wav
- MotorP3_L.wav, MotorP3_H.wav
- MotorP4_Oflat.wav, MotorP4_Sub.wav, MotorP4_H.wav

**BRAKE SOUNDS (12 files):**
- MotorB1_Oflat.wav, MotorB1_Sub.wav, MotorB1-B2_L.wav
- MotorB2_Oflat.wav, MotorB2_Sub.wav, MotorB2-B3_L.wav
- MotorB3_L.wav, MotorB3_H.wav
- MotorB4_Oflat.wav, MotorB4_Sub.wav, MotorB4_H.wav
- MotorB5_Sub.wav

**ATS SOUNDS (10 files):**
- ATS_Ding.wav, ATS_Beep1.wav, ATS_Beep2.wav, etc.

## Troubleshooting

### Sound Not Playing?

1. **Check file paths in console**
   - Look for: `"BVE loaded 33 WAV buffers"`
   - If fewer than 33, check `/static/E233/OriginalData/Sound/Motor/` directory

2. **Check CSV loading**
   - Look for: `"BVE CSV tables loaded successfully"`
   - If missing, check `/static/E233/OriginalData/MotorNoise/` for CSV files

3. **Check AudioContext state**
   - Browser audio policies suspend AudioContext until user interaction
   - Sound should resume after startRun() function

4. **Check browser console for errors**
   - Any red error messages?
   - Check for: `"BVE: Error starting"` or `"BVE: Buffer not found"`

5. **Check volume levels**
   - Master gain is set to 0.3 in code (30% volume)
   - Individual gains fade in at 0.3 seconds
   - If still too quiet, increase master gain in BveSoundEngine constructor

### Intermittent Sound?

1. Check for discontinuities in CSV data
   - The interpolation should smooth over gaps
   - But if too many gaps, volume might drop to zero

2. Check speed thresholds
   - Sound only plays if speed >= 0.5 km/h
   - Stationary vehicle won't produce sound

3. Check mode transitions
   - When switching between POWER/BRAKE, sounds should cross-fade
   - Might be quiet during transition

## Debug Commands

In browser console, you can test:

```javascript
// Check if engine exists
console.log(bveSoundEngine ? "Engine ready" : "Engine not initialized");

// Check loaded buffers
console.log("Loaded buffers:", Object.keys(bveSoundEngine.wavBuffers).length);

// Check CSV data
console.log("Power table speeds:", Object.keys(bveSoundEngine.powerVolTable).length);

// Force a sound to play (for testing)
// bveSoundEngine.ensurePlaying("MotorP1_Oflat.wav", 10, "power");
```

## Code Reference

### Key Files and Lines

- **BveSoundEngine class**: lines 3712-4065
- **initializeBveEngine()**: lines 4072-4138
- **Sound update loop**: lines 4673-4691
- **startRun() initialization**: lines 4460

### Key Methods

- `parseCSV(csvText)` - Parse BVE CSV format (handles metadata rows)
- `loadCsvTables(...)` - Load 4 CSV files asynchronously
- `loadWavBuffers(dirPath, names)` - Load WAV files as AudioBuffers
- `update(speed, powerNotch, ...)` - Main per-frame update
- `ensurePlaying(wavName, speed, mode)` - Create/update individual audio source
- `getInterpolatedVol(speed, wavName, mode)` - Get volume at speed
- `getInterpolatedFreq(speed, wavName, mode)` - Get frequency at speed

### Configuration

Edit in BveSoundEngine constructor (line ~3716):
```javascript
this.master.gain.value = 0.3; // Master volume (0.0 to 1.0)
```

Edit in ensurePlaying() (line ~3936):
```javascript
const fadeInTime = 0.3;  // Fade-in duration when starting
```

Edit in update loop (line ~3964):
```javascript
const fadeOutTime = 0.2;  // Fade-out duration when stopping
```

## Next Steps

1. **Test the simulator** - Follow testing steps above
2. **Report any issues** with specific console output
3. **Fine-tune volumes** if sounds are too loud or quiet
4. **Adjust fade times** if transitions are too abrupt or too slow

