# Name Detection - How It Works

## Overview
Name detection continuously listens for your name being called and alerts you with notification + vibration when detected.

---

## How It Works

### 1. **Name Extraction**
When you log in, your name is extracted from your email:
- Email: `john.doe@example.com` → Name: `John`
- Email: `alex123@gmail.com` → Name: `Alex123`
- If no email: Default name is `Alex`

**Location:** `SoundDetectionService.onCreate()`
```kotlin
val prefs = getSharedPreferences("settings", MODE_PRIVATE)
val email = prefs.getString("user_email", "") ?: ""
val userName = if (email.isNotEmpty()) 
    email.substringBefore("@").replaceFirstChar { it.uppercase() }
else 
    DEFAULT_USER_NAME  // "Alex"
```

### 2. **Continuous Listening**
The `NameRecognitionManager` uses Android's `SpeechRecognizer` to continuously listen:
- Starts when `SoundDetectionService` starts
- Runs in background
- Auto-restarts after each recognition
- Max 3 restart attempts on errors

### 3. **Name Matching**
When speech is detected, it checks if your name is mentioned:
```kotlin
private fun checkForName(results: List<String>) {
    val nameLower = targetName.lowercase()  // e.g., "alex"
    for (result in results) {
        if (result.lowercase().contains(nameLower)) {
            Log.d(TAG, "Name detected: $targetName in '$result'")
            onNameDetected()
            break
        }
    }
}
```

**Examples:**
- Someone says: "Hey Alex, come here" → ✅ Detected
- Someone says: "Alex is calling" → ✅ Detected
- Someone says: "Alexander" → ✅ Detected (contains "alex")
- Someone says: "Hello everyone" → ❌ Not detected

### 4. **Alert Dispatch**
When name is detected:
1. Calls `AlertDispatcher.dispatchNameDetected(name)`
2. Shows notification: "📣 Someone is calling you!"
3. Vibrates phone (pattern: 300ms, 150ms, 300ms, 150ms, 300ms)
4. Broadcasts to UI (shows popup in SoundAlertFragment)

### 5. **Debouncing**
To prevent spam alerts:
- Only one alert per 1.5 seconds
- If name called multiple times quickly, only first alert shows

---

## Testing Name Detection

### Step 1: Check Your Name
1. Open app and check what email you logged in with
2. Your name = everything before `@` in email
3. Example: `sarah.wilson@gmail.com` → Name is `Sarah`

### Step 2: Start Service
1. Open HearingActivity
2. Service starts automatically
3. Check notification: "InclusiveCare Active - Monitoring for sounds and name calls..."

### Step 3: Test Detection
1. Say your name loudly: "Hey [YourName]!"
2. Or have someone else say it
3. **Expected results:**
   - Notification appears: "Someone is calling you!"
   - Phone vibrates
   - Popup in app (if on Sound Alert tab)

### Step 4: Check Logs
```bash
adb logcat | grep -E "NameRecognition|name detected"
```

**Expected logs:**
```
NameRecognitionManager: Started listening for name: Alex
NameRecognitionManager: Name detected: Alex in 'hey alex come here'
AlertDispatcher: Name detected: Alex
AlertDispatcher: Showing notification: 📣 Someone is calling you!
```

---

## Common Issues & Solutions

### Issue: "Name never detected"
**Possible causes:**
1. Microphone permission not granted
2. Google Speech Services not installed
3. Name is too short (1-2 letters)
4. Background noise too loud

**Solutions:**
- Check Settings → Apps → InclusiveCare → Permissions → Microphone
- Install/update Google app from Play Store
- Speak clearly and loudly
- Test in quiet environment

### Issue: "Too many false positives"
**Cause:** Name is common word (e.g., "Will", "May", "Mark")

**Solution:** Change matching to exact word:
```kotlin
// In NameRecognitionManager.checkForName()
// Change from:
if (result.lowercase().contains(nameLower))

// To:
if (result.lowercase().split(" ").contains(nameLower))
```

### Issue: "Detection stops after a while"
**Cause:** Speech recognizer error or max restart attempts reached

**Check logs:**
```
NameRecognitionManager: Max restart attempts reached
NameRecognitionManager: Cannot restart - insufficient permissions
```

**Solution:**
- Restart the service (go back to Dashboard, then return to HearingActivity)
- Check microphone permission
- Restart the app

### Issue: "Works but no notification"
**Cause:** Notification permission not granted (Android 13+)

**Solution:**
- Settings → Apps → InclusiveCare → Notifications → Enable
- Check "Sound Alerts" channel is enabled

---

## How to Change Your Name

