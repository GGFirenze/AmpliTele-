# Guides & Surveys Installation - Canal+ Espace Client Demo

## Overview

This document explains how Amplitude Guides & Surveys was installed on the Canal+ Espace Client demo webapp while maintaining GDPR compliance.

## Installation Method

Since the Canal+ demo uses **manual SDK loading** for GDPR compliance (scripts only load after cookie consent), we followed the [Amplitude Browser SDK 2 plugin approach](https://amplitude.com/docs/guides-and-surveys/sdk).

### SDK Loading Order

```
1. Analytics Browser SDK 2.32.0
2. Session Replay Plugin 1.13.10
3. Guides & Surveys SDK (engagement.js)
```

All scripts are loaded **dynamically** via JavaScript only after the user accepts cookies.

## Code Changes

### 1. Added Engagement Script Loading

In the `loadAmplitudeSDK()` function, after Session Replay loads:

```javascript
// Load Guides & Surveys SDK
const engagementScript = document.createElement('script');
engagementScript.src = `https://cdn.amplitude.com/script/${AMPLITUDE_API_KEY}.engagement.js`;
engagementScript.onload = () => {
    console.log('✅ Guides & Surveys SDK loaded');
    sdkLoaded = true;
    resolve();
};
engagementScript.onerror = (error) => {
    console.warn('⚠️ Guides & Surveys SDK failed to load (optional feature):', error);
    // Still resolve - G&S is optional, don't block analytics
    sdkLoaded = true;
    resolve();
};
document.body.appendChild(engagementScript);
```

### 2. Added Plugin Registration

In `initializeAmplitude()`, before `amplitude.init()`:

```javascript
// Add Guides & Surveys plugin (if loaded successfully)
if (window.engagement && window.engagement.plugin) {
    window.amplitude.add(window.engagement.plugin());
    console.log('✅ Guides & Surveys plugin added');
}
```

## Key Implementation Notes

| Aspect | Details |
|--------|---------|
| **API Key** | `b16fbcf70484402413f712160befa8cf` (same for Analytics and G&S) |
| **GDPR Compliance** | SDK only loads after cookie consent accepted |
| **Error Handling** | Non-blocking - if G&S fails, analytics continues |
| **No manual init/boot** | Plugin approach handles initialization automatically |

## Verification

Open browser console after accepting cookies. You should see:

```
✅ Amplitude Analytics SDK loaded
✅ Session Replay Plugin loaded
✅ Guides & Surveys SDK loaded
✅ Guides & Surveys plugin added
✅ Amplitude initialized with Session Replay, Frustration Analytics, and Guides & Surveys
```

## Creating Guides & Surveys

1. Go to your Amplitude project
2. Navigate to **Guides & Surveys** in the left menu
3. Create a new Guide or Survey
4. Set targeting rules (page URL, user properties, events, etc.)
5. Publish - it will appear on the demo automatically

## References

- [Amplitude Guides & Surveys SDK Documentation](https://amplitude.com/docs/guides-and-surveys/sdk)
- Demo file: `canalplus-espace-client-demo.html`
- Local testing: `python3 -m http.server 8888`

---

*Installation completed: January 2026*

