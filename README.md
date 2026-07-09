# Airframe

**Airframe is an open architecture for moving real-time media between devices, independent of how it is captured or transported.**

A wireless camera system that decomposes media pipelines into independent layers, enabling reliable, transparent, and flexible video streaming from mobile devices to production environments.

## Why Airframe Exists

Most media workflows today are built as vertical stacks. A camera application captures video, encodes it, sends it over a specific protocol, and delivers it to a specific receiver. The capture logic, the transport logic, and the consumption logic are welded together.

This works until you need to change something.

Consider a common setup. A church livestream team uses a phone as a camera source. The phone runs DroidCam. DroidCam captures the camera feed, encodes it, transports it over the local network using a proprietary protocol, and exposes it as a virtual webcam on a Windows machine running OBS.

```
Phone Camera
     │
     ▼
  DroidCam
  (capture + encode + transport)
     │
     ▼
  DroidCam Driver
  (receive + decode + virtual device)
     │
     ▼
    OBS
```

This chain works—until it doesn't.

The Wi-Fi drops. There is no automatic recovery. The connection dies silently, and the operator discovers it when the congregation notices the stream is frozen.

The team wants to add a second camera. DroidCam has no concept of multi-device session management. Each phone is an independent, uncoordinated connection.

The team wants to see diagnostics—latency, bitrate, packet loss. The tool doesn't expose this information because it was designed to be simple, not transparent.

The team wants to switch from Wi-Fi to USB for reliability. This is a different DroidCam mode with different behavior, different limitations, and no way to fall back to Wi-Fi if USB disconnects.

Every one of these problems exists because three distinct concerns—capturing media, transporting media, and consuming media—are treated as a single concern. They share code, share state, and share failure modes. Changing the transport means changing the capture pipeline. Changing the receiver means changing the transport assumptions.

**Airframe exists to decompose this.**

## What Airframe Is

Airframe is a layered architecture that separates concerns at the structural level:

```
Reality
    │
    ▼
Capture Layer
(Camera · Screen · Sensor)
    │
    ▼
Airframe Core
(Session · Routing · Timing)
    │
    ▼
Transport Layer
(WebRTC · USB · Bluetooth · Future)
    │
    ▼
Receiver Layer
    │
    ▼
Consumer
(OBS · Preview · Recording)
```

Each layer is independent. Each layer has a single responsibility. Each layer communicates with its neighbors through a defined contract. Replace the transport—the capture layer does not notice. Replace the capture source—the transport layer does not care.

This is not a novel architectural idea. It is how networking stacks, operating systems, and compilers have been designed for decades. But it has not been applied to real-time media transport between mobile devices and desktop production environments. Airframe applies it.

## Design Philosophy

Airframe is built on one structural idea and five invariants that follow from it.

### The Structural Idea

Every responsibility deserves its own layer. A layer should know as little as possible about the layers above and below it.

This is not minimalism for its own sake. It is a direct response to the coupling problem. When a camera application handles capture, encoding, transport, session management, and receiver registration in a single codebase, every change touches everything. Bugs propagate across boundaries that shouldn't exist. Testing requires the full stack. Deployment is all-or-nothing.

Airframe draws hard boundaries between responsibilities. Each boundary is defined by a contract—an interface that specifies inputs, outputs, and behavioral guarantees without specifying implementation.

### Design Invariants

These are the laws of Airframe. They constrain every decision—architectural, product, and engineering.

**Invariant 1. Airframe is transport-agnostic.**

Airframe never assumes how data moves between endpoints. The initial implementation uses WebRTC. Future implementations may use raw UDP, USB, Bluetooth, or technologies that do not yet exist. The architecture does not privilege any transport.

**Invariant 2. Airframe is capture-agnostic.**

Airframe never assumes where media originates. The initial implementation captures from an Android device camera. Future implementations may capture from screen recording, depth sensors, thermal sensors, LiDAR, or sources that do not exist yet.

**Invariant 3. Airframe coordinates; it does not create media.**

Airframe does not generate video. It does not generate audio. It does not apply filters, effects, or transformations to media content. It receives media from the Capture Layer, routes it through the Transport Layer, delivers it to the Receiver Layer, and provides session management, diagnostics, and reliability guarantees along the way.