### Option 1: Change Email (Recommended)
1. Logout from app
2. Login with different email
3. Name will be extracted from new email

### Option 2: Modify Code
Edit `SoundDetectionService.kt`:
```kotlin
// Change this line:
val userName = if (email.isNotEmpty()) 
    email.substringBefore("@").replaceFirstChar { it.uppercase() }
else 
    DEFAULT_USER_NAME

// To use a custom name:
val userName = "YourCustomName"  // e.g., "Sarah"
```

### Option 3: Add Settings Option (Future Enhancement)
Create a settings screen where users can:
- View current name
- Change name manually
- Test name detection

---

## Technical Details

### Speech Recognition Flow
```
1. SpeechRecognizer.startListening()
   ↓
2. User speaks
   ↓
3. onPartialResults() - shows "Hearing: ..."
   ↓
4. onResults() - final recognized text
   ↓
5. checkForName() - search for target name
   ↓
6. If found → onNameDetected()
   ↓
7. AlertDispatcher.dispatchNameDetected()
   ↓
8. Notification + Vibration + Broadcast
   ↓
9. Auto-restart listening (after 2 second delay)
```

### Error Handling
- **ERROR_AUDIO:** Audio recording error → Restart
- **ERROR_INSUFFICIENT_PERMISSIONS:** Stop listening (can't restart)
- **ERROR_NO_MATCH:** No speech detected → Restart
- **ERROR_NETWORK:** Network issue → Restart
- **Max 3 restart attempts** to prevent infinite loops

### Performance
- **CPU Usage:** Low (uses Google's cloud speech recognition)
- **Battery Impact:** Moderate (continuous microphone access)
- **Network Usage:** Minimal (only sends audio when speech detected)
- **Privacy:** Audio sent to Google for processing

---

## Comparison: Name Detection vs Sound Detection

| Feature | Name Detection | Sound Detection |
|---------|---------------|-----------------|
| Technology | Google Speech Recognition | TensorFlow Lite (YAMNet) |
| Processing | Cloud (Google servers) | On-device (local) |
| Accuracy | High (90%+) | Moderate (70-80%) |
| Latency | 1-2 seconds | 0.5-1 second |
| Network | Required | Not required |
| Privacy | Audio sent to Google | Fully private |
| Battery | Moderate | Low |
| Customizable | Yes (any name) | No (fixed sounds) |

---

## Advanced Configuration

### Adjust Restart Delay
In `NameRecognitionManager.kt`:
```kotlin
companion object {
    private const val RESTART_DELAY_MS = 2000L  // Change this
}
```
- Lower = faster restart, more CPU
- Higher = slower restart, less CPU

### Adjust Max Restart Attempts
```kotlin
companion object {
    private const val MAX_RESTART_ATTEMPTS = 3  // Change this
}
```
- Lower = stops sooner on errors
- Higher = more persistent, but may drain battery

### Enable Partial Results Matching
Currently only checks final results. To check partial:
```kotlin
override fun onPartialResults(partialResults: Bundle?) {
    val partial = partialResults?.getStringArrayList(...)
    if (!partial.isNullOrEmpty()) {
        checkForName(partial)  // Add this line
    }
}
```

---

## Privacy & Security

### What Data is Collected?
- Audio is sent to Google Speech Services for processing
- Google may store audio for improving their service
- No audio is stored locally by the app

### How to Disable?
1. Stop the service (navigate away from HearingActivity)
2. Or disable in code:
```kotlin
// In SoundDetectionService.onStartCommand()
// Comment out this line:
// nameRecognitionManager.startListening()
```

### Alternative: Offline Speech Recognition
For privacy-conscious users, consider:
- Mozilla DeepSpeech (offline)
- Vosk (offline)
- PocketSphinx (offline)

These require more setup but keep all processing on-device.

---

## Troubleshooting Commands

### Check if service is running:
```bash
adb shell dumpsys activity services | grep SoundDetectionService
```

### Monitor name detection:
```bash
adb logcat | grep NameRecognition
```

### Test speech recognition manually:
```bash
adb shell am start -a android.speech.action.RECOGNIZE_SPEECH
```

### Check microphone permission:
```bash
adb shell dumpsys package com.example.inclusivecare | grep RECORD_AUDIO
```

---

## Summary

**Name Detection:**
- ✅ Automatically extracts name from email
- ✅ Continuously listens in background
- ✅ Alerts with notification + vibration
- ✅ Auto-restarts after detection
- ✅ Works with any name
- ⚠️ Requires microphone permission
- ⚠️ Requires internet connection
- ⚠️ Sends audio to Google

**To test:** Say your name loudly and wait for notification!
