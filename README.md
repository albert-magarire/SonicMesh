# SonicMesh 📡🔊

**Multi-modal offline mesh network for smartphones.**  
Operates entirely without internet — no servers, no cloud, no infrastructure.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        React Native UI                          │
│   ChatScreen  │  NetworkScreen  │  CameraScreen  │  Settings   │
└───────────────────────────┬─────────────────────────────────────┘
                            │ observe()
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      WatermelonDB (SQLite)                      │
│   messages  │  known_peers  │  seen_ids                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │ read/write
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      RoutingEngine.js                           │
│   Gossip Protocol  │  TTL Decrement  │  UUID Dedup  │  Relay   │
└────────────┬──────────────────────────────────┬────────────────┘
             │                                  │
             ▼                                  ▼
┌────────────────────────┐          ┌───────────────────────────┐
│     NearbyService      │          │       SonicService        │
│  Google Nearby API     │          │  ggwave 18–20 kHz audio   │
│  BT + WiFi-Direct      │          │  Ultrasonic fallback      │
│  P2P_CLUSTER topology  │          │  Text payloads only       │
└────────────────────────┘          └───────────────────────────┘
```

## Gossip Protocol

Every message is formatted as a compact pipe-delimited string:

```
UUID | TTL | SENDER_ID | TYPE | PAYLOAD
```

**Example:**
```
550e8400-e29b-41d4-a716-446655440000|5|NODE-A3F2|text|Need medevac at sector 4
```

**TTL flow:**
- Originating node sends with TTL = 5 (SOS = 7)
- Each relay hop decrements TTL by 1
- At TTL = 0, message is saved but not forwarded
- UUID deduplication prevents message loops

## Transport Decision Logic

```
send()
  ├─ RF peers available?
  │    YES → broadcast via Nearby Connections (all peers)
  │    NO  → type == image? → queue (wait for RF)
  │          type == text?  → emit via ggwave ultrasonic
  └─ mark seenId, update DB status
```

## Tech Stack

| Layer | Library |
|-------|---------|
| UI | React Native 0.73 |
| Database | WatermelonDB (SQLite + JSI) |
| RF Mesh | react-native-google-nearby-connection |
| Sonic | react-native-ggwave |
| Camera | react-native-vision-camera |
| Background | react-native-background-actions |
| Image compression | react-native-image-resizer |
| Permissions | react-native-permissions |

## Setup

### Prerequisites

- Node.js 18+
- React Native CLI (not Expo)
- Xcode 14+ (iOS)
- Android Studio + NDK (Android)

### Installation

```bash
git clone <repo>
cd SonicMesh
npm install

# iOS
cd ios && pod install && cd ..
npx react-native run-ios

# Android
npx react-native run-android
```

### Critical: Native Module Linking

WatermelonDB requires JSI. Ensure in `android/app/build.gradle`:
```gradle
android {
  defaultConfig {
    externalNativeBuild {
      cmake { cppFlags "-std=c++17" }
    }
  }
}
```

### Permissions

The app requests all permissions at launch. On Android, ensure you accept:
- Nearby Devices (Bluetooth + WiFi)
- Location (required by Nearby API to scan)
- Microphone (ultrasonic RX)
- Camera

## Key Design Decisions

- **WatermelonDB over AsyncStorage:** SQLite with JSI gives ~10x faster reactive queries; critical for real-time mesh state.
- **P2P_CLUSTER over P2P_STAR:** Cluster strategy allows M-to-N topology. Star would require a central host node, defeating the "no single point of failure" property.
- **UUID deduplication with dedicated table:** Rather than scanning the full messages table on each receive (slow), a lean `seen_ids` table provides O(log n) indexed lookups.
- **TTL starts at 5 (SOS=7):** Enough to traverse a 5-hop network without flooding. SOS gets extra priority hops.
- **Base64 images under 25KB:** Avoids the complexity of native `Payload.fromFile()` stream bridging. For a hackathon demo, 400×400 at 40% quality compresses to ~18KB — perfectly adequate.
- **Foreground service:** Android will aggressively kill background processes. The persistent notification is non-negotiable for radio continuity.