**Invariant 4. Layers communicate through contracts, not implementations.**

Every layer depends on interfaces. Never on concrete types, never on implementation classes, never on internal state of another layer. The contract defines what a layer provides and what it requires. The implementation fulfills the contract.

**Invariant 5. Every component has a single responsibility.**

The Session Manager manages sessions. The Transport sends data. The Capture Layer captures frames. The Receiver receives them. No component does two things.

## Product Principles

Design invariants constrain the architecture. Product principles constrain the experience.

### Reliability Is the Feature

Airframe's first users are church livestream operators. They do not have a backup camera crew. They do not have a network engineer on staff. They have one phone, one laptop, one Sunday morning, and a congregation watching online. If the stream drops, there is no second take.

The most important feature Airframe can offer is not resolution, not frame rate, not low latency. It is reliability. The stream should not drop. If it does drop, it should recover automatically. If it cannot recover, it should tell the operator exactly what happened and what to do about it.

### Transparency Over Simplicity

Many tools hide complexity by hiding information. Airframe hides complexity by presenting information clearly.

The operator should always be able to answer three questions without navigating away from the primary screen:

1. Is the stream live?
2. Is the connection healthy?
3. If something is wrong, what is it?

This means real-time diagnostics are not an advanced feature. They are a primary interface element. Latency, bitrate, packet loss, and signal quality are always visible.

### Respect the Operator's Time

A livestream setup happens under time pressure. The service starts at a fixed time. The operator is often a volunteer, not a technician. Setup should take seconds, not minutes.

Device discovery should be automatic. Pairing should require one confirmation, not a configuration sequence. Reasonable defaults should cover 90% of use cases.

## Implementation Stages

Airframe is being implemented in stages, each building on the previous one.

### Stage 1: MVP (Current)

**Goal:** Prove the layered architecture with a working end-to-end system.

**Components:**
- **Capture App (Android):** React Native + Expo application that captures camera feed and streams via WebRTC
- **Airframe Dashboard (Desktop):** Wails-based receiver application with WebRTC signaling and diagnostics
- **Signaling Server:** Go-based WebSocket server for WebRTC peer connection establishment
- **Network Discovery:** mDNS/Bonjour for automatic receiver discovery

**Features:**
- 1080p @ 60fps video streaming
- Real-time latency, bitrate, and FPS monitoring
- Automatic device discovery on local network
- Connection quality indicators
- Error recovery and diagnostics

**Technology Stack:**
- **Capture App:** React Native 0.86.0, Expo ~57.0.4, react-native-vision-camera, react-native-webrtc
- **Dashboard:** Wails v2, React 19.1.0, TypeScript, Tailwind CSS v4
- **Signaling:** Go 1.22+, WebSocket, mDNS

### Stage 2: v0.2

**Goal:** Add multi-device support and improve reliability.

**Components:**
- Multi-device session management
- Coordinated timing across multiple cameras
- Centralized monitoring dashboard
- Enhanced error recovery

**Features:**
- Support for 3+ simultaneous camera sources
- Shared session state across devices
- Camera switching with smooth transitions
- Advanced diagnostics and logging
- USB transport fallback

### Stage 3: v0.5

**Goal:** Expand capture sources and transport options.

**Components:**
- Screen capture support
- USB transport implementation
- Bluetooth transport for low-latency control
- Plugin architecture for extensibility

**Features:**
- Screen recording as capture source
- USB tethering for maximum reliability
- Bluetooth-based camera control
- Plugin SDK for custom capture sources
- Recording capabilities

### Stage 4: v1.0

**Goal:** Production-ready platform with full feature set.

**Components:**
- Complete transport layer (WebRTC, USB, Bluetooth, raw UDP)
- Complete capture layer (camera, screen, sensors)
- Production-grade reliability and monitoring
- Comprehensive documentation and tooling

**Features:**
- All transport options fully implemented
- All capture sources supported
- Enterprise-grade reliability guarantees
- Full API documentation
- Developer SDK and tooling

