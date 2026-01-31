# Cross-Device Identity Resolution for QR Code Login Flows

## Amplitude Implementation Guide for Streaming Services

**Version:** 1.0  
**Last Updated:** January 2026  
**Author:** Giuliano Giannini, Amplitude

---

## Table of Contents

1. [The Problem](#the-problem)
2. [Why Funnels Break](#why-funnels-break)
3. [Solution Overview](#solution-overview)
4. [Implementation Guide](#implementation-guide)
   - [Solution 1: Shared Flow ID](#solution-1-shared-flow-id-recommended)
   - [Solution 2: Server-Side Event Stitching](#solution-2-server-side-event-stitching)
   - [Solution 3: User ID Propagation](#solution-3-user-id-propagation)
   - [Solution 4: Group Analytics](#solution-4-group-analytics)
5. [Recommended Architecture](#recommended-architecture)
6. [Event Taxonomy](#event-taxonomy)
7. [Funnel Analysis Setup](#funnel-analysis-setup)
8. [Best Practices](#best-practices)
9. [FAQ](#faq)

---

## The Problem

When users authenticate via QR code (e.g., linking a TV to their account using a mobile phone), the login journey spans **two separate devices**:

```
┌─────────────┐                      ┌─────────────┐
│     TV      │                      │    Phone    │
│             │                      │             │
│ Device ID:  │                      │ Device ID:  │
│ tv_abc123   │                      │ phone_xyz   │
│             │                      │             │
│ Shows QR    │  ──── User Scans ──► │ Completes   │
│ Code        │                      │ Login       │
└─────────────┘                      └─────────────┘
```

**The Challenge:** Amplitude sees these as two completely separate anonymous users because each device has a unique Device ID.

---

## Why Funnels Break

Consider a typical QR login funnel:

| Step | Event | Device | Device ID |
|------|-------|--------|-----------|
| 1 | QR Code Displayed | TV | `tv_abc123` |
| 2 | QR Code Scanned | Phone | `phone_xyz789` |
| 3 | Login Started | Phone | `phone_xyz789` |
| 4 | Login Completed | Phone | `phone_xyz789` |

**What Amplitude sees:**
- **"User A"** (TV): Started journey, never completed → 0% conversion
- **"User B"** (Phone): Completed login but never started → Appears from nowhere

**Result:** Your funnel shows artificially low conversion rates because Amplitude cannot connect these two devices as the same user journey.

---

## Solution Overview

| Solution | Complexity | Reliability | Best For |
|----------|------------|-------------|----------|
| Shared Flow ID | Low | High | Quick implementation, funnel analysis |
| Server-Side Stitching | Medium | Highest | Production systems, accurate attribution |
| User ID Propagation | Low | Medium | Post-login journey linking |
| Group Analytics | Medium | High | Advanced cross-device analysis |

**Recommendation:** Combine **Shared Flow ID** + **Server-Side Stitching** for the most robust solution.

---

## Implementation Guide

### Solution 1: Shared Flow ID (Recommended)

Generate a unique identifier when the QR code is created and pass it through the entire flow.

#### Step 1: TV Generates QR Code

```javascript
// Generate unique flow ID (can be the QR token itself)
const qrLoginFlowId = crypto.randomUUID(); // e.g., "a1b2c3d4-e5f6-7890-abcd-ef1234567890"

// Store TV's device ID to pass through the flow
const tvDeviceId = amplitude.getDeviceId();

// Track QR code display
amplitude.track('QR Code Displayed', {
  qr_login_flow_id: qrLoginFlowId,
  device_type: 'tv',
  tv_device_id: tvDeviceId,
  tv_model: getTVModel(), // Optional: Samsung, LG, etc.
  app_version: getAppVersion()
});

// QR code URL includes the flow ID
const qrCodeUrl = `https://www.canalplus.com/tv-login?flow_id=${qrLoginFlowId}&tv_device=${tvDeviceId}`;
displayQRCode(qrCodeUrl);
```

#### Step 2: Phone Scans QR Code

```javascript
// Extract flow ID and TV device ID from QR code URL
const urlParams = new URLSearchParams(window.location.search);
const qrLoginFlowId = urlParams.get('flow_id');
const tvDeviceId = urlParams.get('tv_device');

// Store for use throughout the login flow
sessionStorage.setItem('qr_login_flow_id', qrLoginFlowId);
sessionStorage.setItem('tv_device_id', tvDeviceId);

// Track QR scan
amplitude.track('QR Code Scanned', {
  qr_login_flow_id: qrLoginFlowId,
  device_type: 'mobile',
  tv_device_id: tvDeviceId,
  scan_method: 'camera' // or 'in_app_scanner'
});
```

#### Step 3: Phone Completes Login

```javascript
// Retrieve stored flow ID
const qrLoginFlowId = sessionStorage.getItem('qr_login_flow_id');
const tvDeviceId = sessionStorage.getItem('tv_device_id');

// Track login steps - ALWAYS include flow_id
amplitude.track('Login Started', {
  qr_login_flow_id: qrLoginFlowId,
  device_type: 'mobile',
  login_method: 'qr_code'
});

// After successful authentication
amplitude.track('Login Completed', {
  qr_login_flow_id: qrLoginFlowId,
  device_type: 'mobile',
  tv_device_id: tvDeviceId,
  login_method: 'qr_code',
  time_to_complete_seconds: calculateDuration()
});

// Set user ID on phone
amplitude.setUserId(hashedUserId);
```

#### Step 4: TV Receives Authentication Success

```javascript
// TV receives callback from server with user info
function onTVAuthSuccess(authData) {
  const qrLoginFlowId = authData.flow_id;
  const userId = authData.user_id;
  
  // Set user ID on TV
  amplitude.setUserId(userId);
  
  // Track TV activation
  amplitude.track('TV Activated', {
    qr_login_flow_id: qrLoginFlowId,
    device_type: 'tv',
    activation_method: 'qr_code'
  });
}
```

---

### Solution 2: Server-Side Event Stitching

For highest reliability, have your backend send the critical conversion event with proper attribution.

#### Server-Side Implementation (Node.js)

```javascript
const { Amplitude } = require('@amplitude/analytics-node');

// Initialize server-side Amplitude
Amplitude.init(AMPLITUDE_API_KEY);

async function handleQRLoginComplete(loginData) {
  const { userId, flowId, tvDeviceId, phoneDeviceId, loginDuration } = loginData;
  
  // Option A: Attribute conversion to TV (recommended for streaming services)
  // This credits the TV device with the successful login
  await Amplitude.track(
    'QR Login Completed',
    {
      qr_login_flow_id: flowId,
      authenticated_via_device: 'mobile',
      phone_device_id: phoneDeviceId,
      login_duration_seconds: loginDuration,
      login_method: 'qr_code'
    },
    {
      user_id: userId,
      device_id: tvDeviceId  // Attribute to TV!
    }
  );
  
  // Option B: Send to both devices for complete tracking
  // TV event
  await Amplitude.track(
    'Login Completed',
    {
      qr_login_flow_id: flowId,
      device_type: 'tv',
      login_method: 'qr_code'
    },
    {
      user_id: userId,
      device_id: tvDeviceId
    }
  );
  
  // Phone event
  await Amplitude.track(
    'Login Completed',
    {
      qr_login_flow_id: flowId,
      device_type: 'mobile',
      login_method: 'qr_code'
    },
    {
      user_id: userId,
      device_id: phoneDeviceId
    }
  );
  
  await Amplitude.flush();
}
```

#### Server-Side Implementation (Python)

```python
from amplitude import Amplitude, BaseEvent

amplitude = Amplitude(AMPLITUDE_API_KEY)

def handle_qr_login_complete(login_data):
    user_id = login_data['user_id']
    flow_id = login_data['flow_id']
    tv_device_id = login_data['tv_device_id']
    phone_device_id = login_data['phone_device_id']
    
    # Attribute to TV device
    amplitude.track(
        BaseEvent(
            event_type='QR Login Completed',
            user_id=user_id,
            device_id=tv_device_id,
            event_properties={
                'qr_login_flow_id': flow_id,
                'authenticated_via_device': 'mobile',
                'phone_device_id': phone_device_id,
                'login_method': 'qr_code'
            }
        )
    )
    
    amplitude.flush()
```

---

### Solution 3: User ID Propagation

After successful login, ensure both devices are linked via User ID.

```javascript
// ON PHONE - After successful login
const userId = hashEmail(authenticatedUser.email);
amplitude.setUserId(userId);

// Your backend tells the TV that login succeeded
// Include the user_id in that callback

// ON TV - When receiving login success
function onLoginSuccess(data) {
  // Set the same User ID
  amplitude.setUserId(data.user_id);
  
  // From this point forward, both devices are linked
  amplitude.track('Content Playback Started', {
    content_id: 'movie_123',
    login_method: 'qr_code'
  });
}
```

> **Note:** This links future events but does NOT retroactively link pre-login events. Use with Solution 1 for complete coverage.

---

### Solution 4: Group Analytics

Use Amplitude's Group Analytics to analyze at the session/flow level rather than user level.

```javascript
// ON TV
amplitude.setGroup('qr_login_session', flowId);
amplitude.track('QR Code Displayed');

// ON PHONE
amplitude.setGroup('qr_login_session', flowId);
amplitude.track('QR Code Scanned');
amplitude.track('Login Completed');
```

**In Amplitude:**
1. Navigate to your funnel chart
2. Change "Group by" from "User" to "qr_login_session"
3. Analyze conversion at the session level

---

## Recommended Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     QR CODE LOGIN FLOW ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐    │
│  │      TV      │         │    SERVER    │         │    PHONE     │    │
│  │   App/Web    │         │   Backend    │         │   App/Web    │    │
│  └──────┬───────┘         └──────┬───────┘         └──────┬───────┘    │
│         │                        │                        │             │
│         │  1. Request QR Code    │                        │             │
│         │ ─────────────────────► │                        │             │
│         │                        │                        │             │
│         │  2. Return QR Code +   │                        │             │
│         │     flow_id +          │                        │             │
│         │     tv_device_id       │                        │             │
│         │ ◄───────────────────── │                        │             │
│         │                        │                        │             │
│         │  3. Track:             │                        │             │
│         │     "QR Code           │                        │             │
│         │      Displayed"        │                        │             │
│         │     {flow_id,          │                        │             │
│         │      tv_device_id}     │                        │             │
│         │ ───────────────────────────────────────────────────► AMPLITUDE│
│         │                        │                        │             │
│         │                        │                        │             │
│         │        ┌───────────────────────────┐            │             │
│         │        │  USER SCANS QR WITH PHONE │            │             │
│         │        └───────────────────────────┘            │             │
│         │                        │                        │             │
│         │                        │  4. Phone extracts     │             │
│         │                        │     flow_id from QR    │             │
│         │                        │ ◄───────────────────── │             │
│         │                        │                        │             │
│         │                        │  5. Track:             │             │
│         │                        │     "QR Code Scanned"  │             │
│         │                        │     {flow_id}          │             │
│         │                        │ ───────────────────────────► AMPLITUDE│
│         │                        │                        │             │
│         │                        │  6. User authenticates │             │
│         │                        │ ◄───────────────────── │             │
│         │                        │                        │             │
│         │                        │  7. Track:             │             │
│         │                        │     "Login Completed"  │             │
│         │                        │     {flow_id,          │             │
│         │                        │      phone_device_id}  │             │
│         │                        │ ───────────────────────────► AMPLITUDE│
│         │                        │                        │             │
│         │                        │                        │             │
│    ┌────┴────────────────────────┴────────────────────────┴────┐       │
│    │              8. SERVER-SIDE EVENT (CRITICAL)               │       │
│    │                                                            │       │
│    │   Server tracks "QR Login Completed" attributed to         │       │
│    │   TV's device_id with user_id, linking the journey         │       │
│    │                                                            │       │
│    └────┬────────────────────────┬────────────────────────┬────┘       │
│         │                        │                        │             │
│         │                        │ ───────────────────────────► AMPLITUDE│
│         │                        │                        │             │
│         │  9. Auth success +     │                        │             │
│         │     user_id            │                        │             │
│         │ ◄───────────────────── │                        │             │
│         │                        │                        │             │
│         │  10. setUserId()       │                        │             │
│         │      on TV             │                        │             │
│         │ ───────────────────────────────────────────────────► AMPLITUDE│
│         │                        │                        │             │
│         │  11. Track:            │                        │             │
│         │      "TV Activated"    │                        │             │
│         │      {user_id,         │                        │             │
│         │       flow_id}         │                        │             │
│         │ ───────────────────────────────────────────────────► AMPLITUDE│
│         │                        │                        │             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Event Taxonomy

### Recommended Events for QR Login Flow

| Event Name | Triggered By | Required Properties | Optional Properties |
|------------|--------------|---------------------|---------------------|
| `QR Code Displayed` | TV | `qr_login_flow_id`, `device_type: "tv"`, `tv_device_id` | `tv_model`, `app_version` |
| `QR Code Scanned` | Phone | `qr_login_flow_id`, `device_type: "mobile"`, `tv_device_id` | `scan_method` |
| `Login Started` | Phone | `qr_login_flow_id`, `login_method: "qr_code"` | `account_type` |
| `Login Completed` | Phone/Server | `qr_login_flow_id`, `login_method: "qr_code"` | `time_to_complete_seconds` |
| `QR Login Completed` | Server | `qr_login_flow_id`, `authenticated_via_device`, `tv_device_id`, `phone_device_id` | `login_duration_seconds` |
| `TV Activated` | TV | `qr_login_flow_id`, `activation_method: "qr_code"` | `tv_model` |

### Property Definitions

| Property | Type | Description |
|----------|------|-------------|
| `qr_login_flow_id` | String (UUID) | Unique identifier for the entire QR login flow. **Critical for stitching.** |
| `device_type` | String | `"tv"` or `"mobile"` |
| `tv_device_id` | String | The Amplitude Device ID of the TV. Passed through QR code. |
| `phone_device_id` | String | The Amplitude Device ID of the phone. |
| `login_method` | String | `"qr_code"`, `"email"`, `"social"`, etc. |
| `time_to_complete_seconds` | Number | Duration from QR scan to login completion |

---

## Funnel Analysis Setup

### Building the Cross-Device Funnel

1. **Navigate to:** Amplitude → Analytics → Funnel Analysis

2. **Create Funnel Steps:**
   - Step 1: `QR Code Displayed`
   - Step 2: `QR Code Scanned`
   - Step 3: `Login Completed` (or `QR Login Completed` from server)
   - Step 4: `TV Activated`

3. **Configure Grouping:**
   - Click "Group by"
   - Select `qr_login_flow_id`
   - This groups events by the shared flow ID instead of Device ID

4. **Set Conversion Window:**
   - Recommended: 15-30 minutes (typical QR login flow duration)

### Example Funnel Query

```
Funnel: QR Code Login
├── Step 1: QR Code Displayed (where device_type = "tv")
├── Step 2: QR Code Scanned (where device_type = "mobile")
├── Step 3: Login Completed (where login_method = "qr_code")
└── Step 4: TV Activated

Group by: qr_login_flow_id
Conversion window: 30 minutes
```

### Segmentation Options

- **By TV Model:** See which TV brands have higher/lower conversion
- **By App Version:** Identify version-specific issues
- **By Time of Day:** Understand usage patterns
- **By Geography:** Regional conversion differences

---

## Best Practices

### ✅ Do

1. **Always include `qr_login_flow_id`** in every event related to the QR login flow
2. **Use server-side tracking** for the critical conversion event
3. **Set User ID on both devices** after successful authentication
4. **Pass TV Device ID through the QR code** so the phone knows which device initiated
5. **Use consistent event naming** across platforms
6. **Implement timeout handling** - track `QR Code Expired` if the user doesn't complete in time

### ❌ Don't

1. **Don't rely solely on client-side tracking** - server-side is more reliable
2. **Don't generate new Device IDs** - use Amplitude's auto-generated ones
3. **Don't skip the flow_id** - it's your only way to stitch cross-device journeys
4. **Don't track PII in flow_id** - use UUIDs or opaque tokens
5. **Don't forget error states** - track `QR Login Failed` with error reasons

### Error Handling Events

```javascript
// Track failures for debugging
amplitude.track('QR Login Failed', {
  qr_login_flow_id: flowId,
  error_type: 'timeout', // 'invalid_code', 'auth_failed', 'network_error'
  error_message: 'QR code expired after 5 minutes',
  device_type: 'mobile'
});
```

---

## FAQ

### Q: Why can't Amplitude automatically link these devices?

**A:** Amplitude identifies users primarily by Device ID (generated per device) and User ID (set after authentication). Before login, there's no shared identifier between the TV and phone - they're just two anonymous devices. The `qr_login_flow_id` creates this shared link.

### Q: Should I attribute the conversion to the TV or the phone?

**A:** For streaming services, **attribute to the TV** since that's where content consumption happens. The phone is just an authentication mechanism. Use server-side tracking to send the conversion event with the TV's device_id.

### Q: What if the user abandons the flow and tries again later?

**A:** Each QR code generation creates a new `flow_id`. Track `QR Code Expired` or `QR Login Abandoned` when the previous flow times out. This gives you accurate funnel metrics.

### Q: How long should the QR code be valid?

**A:** Typically 5-10 minutes. Track the `time_to_complete_seconds` property to understand actual user behavior and adjust accordingly.

### Q: Can I use this pattern for other cross-device flows?

**A:** Yes! This pattern works for any cross-device authentication:
- Smart TV login via mobile
- Desktop-to-mobile app linking
- IoT device setup
- Two-factor authentication flows

### Q: How do I debug if the funnel still looks broken?

**A:** 
1. Check that `qr_login_flow_id` is present in all events (User Lookup → find a test user → verify properties)
2. Ensure the flow_id values match exactly across devices (case-sensitive)
3. Verify the funnel is grouped by `qr_login_flow_id`, not User or Device
4. Check conversion window is long enough for your flow

---

## Support

For implementation questions or assistance, contact your Amplitude Solutions Engineer.

---

*This guide is designed for streaming services implementing QR code login flows but the patterns apply to any cross-device authentication scenario.*

