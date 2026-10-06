# 4. Android App Diagrams

| Field | Value |
|---|---|
| Document ID | GDN-DIA-004 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |

The app is a native Kotlin app on the SIYI MK15 ground unit. Design text: [ground-app.md](../10-communication/ground-app.md). The app has not been built; these diagrams are the design.

## D4.1 The app in one simple picture

```mermaid
flowchart LR
    OP(["Operator"]) --> APP["GDN Ground app<br/>on the MK15 screen"]
    APP -- "requests: search, track, follow, stop" --> GW["app_gateway<br/>on the Raspberry Pi"]
    GW -- "video, status, findings, map tiles" --> APP
    GW <--> MIS["Mission software"]
    MIS --> FC["Flight controller"]
    PILOT(["Pilot"]) -- "sticks and switches, always in charge" --> FC
```

## D4.2 App architecture

```mermaid
flowchart TB
    subgraph UI["User interface - Jetpack Compose"]
        S1["Live screen"]
        S2["Map screen"]
        S3["Findings screen"]
        S4["Status screen"]
    end
    subgraph VM["ViewModels"]
        V1["LiveViewModel"]
        V2["MapViewModel"]
        V3["FindingsViewModel"]
        V4["StatusViewModel"]
    end
    subgraph DATA["Data layer"]
        REPO["DroneRepository: one source of truth for state"]
        WS["ControlClient: WebSocket, JSON"]
        VID["VideoClient: JPEG frames"]
        TILE["TileCache: offline map tiles"]
        DB[("Room database: findings, events")]
    end
    S1 --> V1
    S2 --> V2
    S3 --> V3
    S4 --> V4
    V1 --> REPO
    V2 --> REPO
    V3 --> REPO
    V4 --> REPO
    REPO --> WS
    REPO --> VID
    REPO --> TILE
    REPO --> DB
    WS <-- "port 8765" --> PI["Raspberry Pi: app_gateway"]
    VID <-- "port 8766" --> PI2["Raspberry Pi: video_streamer"]
    TILE <-- "port 8767, HTTP" --> PI3["Raspberry Pi: tile server"]
```

## D4.3 Screens and how the operator moves between them

```mermaid
stateDiagram-v2
    [*] --> Connecting
    Connecting --> Status: connected
    Connecting --> Connecting: retry every 2 s
    Status --> Live: tab
    Status --> Map: tab
    Status --> Findings: tab
    Live --> Map: tab
    Map --> Live: tab
    Map --> Findings: tab or tap a pin
    Findings --> Map: show on map
    Findings --> Live: tab
    Live --> Status: tab
    Live --> Connecting: link lost
    Map --> Connecting: link lost
    Findings --> Connecting: link lost
    Status --> Connecting: link lost
```

## D4.4 Live screen

```mermaid
flowchart TB
    subgraph LIVE["Live screen"]
        direction TB
        BAR["Top bar: navigation mode, confidence, time since last map fix, battery, height"]
        VIDEO["Video with detection boxes drawn by the app"]
        BTN["Buttons: Track, Follow, Stop"]
    end
    VF["Video frames with frame id"] --> VIDEO
    DETM["detections message for that frame id"] --> VIDEO
    STM["status message, 2 Hz"] --> BAR
    VIDEO -- "tap on an object" --> SEL["select_target: frame id + x, y"]
    BTN -- "Follow" --> FOL["follow_start"]
    BTN -- "Stop" --> HOLD["hold"]
    SEL --> PI["to app_gateway"]
    FOL --> PI
    HOLD --> PI
```

## D4.5 Map screen

```mermaid
flowchart TB
    subgraph MAPS["Map screen"]
        direction TB
        MAPV["Satellite map from the drone's own map pack, offline"]
        OVER["Overlays: drone position and heading, uncertainty circle, path, map edge, geofence"]
        SRCH["Search layer: drawn area, planned lines, covered area"]
        PINS["Finding pins"]
        CTRL["Controls: draw area, height, spacing, Start, Pause, Abort, Stop"]
    end
    TILES["Tiles from the Pi, cached"] --> MAPV
    STAT["status"] --> OVER
    SMSG["search progress"] --> SRCH
    FMSG["finding messages"] --> PINS
    CTRL -- "Start search" --> REQ["search_start: polygon, height, overlap"]
    REQ --> PI["to app_gateway"]
    PI -- "ack with plan, or nack with reason" --> SRCH
```

