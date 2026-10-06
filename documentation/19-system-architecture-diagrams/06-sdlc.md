# 6. Software Development Life Cycle

| Field | Value |
|---|---|
| Document ID | GDN-DIA-006 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |

The project follows the seven-phase software development life cycle. This document shows the cycle, how each phase applies to this project, and where the project stands.

## D6.1 The seven phases

```mermaid
flowchart LR
    P1["Phase 1<br/>Planning"] --> P2["Phase 2<br/>Requirement analysis"]
    P2 --> P3["Phase 3<br/>Design"]
    P3 --> P4["Phase 4<br/>Development"]
    P4 --> P5["Phase 5<br/>Testing"]
    P5 --> P6["Phase 6<br/>Deployment"]
    P6 --> P7["Phase 7<br/>Maintenance and support"]
    P7 -- "feedback, next version" --> P1
    P5 -- "defects" --> P4
    P5 -- "design problem" --> P3
```

## D6.2 What each phase means in this project

| Phase | What we do | Main outputs | Where it is documented | Status |
|---|---|---|---|---|
| **1 Planning** | Set the goal, check feasibility, fix the scope, list risks, define the minimum result | Objective; scope; feasibility; risk list; delivery levels (Bronze / Silver / Gold) | [project-overview](../00-project-overview/project-overview.md), [roadmap](../16-development-roadmap/roadmap.md), [BOM](../15-bom/bill-of-materials.md) | Done |
| **2 Requirement analysis** | Write what the system must do and how well | Functional requirements FR-001 to FR-131; non-functional NFR-001 to NFR-092 | [system-requirements](../01-requirements/system-requirements.md) | Done (version 3.0) |
| **3 Design** | Architecture, interfaces, workflows, app screens | System, hardware, software, ROS 2, app and communication designs; 18 decision records; these diagrams | Sections 02 to 12 and 17; this folder | Done, pending review |
| **4 Development** | Build the ROS 2 packages, the Android app, configuration and tools | 22 ROS 2 packages; Android app; flight-controller parameters and script; map-pack tool | [package-structure](../05-ros2/package-structure.md) | **Not started** |
| **5 Testing** | Eight test levels, from unit tests to staged flights | Test reports; measured performance | [testing-strategy](../13-testing/testing-strategy.md), [performance](../14-performance/performance-requirements.md) | Not started |
| **6 Deployment** | Put the software on the Pi, the Pixhawk and the MK15; pre-flight check; demonstration | Pi image; parameter files; signed APK; demonstration | D6.6 below; [roadmap](../16-development-roadmap/roadmap.md) | Not started |
| **7 Maintenance and support** | Review every flight, fix, re-test, update documents | Flight logs; issue list; new versions; updated baseline | D6.7 below; [project-status](../project-status.md) | Not started |

## D6.3 Life-cycle model used: iterative, with gates

The project is not a single pass. The phases repeat in increments, and a gate must be passed before work that depends on it begins.

```mermaid
flowchart TB
    PLAN["Planning + requirements + design<br/>(done once, then revised by change notes)"] --> I1
    subgraph I1["Increment 1: map matching"]
        direction LR
        A1["Develop"] --> A2["Test on recordings"] --> A3{"Gate G2:<br/>works on our site?"}
    end
    subgraph I2["Increment 2: hand-over and simulation"]
        direction LR
        B1["Develop"] --> B2["Test in simulation"] --> B3{"Gate G3:<br/>works in simulation?"}
    end
    subgraph I3["Increment 3: app as monitor"]
        direction LR
        C1["Develop"] --> C2["Test on the MK15"] --> C3{"Gate G0 passed<br/>and app shows video?"}
    end
    subgraph I4["Increment 4: flight without GPS"]
        direction LR
        D1["Integrate"] --> D2["Bench, then flight"] --> D3{"Gates G4, G5"}
    end
    subgraph I5["Increment 5: grid search"]
        direction LR
        E1["Develop"] --> E2["Simulation, then flight"]
    end
    subgraph I6["Increment 6: track and follow"]
        direction LR
        F1["Develop"] --> F2["Simulation, then flight"]
    end
    I1 --> I2 --> I4 --> I5 --> I6
    PLAN --> I3 --> I4
    I6 --> REL["Final demonstration and report"]
```

Follow (increment 6) is the first to be dropped if time runs out.

## D6.4 Design and test levels paired (V-model view)

```mermaid
flowchart LR
    R["Requirements"] --- AT["Acceptance: flight demonstration, L8"]
    S["System design"] --- ST["System tests: simulation, hardware-in-loop, bench, L4 to L7"]
    A["Architecture and interfaces"] --- IT["Integration tests: ROS 2 graph, app + gateway, L3"]
    D["Detailed design of each node"] --- UT["Unit tests, L1; sensor tests, L2"]
    R --> S --> A --> D --> CODE["Code"]
    CODE --> UT --> IT --> ST --> AT
```

## D6.5 Phases mapped to the roadmap

