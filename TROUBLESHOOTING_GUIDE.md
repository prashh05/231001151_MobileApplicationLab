# InclusiveCare - Troubleshooting Guide

## Microphone & Sound Detection Not Working

### Step 1: Check Permissions
1. Open the app
2. Go to HearingActivity
3. Check logcat for these messages:
   ```
   HearingActivity: Initializing app with permissions granted
   SoundDetectionService: SoundDetectionService started
   SoundRecognitionManager: AudioRecord started successfully
   SoundRecognitionManager: Listening thread started
   ```

If you see "Permission denied for microphone access", permissions are not granted.

**Fix:**
- Go to Android Settings → Apps → InclusiveCare → Permissions
- Enable "Microphone" permission
- Restart the app

### Step 2: Check AudioRecord Initialization
Look for these error messages in logcat:
- `AudioRecord not initialized properly`
- `AudioRecord not recording`
- `Invalid buffer size`

**Common Causes:**
1. **Another app is using the microphone**
   - Close all other apps that might use the mic (voice recorders, video calls, etc.)
   - Restart the device

2. **Permission not granted at runtime**
   - The app requests permissions when HearingActivity starts
   - Make sure to tap "Allow" when prompted

3. **Android 12+ restrictions**
   - On Android 12+, microphone access requires explicit user permission
   - Check notification shade for "App is using microphone" indicator

### Step 3: Test Microphone Manually
1. Open HearingActivity
2. Navigate to "Sound Alert" tab
3. Watch the sound level bar at the top
4. Make a loud noise (clap, speak loudly)
5. The bar should move and show percentage

**Expected behavior:**
- Status shows: "🟢 Listening... (Level: XX%)"
- Sound level bar moves when you make noise
- If amplitude is always 0%, microphone is not working

### Step 4: Check Service Status
In logcat, filter by "SoundDetectionService":
```
SoundDetectionService: SoundDetectionService started
SoundDetectionService: Sound recognition started
SoundDetectionService: Name recognition started
```

If service doesn't start:
- Check if notification appears: "InclusiveCare Active - Monitoring for sounds..."
- If no notification, service failed to start
- Check logcat for error messages

### Step 5: Test Sound Detection
1. Play test sounds on another device:
   - Doorbell sound
   - Alarm sound
   - Baby crying sound
   - Dog barking

2. Watch for:
   - Notification with vibration
   - Alert popup in the app
   - New entry in the alerts list

**If no detection:**
- Check if amplitude is being detected (Step 3)
- Verify yamnet.tflite model exists in assets folder
- Look for "YAMNet model loaded successfully" in logcat

### Step 6: Check Vibration
If sound is detected but no vibration:

1. Check device settings:
   - Settings → Sound → Vibration
   - Make sure vibration is enabled

2. Check app notification settings:
   - Settings → Apps → InclusiveCare → Notifications
   - Enable "Sound Alerts" channel
   - Enable vibration for notifications

3. Check logcat for vibration errors

### Step 7: Test Name Recognition
1. Make sure you're logged in (name is extracted from email)
2. Say your name loudly: "Alex" (or your email username)
3. Should see notification: "Someone is calling you!"

**If not working:**
- Check if Google Speech Recognition is installed
- Go to Settings → Apps → Google → Speech Recognition
- Make sure it's enabled and updated

### Common Error Messages & Solutions

#### "AudioRecord not initialized properly"
**Cause:** Microphone permission denied or hardware issue
**Fix:** 
- Grant microphone permission
- Restart device
- Check if other apps can use microphone

#### "Permission denied for microphone access"
**Cause:** RECORD_AUDIO permission not granted
**Fix:**
- Settings → Apps → InclusiveCare → Permissions → Microphone → Allow

#### "Speech recognition not available"
**Cause:** Google Speech Services not installed
**Fix:**
- Install "Google" app from Play Store
- Update Google app to latest version

#### "Failed to load YAMNet model"
**Cause:** yamnet.tflite file missing from assets
**Fix:**
- Verify file exists: app/src/main/assets/yamnet.tflite
- Rebuild the app

#### "Error reading audio: -3"
**Cause:** AudioRecord in bad state (ERROR_INVALID_OPERATION)
**Fix:**
- Stop and restart the service
- Navigate away from HearingActivity and back
- Restart the app

### Testing Checklist

- [ ] Microphone permission granted
- [ ] Notification appears when service starts
- [ ] Sound level bar moves when making noise
- [ ] Status shows "Listening... (Level: XX%)"
- [ ] Test sound triggers notification
- [ ] Vibration works
- [ ] Alert appears in the list
- [ ] Name detection works
- [ ] Switching tabs doesn't crash

### Debug Commands

Enable verbose logging:
```bash
adb shell setprop log.tag.SoundRecognitionManager VERBOSE
adb shell setprop log.tag.SoundDetectionService VERBOSE
adb shell setprop log.tag.NameRecognitionManager VERBOSE
```

Check running services:
```bash
adb shell dumpsys activity services | grep SoundDetectionService
```

Check permissions:
```bash
adb shell dumpsys package com.example.inclusivecare | grep permission
```

### Still Not Working?

1. **Clear app data:**
   - Settings → Apps → InclusiveCare → Storage → Clear Data
   - Restart app and login again

2. **Reinstall app:**
   - Uninstall completely
   - Reinstall from Android Studio
   - Grant all permissions when prompted

3. **Check device compatibility:**
   - Minimum Android version: 8.0 (API 26)
   - Microphone must be functional
   - Test microphone with voice recorder app

4. **Check logcat for detailed errors:**
   ```bash
   adb logcat | grep -E "SoundRecognition|SoundDetection|NameRecognition|AudioRecord"
   ```

### Expected Logcat Output (Working State)

```
HearingActivity: Initializing app with permissions granted
HearingActivity: Sound detection service started
SoundDetectionService: SoundDetectionService started
SoundRecognitionManager: YAMNet model loaded successfully
SoundRecognitionManager: Creating AudioRecord with buffer size: 3200
SoundRecognitionManager: AudioRecord started successfully
SoundRecognitionManager: Listening thread started
NameRecognitionManager: Started listening for name: Alex
SoundAlertFragment: Registered broadcast receiver
```

When sound is detected:
```
SoundRecognitionManager: Sound detected: Doorbell (0.87)
SoundDetectionService: Sound: Doorbell @ 0.87
AlertDispatcher: Showing notification: 🔔 Someone at the Door
```