## D4.6 Findings screen

```mermaid
flowchart LR
    F["finding message: class, score, coordinates, thumbnail"] --> DB[("Local database")]
    DB --> LIST["List: thumbnail, class, confidence, coordinates, time"]
    LIST -- "Confirm / Reject" --> REV["finding_review"]
    LIST -- "Go there" --> GO["goto_finding"]
    LIST -- "Export" --> FILE["GeoJSON or CSV file"]
    LIST -- "Show on map" --> MAP["Map screen, centred on the pin"]
    REV --> PI["to app_gateway"]
    GO --> PI
```

## D4.7 Status screen and pre-flight check

```mermaid
flowchart TB
    RUN["Operator presses Run pre-flight check"] --> REQ["preflight_run"]
    REQ --> PI["app_gateway to safety_supervisor"]
    PI --> RES{"All checks pass?"}
    RES -- "yes" --> GO["GO, shown in green with the word GO"]
    RES -- "no" --> NOGO["NO-GO with the list of reasons"]
    INFO["Also shown: map pack name, date, licence; software versions; link quality"]
```

## D4.8 Connection state of the app

```mermaid
stateDiagram-v2
    [*] --> Disconnected
    Disconnected --> Connecting: app opened
    Connecting --> Handshake: socket open
    Handshake --> Connected: hello received, key accepted
    Handshake --> Disconnected: key refused or version mismatch
    Connected --> Stale: no ping for 3 s
    Stale --> Connected: ping received
    Stale --> Disconnected: no ping for 10 s
    Disconnected --> Connecting: retry after 2 s
    Connected --> Disconnected: socket closed
```

## D4.9 What happens to a request from the app

```mermaid
flowchart TB
    R["Request from the app"] --> A{"Valid message,<br/>correct key,<br/>within rate limit?"}
    A -- "no" --> N1["Dropped and counted"]
    A -- "yes" --> B{"Numbers in range?<br/>Area inside map and fence?"}
    B -- "no" --> N2["nack: reason shown to the operator"]
    B -- "yes" --> C{"Armed by the pilot?<br/>GUIDED selected by the pilot?<br/>Autonomy switch on?"}
    C -- "no" --> N2
    C -- "yes" --> D{"Safety NOMINAL?<br/>Navigation mode permits motion?"}
    D -- "no" --> N2
    D -- "yes" --> E["Pass to mission_manager"]
    E --> OK["ack"]
```

## D4.10 If the app link is lost

```mermaid
flowchart TB
    L["No heartbeat from the app for 3 s"] --> Q{"What is the drone doing?"}
    Q -- "Nothing or hold" --> A1["No change"]
    Q -- "Tracking" --> A2["Keep tracking"]
    Q -- "Following" --> A3["Continue 10 s, then hold"]
    Q -- "Grid search" --> A4["Continue; after 30 s finish the line and hold"]
    Q -- "Going to a finding" --> A5["Arrive, then hold"]
    A3 --> P["Pilot and QGroundControl are unaffected"]
    A4 --> P
    A5 --> P
```

## D4.11 Tap to track: timing between app and drone

```mermaid
sequenceDiagram
    participant OP as Operator
    participant APP as App
    participant GW as app_gateway
    participant TT as target_tracker
    participant VS as video_streamer
    VS->>APP: frame 812 (JPEG)
    GW->>APP: detections for frame 812
    APP->>APP: draw boxes on frame 812
    OP->>APP: tap on a person
    APP->>GW: select_target, frame 812, x 0.41, y 0.63
    GW->>TT: /tracking/select
    TT->>TT: find the detection under the tap in frame 812
    TT-->>GW: track id 7
    GW-->>APP: ack
    loop 5 Hz
        TT->>GW: /tracking/target
        GW->>APP: target message
        APP->>APP: highlight the target box
    end
```

## D4.12 Development stages for the app

```mermaid
flowchart LR
    S1["Stage 1<br/>App + mock gateway on a laptop"] --> S2["Stage 2<br/>App + real gateway + simulated drone"]
    S2 --> S3["Stage 3<br/>App on the MK15, Pi on the bench, real radio link"]
    S3 --> S4["Stage 4<br/>Flight: monitor only"]
    S4 --> S5["Stage 5<br/>Flight: search, then follow"]
```