```mermaid
flowchart TB
    subgraph PH1["1 Planning"]
        R1["P01 Research and design baseline"]
    end
    subgraph PH2["2 Requirement analysis"]
        R2["P01 Requirements, revised in DB-2.0 and DB-3.0"]
    end
    subgraph PH3["3 Design"]
        R3["P01 Architecture, ADR-001 to ADR-018, diagrams"]
    end
    subgraph PH4["4 Development"]
        R4["P02 Environment"]
        R5["P03, P05, P10 Simulation and logic"]
        R6["P04, P06, P06G, P07, P07G, P09 Sensors, map matching, AI"]
        R7["P18 Android app; P19 Search; P20 Track and follow"]
    end
    subgraph PH5["5 Testing"]
        R8["L1 to L3 with each package"]
        R9["P08 Hardware-in-loop; P12 Bench"]
        R10["P13 to P15 Flight stages"]
    end
    subgraph PH6["6 Deployment"]
        R11["P12 Vehicle integration"]
        R12["P17 Final demonstration"]
    end
    subgraph PH7["7 Maintenance"]
        R13["P16 Optimisation and evaluation"]
        R14["After each flight: review and fix"]
    end
    PH1 --> PH2 --> PH3 --> PH4 --> PH5 --> PH6 --> PH7
```

## D6.6 Deployment: how software reaches each device

```mermaid
flowchart LR
    REPO[("Git repository")] --> CI["Build + unit tests + simulation tests"]
    CI --> REL["Tagged release"]
    REL --> PIB["Raspberry Pi: build on the Pi, install, restart gdn.service"]
    REL --> FCP["Pixhawk: load parameter file, copy Lua watchdog script"]
    REL --> APK["MK15: install signed APK"]
    MAPS[("Map pack built with map_prepare")] --> PIB
    MODEL[("AI models exported to NCNN")] --> PIB
    PIB --> PRE["Pre-flight check: versions, parameters, map pack, calibration"]
    FCP --> PRE
    APK --> PRE
    PRE --> GO{"GO?"}
    GO -- "yes" --> FLY["Cleared for the test card"]
    GO -- "no" --> FIX["Fix and repeat"]
```

## D6.7 Maintenance: what happens after every test

```mermaid
flowchart TB
    T["Test or flight"] --> LOGS["Collect: rosbag, flight log, telemetry log, app log"]
    LOGS --> REV["Review against the test card"]
    REV --> Q{"Result?"}
    Q -- "pass" --> REC["Record measured values in the performance table"]
    Q -- "fail or anomaly" --> ISS["Open an issue with the run ID"]
    ISS --> CAUSE{"Cause?"}
    CAUSE -- "code defect" --> FIXC["Fix, add a regression test"]
    CAUSE -- "wrong parameter" --> FIXP["Change the parameter file, re-run simulation"]
    CAUSE -- "design problem" --> ADR["New decision record, update the design"]
    CAUSE -- "requirement wrong" --> REQ["Change the requirement, record it"]
    FIXC --> RET["Re-test at the lowest level that shows the fault"]
    FIXP --> RET
    ADR --> RET
    REQ --> RET
    RET --> T
    REC --> NEXT["Next test card"]
```

## D6.8 Schedule by life-cycle phase

```mermaid
gantt
    title Project schedule by SDLC phase (planning estimate, 42 weeks)
    dateFormat YYYY-MM-DD
    axisFormat %b
    section 1-3 Plan, requirements, design
    Planning, requirements, design      :done, p1, 2026-10-05, 3w
    section 4 Development
    Map matching on recordings          :d1, 2026-10-26, 9w
    Simulation and navigation logic     :d2, 2026-10-26, 17w
    Cameras, odometry, AI               :d3, 2026-10-26, 13w
    Android app                         :d4, 2026-10-26, 21w
    Search, track and follow            :d5, 2027-02-01, 15w
    section 5 Testing
    Unit and integration tests          :t1, 2026-10-26, 30w
    Bench and hardware-in-loop          :t2, 2027-01-04, 11w
    Flight testing                      :t3, 2027-03-22, 14w
    section 6 Deployment
    Vehicle build and integration       :b1, 2027-01-04, 11w
    Final demonstration                 :milestone, m1, 2027-07-19, 0d
    section 7 Maintenance
    Review and fix after each test      :s1, 2027-01-04, 28w
    Evaluation and report               :s2, 2027-06-28, 4w
```

Dates assume a start on 5 October 2026 and are planning estimates only.

## D6.9 Where the project is now

```mermaid
flowchart LR
    P1["1 Planning<br/>DONE"] --> P2["2 Requirements<br/>DONE"]
    P2 --> P3["3 Design<br/>DONE, in review"]
    P3 --> G{"Gate G1:<br/>design accepted?"}
    G --> P4["4 Development<br/>NOT STARTED"]
    P4 --> P5["5 Testing<br/>NOT STARTED"]
    P5 --> P6["6 Deployment<br/>NOT STARTED"]
    P6 --> P7["7 Maintenance<br/>NOT STARTED"]
```

Development must not start until gate G1 is passed.