### Stage 5: Airframe Platform

**Goal:** Transform Airframe from a tool into a platform.

**Components:**
- Cloud-based signaling and coordination
- Multi-location support
- Advanced analytics and monitoring
- Integration with production tools

**Features:**
- Cloud signaling for remote connections
- Geographic load balancing
- Advanced stream analytics
- Integration with OBS, vMix, and other production tools
- API for third-party integrations

### Stage 6: Plugin SDK

**Goal:** Enable third-party extensions.

**Components:**
- Plugin SDK for custom capture sources
- Plugin SDK for custom transports
- Plugin marketplace
- Plugin development tools

**Features:**
- Extensible capture source API
- Extensible transport API
- Plugin distribution system
- Plugin development documentation and examples

## Current Status

**Stage 1 (MVP) is in active development.**

- ✅ Capture App: Functional with WebRTC streaming and mDNS discovery
- ✅ Dashboard: Functional with WebRTC signaling and real-time metrics
- ✅ Signaling Server: Implemented with mDNS broadcasting
- ✅ Network Discovery: mDNS/Bonjour integration
- 🔄 Multi-device support: Planned for v0.2
- 🔄 USB transport: Planned for v0.5

## Getting Started

### Prerequisites

- Node.js 18 or higher
- Go 1.22 or higher
- Expo CLI (for mobile development)
- iOS: Xcode 15+ (for iOS development)
- Android: Android Studio with SDK 34+ (for Android development)

### Installation

**Desktop Dashboard:**
```bash
cd airframe-dashboard
npm install
wails dev
```

**Mobile Capture App:**
```bash
cd capture-app
npm install
npm start
```

### Building

**Desktop:**
```bash
cd airframe-dashboard
wails build
```

**Android:**
```bash
cd capture-app
eas build --platform android
```

**iOS:**
```bash
cd capture-app
eas build --platform ios
```

## Architecture Documentation

For detailed architecture documentation, see:
- [ARCHITECTURE.md](ARCHITECTURE.md) - Complete system architecture with mermaid diagrams
- [DESIGN_BOOK.md](DESIGN_BOOK.md) - Comprehensive design specification
- [capture-app/README.md](capture-app/README.md) - Mobile capture app documentation
- [airframe-dashboard/README.md](airframe-dashboard/README.md) - Desktop dashboard documentation

## Project Structure

```
Airframe/
├── airframe-dashboard/          # Desktop receiver application
│   ├── frontend/                # React frontend
│   ├── main.go                  # Wails entry point
│   └── signaling.go             # WebSocket signaling server
├── capture-app/                 # Mobile capture application
│   ├── screens/                 # React Native screens
│   ├── App.tsx                  # Main app with WebRTC logic
│   └── MOBILE_APP_DESIGN.md     # Mobile design specification
├── capture-app-old/            # Original capture app (preserved)
├── airframe-dashboard-old/      # Original dashboard (preserved)
├── ARCHITECTURE.md              # System architecture documentation
├── DESIGN_BOOK.md              # Complete design specification
└── README.md                    # This file
```

## Contributing

Airframe is a living project. The Design Book is updated as the architecture evolves. Contributions should align with the design invariants and product principles.

Before contributing:
1. Read the [DESIGN_BOOK.md](DESIGN_BOOK.md)
2. Review the [ARCHITECTURE.md](ARCHITECTURE.md)
3. Ensure changes respect the five design invariants
4. Test changes across layer boundaries

## License

See LICENSE file for details.

## The Name

The name originated from the problem that started the project: moving video frames from a phone to a laptop over Wi-Fi. Frames, traveling through the air. Air-frame.

But the name turned out to carry a deeper meaning. In aviation, the airframe is the mechanical structure of an aircraft—the fuselage, wings, and structural components that everything else attaches to. It is not the engine. It is not the avionics. It is not the payload. It is the structure that holds everything together and makes flight possible.

Airframe, the software platform, occupies the same role. It is not the camera (engine). It is not the UI (avionics). It is not the video content (payload). It is the structure that connects capture to transport to consumption and makes the media pipeline work.

The name describes what Airframe *is*, not what it currently *does*.
