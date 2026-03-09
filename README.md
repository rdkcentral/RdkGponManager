# RdkGponManager Documentation

RdkGponManager is the RDK-B component responsible for managing GPON (Gigabit Passive Optical Network) ONT (Optical Network Terminal) physical layer interfaces and providing standardized TR-181 data model access to GPON hardware capabilities. This component serves as the management interface between RDK-B middleware and vendor-specific GPON hardware through a JSON HAL (Hardware Abstraction Layer) client-server architecture.

RdkGponManager monitors and controls GPON physical media modules, manages Virtual Ethernet Interface Points (VEIPs), tracks GEM (GPON Encapsulation Method) port configurations, monitors PLOAM (Physical Layer Operations Administration and Maintenance) registration states, and provides access to OMCI (ONT Management and Control Interface) statistics. The component implements the Device.X_RDK_ONT TR-181 data model namespace, enabling standardized access to GPON-specific parameters and operational data.

The component integrates with WAN Manager for unified WAN interface management in newer RDK-B releases, supporting both standalone GPON management and WAN Manager-unified deployment scenarios through conditional compilation. RdkGponManager maintains real-time awareness of GPON link states through an internal state machine that responds to hardware events and manages interface lifecycle transitions.

```mermaid
graph LR
    subgraph "External Systems"
        RemoteMgmt["Remote Management"]
        ExtServices["External Services"]
    end

    subgraph "RDK-B Platform"
        subgraph "RDK-B Middleware"
            WanMgr["WAN Manager"]
            PandM["CcspPandM"]
            WebPA["WebPA"]
        end

        subgraph "RdkGponManager"
            GponMgr["GPON Manager Controller"]
        end
    end

    subgraph "HAL & System Layer"
        JsonHAL["JSON HAL Client/Server"]
        GPONHw["GPON Hardware"]
    end

    RemoteMgmt -->|RBus| WebPA
    ExtServices -->|RBus| PandM
    WebPA -->|RBus| PandM
    PandM -->|RBus| GponMgr
    WanMgr -->|RBus| GponMgr
    GponMgr -->|JSON RPC| JsonHAL
    JsonHAL -->|HAL APIs| GPONHw

    classDef external fill:#fff3e0,stroke:#ef6c00,stroke-width:2px;
    classDef gponmgr fill:#e3f2fd,stroke:#1976d2,stroke-width:3px;
    classDef rdkbComponent fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px;
    classDef system fill:#fce4ec,stroke:#c2185b,stroke-width:2px;

    class RemoteMgr,ExtServices external;
    class GponMgr gponmgr;
    class WanMgr,PandM,WebPA rdkbComponent;
    class JsonHAL,GPONHw system;
```

**Key Features & Responsibilities**: 

- **GPON Physical Media Management**: Monitors and controls GPON optical transceiver modules including power levels, temperature, voltage, bias current, and alarm conditions for comprehensive hardware health tracking 
- **Virtual Ethernet Interface Point (VEIP) Control**: Manages VEIP lifecycle including administrative state control, operational status monitoring, and VLAN tagging configuration for upstream and downstream ethernet flows 
- **Link State Machine**: Implements intelligent GPON link state transitions responding to physical media alarms, VEIP operational states, and registration status to ensure reliable link establishment and failover handling 
- **GEM Port and Traffic Management**: Tracks GEM port configurations, traffic statistics, and VLAN flow parameters for both ingress and egress directions supporting multi-service traffic classification 
- **PLOAM and OMCI Statistics**: Provides access to PLOAM registration timers, activation counters, message counts, and OMCI baseline/extended message statistics for operational visibility and troubleshooting 
- **WAN Manager Integration**: Supports unified WAN interface management through conditional integration with WAN Manager, enabling coordinated multi-WAN scenarios with centralized interface lifecycle control 
- **Hardware Event Subscription**: Subscribes to vendor HAL events for real-time notification of physical media status changes, alarm conditions, VEIP state transitions, and PLOAM registration state updates


## Design

RdkGponManager follows a layered architecture separating TR-181 data model implementation, business logic control, and hardware abstraction through well-defined interfaces. The design emphasizes real-time event-driven state management with asynchronous HAL communication to minimize blocking operations during hardware queries. The component maintains a centralized data structure holding current GPON hardware state synchronized through periodic polling and event-driven updates from the vendor HAL layer.

The TR-181 middle layer implements Device.X_RDK_ONT object hierarchy functions for parameter get/set operations, table synchronization, validation, commit, and rollback operations following CCSP convention. The controller module orchestrates initialization, HAL event subscription, and state machine execution providing separation between data model interface and hardware control logic. The link state machine monitors GPON physical media and VEIP operational states to autonomously manage interface enable/disable operations based on link conditions and alarm states.

The northbound interface exposes TR-181 parameters through CCSP Data Model Agent following standard plugin architecture for integration with CcspPandM and other RDK-B components. The southbound interface abstracts vendor-specific GPON hardware control through JSON HAL client communicating with vendor HAL server over local RPC using standardized GPON HAL schema. During initialization, the component sends initial configuration parameters (power thresholds and VEIP flow settings) to the vendor HAL through `GponHal_send_config()`, though these configurations are not persisted across reboots. When WAN Manager unification is enabled, the component provides physical layer status updates to WAN Manager instead of directly managing ethernet gateway interfaces.

```mermaid
graph LR
    subgraph ExternalSystems ["External Systems"]
        TR181Client["TR-181 Clients"]
    end

    subgraph RdkGponManager ["RdkGponManager"]
        subgraph SSP ["Service Support Platform"]
            SSPMain["ssp_main"]
            MsgBus["ssp_messagebus_interface"]
        end

        subgraph Controller ["GPON Controller"]
            GponCtrl["gponmgr_controller"]
            LinkSM["gponmgr_link_state_machine"]
        end

        subgraph TR181Layer ["TR-181 Data Model Layer"]
            DMLPlugin["gponmgr_dml_plugin_main"]
            DMLFunc["gponmgr_dml_func"]
            DMLData["gponmgr_dml_data"]
        end

        subgraph HALInterface ["HAL Interface"]
            HALClient["gponmgr_dml_hal"]
        end
    end

    subgraph HALLayer ["JSON HAL Layer"]
        JsonServer["json_hal_server_gpon"]
    end

    subgraph SystemServices ["System Services"]
        Systemd["systemd"]
        GPONHw["GPON Hardware"]
    end

    TR181Client -->|RBus| MsgBus
    MsgBus --> DMLPlugin
    SSPMain --> MsgBus
    SSPMain --> GponCtrl
    DMLPlugin --> DMLFunc
    DMLFunc --> DMLData
    DMLFunc --> HALClient
    GponCtrl --> LinkSM
    LinkSM --> DMLData
    HALClient -->|JSON RPC| JsonServer
    JsonServer --> GPONHw
    SSPMain --> Systemd

    classDef external fill:#fff3e0,stroke:#ef6c00,stroke-width:2px;
    classDef gponmgr fill:#e3f2fd,stroke:#1976d2,stroke-width:3px;
    classDef hal fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px;
    classDef system fill:#fce4ec,stroke:#c2185b,stroke-width:2px;

    class TR181Client external;
    class SSP,Controller,TR181Layer,HALInterface gponmgr;
    class JsonServer hal;
    class Systemd,GPONHw system;
```

### Prerequisites and Dependencies

**Build-Time Flags and Configuration:**

| Configure Option | DISTRO Feature | Build Flag | Purpose | Default |
|------------------|----------------|------------|---------|---------|
| `--enable-notify` | N/A | `ENABLE_SD_NOTIFY` | Enable systemd service notification for ready state signaling | Disabled |
| N/A | `wan-manager-unification` | `WAN_MANAGER_UNIFICATION_ENABLED` | Enable integration with WAN Manager for unified WAN interface management | Disabled |
| N/A | N/A | `FEATURE_SUPPORT_RDKLOG` | Enable RDK centralized logging framework integration | Enabled |
| N/A | `breakpad` | `INCLUDE_BREAKPAD` | Enable Google Breakpad crash reporting and minidump generation | Disabled |

<br>

**RDK-B Platform and Integration Requirements:**

* **RDK-B Components**: `CcspPandM` for TR-181 data model registration, `CcspCommonLibrary` for base component infrastructure, `WAN Manager` when WAN_MANAGER_UNIFICATION_ENABLED is defined
* **HAL Dependencies**: JSON HAL client library (`libjson_hal_client`), vendor GPON HAL implementation providing json_hal_server_gpon binary and schema compliance
* **Systemd Services**: Component must start after network interfaces are initialized and CCSP Common Component infrastructure is active
* **Message Bus**: CCSP Message Bus registration under component namespace `com.cisco.spvtg.ccsp.gponmanager` for TR-181 parameter access
* **TR-181 Data Model**: Implements `Device.X_RDK_ONT` object hierarchy including PhysicalMedia, Veip, Gem, Gtc, Ploam, Omci, and TR69 sub-objects
* **Configuration Files**: `/etc/rdk/conf/gpon_manager_conf.json` for HAL client configuration (or `gpon_manager_wan_unify_conf.json` when WAN Manager unified), `/etc/rdk/conf/RdkGponManager.xml` for TR-181 object definitions, `/etc/rdk/schemas/gpon_hal_schema.json` for HAL schema validation
* **Startup Order**: Must initialize after CCSP infrastructure and before components requiring GPON interface status (e.g., WAN Manager, Networking components)
* **Resource Constraints**: Requires JSON-C library for HAL message parsing, pthread support for background state machine thread

<br>

**Threading Model** 

RdkGponManager implements a multi-threaded architecture with a main thread handling TR-181 requests and message bus operations, plus a dedicated background thread executing the GPON link state machine.

- **Threading Architecture**: Multi-threaded with main event loop and dedicated state machine thread
- **Main Thread**: Handles CCSP component lifecycle, TR-181 parameter get/set requests through DML layer, message bus communication, and HAL event callback processing
- **Worker Threads**: 
  - **State Machine Thread**: Executes GPON link state machine loop with 500ms interval monitoring physical media alarms, VEIP operational states, and managing link up/down transitions autonomously
- **Synchronization**: Uses mutex locks in DML data layer (`GponMgrDml_GetData_locked`, `GponMgrDml_GetData_release`) for thread-safe access to shared GPON data structures between TR-181 handlers and state machine thread

### Component State Flow

**Initialization to Active State**

RdkGponManager follows a structured initialization sequence starting from component loading through Data Model Agent plugin mechanism, progressing through HAL connection establishment, event subscription, and culminating in active state machine operation for continuous GPON link monitoring.

```mermaid
sequenceDiagram
    autonumber
    participant System as System Startup
    participant SSP as SSP Main
    participant MsgBus as Message Bus Interface
    participant DMLPlugin as DML Plugin Layer
    participant Controller as GPON Controller
    participant HAL as HAL Client
    participant StateMachine as Link State Machine

    System->>SSP: Start GponManager Process
    Note over SSP: State: Process Launch<br/>Parse command line, load config
    
    SSP->>MsgBus: Initialize Message Bus<br/>ssp_Mbi_MessageBusEngage()
    MsgBus-->>SSP: Bus Handle Acquired
    Note over MsgBus: State: Message Bus Connected<br/>Component namespace registered
    
    SSP->>DMLPlugin: Load Plugin<br/>GponMgrDml_Init()
    DMLPlugin->>DMLPlugin: Register Function Callbacks<br/>Get/Set/Validate/Commit
    DMLPlugin-->>SSP: Plugin Initialized
    Note over DMLPlugin: State: DML Registered<br/>TR-181 objects available
    
    SSP->>HAL: Initialize HAL Client<br/>GponHal_Init()
    HAL->>HAL: Start JSON HAL Server<br/>json_hal_server_gpon
    HAL->>HAL: Connect to HAL Server<br/>Retry up to 10 seconds
    HAL-->>SSP: HAL Connection Established
    Note over HAL: State: HAL Connected<br/>RPC channel active
    
    SSP->>Controller: Initialize Controller<br/>GponMgr_Controller_Init()
    Controller->>HAL: Subscribe HAL Events<br/>PhysicalMedia Status/Alarms<br/>Veip Admin/Oper State<br/>Ploam Registration State
    HAL-->>Controller: Subscriptions Confirmed
    Controller->>HAL: Send Initial Configuration<br/>GponHal_send_config()
    HAL-->>Controller: Configuration Applied
    Controller-->>SSP: Controller Ready
    Note over Controller: State: Events Subscribed<br/>Monitoring hardware
    
    SSP->>StateMachine: Start State Machine Thread<br/>gpon_controller_init()
    StateMachine->>StateMachine: Initialize SM Control Structure<br/>Set state GSM_LINK_DOWN
    StateMachine->>StateMachine: Enter State Machine Loop<br/>500ms interval
    StateMachine-->>SSP: State Machine Running
    Note over StateMachine: State: Active Monitoring<br/>Link state management active
    
    SSP->>System: Signal Ready<br/>sd_notify(READY=1) if enabled
    System-->>SSP: Acknowledgment
    Note over System: State: Component Active<br/>Ready for TR-181 requests
```

**Runtime State Changes and Context Switching**

During operation, RdkGponManager responds to hardware events, configuration changes, and external component interactions that trigger operational context changes and state machine transitions.

**State Change Triggers:**

- Physical media alarm state changes (LOS, LOF, SF, SD) causing link state machine transitions between GSM_LINK_DOWN, GSM_VEIP_DISABLED, GSM_VEIP_CONFIGURE, and GSM_LINK_UP states
- VEIP administrative state modifications through TR-181 parameter writes triggering state machine re-evaluation for interface enable/disable operations
- PLOAM registration state changes indicating ONT ranging and activation status affecting link establishment workflow
- WAN Manager synchronization events when WAN_MANAGER_UNIFICATION_ENABLED requiring physical layer status updates instead of direct ethernet interface management

**Context Switching Scenarios:**

- Switching between standalone GPON management mode and WAN Manager unified mode based on WAN_MANAGER_UNIFICATION_ENABLED build flag changing interface lifecycle responsibility
- HAL client reconnection sequence when JSON HAL server restarts requiring event resubscription and configuration resynchronization
- State machine context switch from GSM_LINK_UP to GSM_LINK_DOWN when physical media LOS alarm activates resulting in VEIP interface disablement

### Call Flow

**Initialization Call Flow:**

```mermaid
sequenceDiagram
    participant Proc as Process Startup
    participant Main as ssp_main
    participant MsgBus as Message Bus
    participant DML as DML Backend
    participant Ctrl as Controller
    participant HAL as HAL Client

    Proc->>Main: main() execution start
    Main->>Main: Parse arguments<br/>Load configuration
    Main->>MsgBus: ssp_Mbi_MessageBusEngage()<br/>Register component namespace
    MsgBus-->>Main: Bus handle returned
    Main->>DML: GponMgrDml_Init()<br/>Plugin initialization
    DML->>DML: GponMgrDml_BackEndManagerCreate()<br/>Allocate backend manager
    DML->>DML: GponMgrDml_BackEndManagerInitialize()<br/>Initialize data structures
    DML-->>Main: Plugin ready
    Main->>HAL: GponHal_Init()<br/>Initialize HAL client
    HAL->>HAL: json_hal_client_init()<br/>Load configuration from JSON
    HAL->>HAL: json_hal_client_run()<br/>Start RPC client
    HAL->>HAL: Wait for server connection<br/>Retry up to 10 seconds
    HAL-->>Main: Connection established
    Main->>Ctrl: GponMgr_Controller_Init()<br/>Initialize controller logic
    Ctrl->>HAL: GponMgr_subscribe_hal_events()<br/>Subscribe to hardware events
    HAL-->>Ctrl: Event subscriptions active
    Ctrl->>HAL: GponMgr_send_hal_configuration()<br/>Apply initial config
    HAL-->>Ctrl: Configuration sent
    Ctrl-->>Main: Controller initialized
    Main->>Ctrl: Start state machine thread<br/>gpon_controller_init()
    Ctrl-->>Main: State machine running
    Main->>Proc: Initialization Complete (Active State)
```

**Request Processing Call Flow:**

```mermaid
sequenceDiagram
    participant Client as TR-181 Client
    participant MsgBus as CCSP Message Bus
    participant DML as DML Function Layer
    participant Data as DML Data Layer
    participant HAL as HAL Client
    participant Server as JSON HAL Server

    Client->>MsgBus: GetParameterValues<br/>Device.X_RDK_ONT.PhysicalMedia.1.Status
    MsgBus->>DML: GponPhy_GetParamStringValue()<br/>parameterName="Status"
    DML->>Data: GponMgrDml_GetData_locked()<br/>Acquire mutex lock
    Data-->>DML: GPON_DML_DATA pointer
    DML->>DML: Lookup PhysicalMedia instance<br/>Extract Status field
    DML->>Data: GponMgrDml_GetData_release()<br/>Release mutex lock
    DML-->>MsgBus: Return value "Up"
    MsgBus-->>Client: Response with Status value

    Note over Client,Server: Alternative: Parameter refresh from hardware

    Client->>MsgBus: GetParameterValues<br/>Device.X_RDK_ONT.PhysicalMedia.1.RxPower.SignalLevel
    MsgBus->>DML: GponPhyRxpwr_GetParamIntValue()<br/>parameterName="SignalLevel"
    DML->>HAL: GponHal_get_Physical_Media()<br/>Query hardware for latest data
    HAL->>HAL: Build JSON RPC message<br/>getParameters action
    HAL->>Server: JSON RPC call with parameter path
    Server->>Server: Query hardware registers<br/>Read optical power level
    Server-->>HAL: JSON response with value
    HAL->>HAL: Parse JSON response<br/>Update cached data
    HAL-->>DML: Physical media data updated
    DML->>Data: Update DML_PHY_MEDIA structure<br/>Store SignalLevel value
    DML-->>MsgBus: Return SignalLevel value
    MsgBus-->>Client: Response with RxPower.SignalLevel
```

## TR‑181 Data Models

### Supported TR-181 Parameters

RdkGponManager implements the Device.X_RDK_ONT vendor-specific TR-181 data model namespace providing comprehensive access to GPON ONT hardware parameters, operational statistics, and configuration settings. The implementation supports both read-only monitoring parameters and read-write configuration parameters where hardware capabilities permit modification.

### Object Hierarchy

```
Device.
└── X_RDK_ONT.
    ├── PhysicalMedia.{i}.
    │   ├── Cage (uint32/mapped, R)
    │   ├── ModuleVendor (string, R)
    │   ├── ModuleName (string, R)
    │   ├── ModuleVersion (string, R)
    │   ├── ModuleFirmwareVersion (string, R)
    │   ├── PonMode (uint32/mapped, R)
    │   ├── Connector (uint32/mapped, R)
    │   ├── NominalBitRateDownstream (uint32, R)
    │   ├── NominalBitRateUpstream (uint32, R)
    │   ├── Enable (boolean, R/W) [WAN_MANAGER_UNIFICATION_ENABLED]
    │   ├── Status (uint32/mapped, R)
    │   ├── RedundancyState (uint32/mapped, R)
    │   ├── Alias (string, R/W) [WAN_MANAGER_UNIFICATION_ENABLED]
    │   ├── LastChange (uint32, R) [WAN_MANAGER_UNIFICATION_ENABLED]
    │   ├── LowerLayers (string, R/W) [WAN_MANAGER_UNIFICATION_ENABLED]
    │   ├── Upstream (boolean, R) [WAN_MANAGER_UNIFICATION_ENABLED]
    │   ├── RxPower.
    │   │   ├── SignalLevel (int, R)
    │   │   ├── SignalLevelLowerThreshold (int, R/W)
    │   │   └── SignalLevelUpperThreshold (int, R/W)
    │   ├── TxPower.
    │   │   ├── SignalLevel (int, R)
    │   │   ├── SignalLevelLowerThreshold (int, R/W)
    │   │   └── SignalLevelUpperThreshold (int, R/W)
    │   ├── Voltage.
    │   │   └── VoltageLevel (int, R)
    │   ├── Bias.
    │   │   └── CurrentBias (uint32, R)
    │   ├── Temperature.
    │   │   └── CurrentTemp (int, R)
    │   ├── PerformanceThreshold.
    │   │   ├── SignalFail (uint32, R)
    │   │   └── SignalDegrade (uint32, R)
    │   └── Alarm.
    │       ├── RDI (uint32/mapped, R)
    │       ├── PEE (uint32/mapped, R)
    │       ├── LOS (uint32/mapped, R)
    │       ├── LOF (uint32/mapped, R)
    │       ├── DACT (uint32/mapped, R)
    │       ├── DIS (uint32/mapped, R)
    │       ├── MIS (uint32/mapped, R)
    │       ├── MEM (uint32/mapped, R)
    │       ├── SUF (uint32/mapped, R)
    │       ├── SF (uint32/mapped, R)
    │       ├── SD (uint32/mapped, R)
    │       ├── LCDG (uint32/mapped, R)
    │       ├── TF (uint32/mapped, R)
    │       └── ROGUE (uint32/mapped, R)
    ├── Gtc.
    │   ├── CorrectedFecBytes (uint32, R)
    │   ├── CorrectedFecCodeWords (uint32, R)
    │   ├── UnCorrectedFecCodeWords (uint32, R)
    │   ├── TotalFecCodeWords (uint32, R)
    │   ├── HecErrorCount (uint32, R)
    │   ├── PSBdHecErrors (uint32, R)
    │   ├── FrameHecErrors (uint32, R)
    │   └── FramesLost (uint32, R)
    ├── Ploam.
    │   ├── RegistrationState (uint32/mapped, R)
    │   ├── ActivationCounter (uint32, R)
    │   ├── TxMessageCount (uint32, R)
    │   ├── RxMessageCount (uint32, R)
    │   ├── MicErrors (uint32, R)
    │   └── RegistrationTimers.
    │       ├── TO1 (uint32, R)
    │       └── TO2 (uint32, R)
    ├── Gem.{i}.
    │   ├── PortId (uint32, R)
    │   ├── TrafficType (uint32/mapped, R)
    │   ├── TransmittedFrames (uint32, R)
    │   ├── ReceivedFrames (uint32, R)
    │   └── EthernetFlow.
    │       ├── Ingress.
    │       │   ├── Tagged (uint32/mapped, R)
    │       │   ├── S-VLAN.
    │       │   │   ├── Vid (uint32, R)
    │       │   │   ├── Pcp (uint32, R)
    │       │   │   └── Dei (uint32, R)
    │       │   └── C-VLAN.
    │       │       ├── Vid (uint32, R)
    │       │       ├── Pcp (uint32, R)
    │       │       └── Dei (uint32, R)
    │       └── Egress.
    │           ├── Tagged (uint32/mapped, R)
    │           ├── S-VLAN.
    │           │   ├── Vid (uint32, R)
    │           │   ├── Pcp (uint32, R)
    │           │   └── Dei (uint32, R)
    │           └── C-VLAN.
    │               ├── Vid (uint32, R)
    │               ├── Pcp (uint32, R)
    │               └── Dei (uint32, R)
    ├── Omci.
    │   ├── RxBaseLineMessageCountValid (int, R)
    │   ├── RxExtendedMessageCountValid (int, R)
    │   └── MicErrors (uint32, R)
    ├── Veip.{i}.
    │   ├── MeId (uint32, R)
    │   ├── AdministrativeState (uint32/mapped, R)
    │   ├── OperationalState (uint32/mapped, R)
    │   ├── InterDomainName (string, R)
    │   ├── InterfaceName (string, R)
    │   └── EthernetFlow.
    │       ├── Ingress.
    │       │   ├── Tagged (uint32/mapped, R/W)
    │       │   └── Q-VLAN.
    │       │       ├── Vid (uint32, R/W)
    │       │       ├── Pcp (uint32, R)
    │       │       └── Dei (uint32, R)
    │       └── Egress.
    │           ├── Tagged (uint32/mapped, R/W)
    │           └── Q-VLAN.
    │               ├── Vid (uint32, R/W)
    │               ├── Pcp (uint32, R)
    │               └── Dei (uint32, R)
    └── TR69.
        ├── url (string, R)
        └── AssociatedTag (uint32, R)
```

### Parameter Registration and Access

- **Implemented Parameters**: RdkGponManager implements all parameters within Device.X_RDK_ONT namespace defined in RdkGponManager.xml configuration file with dynamic table support for PhysicalMedia, Gem, and Veip objects supporting up to 128 instances per table
- **Parameter Registration**: Parameters are registered during component initialization through DML plugin mechanism (`GponMgrDml_Init`) which acquires CCSP function callbacks and registers object hierarchy with Data Model Agent enabling TR-181 access via CCSP message bus
- **Access Mechanism**: External components access parameters through CCSP message bus using standard GetParameterValues, SetParameterValues, GetParameterNames, and GetParameterAttributes APIs with component namespace resolution directing requests to appropriate DML handler functions
- **Validation Rules**: Writable parameters undergo validation through Validate/Commit/Rollback pattern with hardware capability checks performed in Validate phase before committing changes to HAL layer, with transaction rollback support for atomic multi-parameter updates

## Internal Modules

RdkGponManager is organized into distinct functional modules handling service support platform integration, TR-181 data model implementation, GPON controller logic, link state management, and hardware abstraction layer communication.

| Module/Class | Description | Key Files |
|-------------|------------|-----------|
| **Service Support Platform** | Component lifecycle management providing process entry point, CCSP message bus integration, signal handling, component action dispatch, and systemd integration for process supervision | `ssp_main.c`, `ssp_action.c`, `ssp_messagebus_interface.c`, `ssp_global.h`, `ssp_internal.h` |
| **DML Plugin Layer** | TR-181 data model plugin providing CCSP Data Model Agent integration through function callback registration, backend manager initialization, and object hierarchy registration for Device.X_RDK_ONT namespace | `gponmgr_dml_plugin_main.c`, `gponmgr_dml_plugin_main.h` |
| **DML Backend Manager** | Backend infrastructure managing COSA data model object creation, initialization lifecycle, and global function callback references for coordinating TR-181 operations across DML modules | `gponmgr_dml_backendmgr.c`, `gponmgr_dml_backendmgr.h` |
| **DML Data Layer** | Centralized data structure management maintaining GPON hardware state cache with thread-safe access through mutex-protected getter/release functions for PhysicalMedia, Veip, Gem, Gtc, Ploam, Omci, and TR69 data | `gponmgr_dml_data.c`, `gponmgr_dml_data.h` |
| **DML Function Handlers** | TR-181 parameter function implementations for PhysicalMedia, Gtc, Ploam, Gem, Omci, Veip, and TR69 objects providing GetParamValue, SetParamValue, IsUpdated, Synchronize, GetEntryCount, GetEntry, Validate, Commit, and Rollback operations | `gponmgr_dml_func.c`, `gponmgr_dml_func.h` |
| **DML Object Layer** | High-level object management for TR-181 tables implementing dynamic instance management, object addition/deletion logic, and coordination between DML function layer and backend data structures | `gponmgr_dml_obj.c`, `gponmgr_dml_obj.h` |
| **DML Internal Logic** | Internal DML utility functions and helper routines supporting data model operations including type conversions, enumeration mappings, and cross-module coordination functions | `gponmgr_dml_internal.c`, `gponmgr_dml_internal.h` |
| **DML Ethernet Interface** | Ethernet gateway interface management module handling interface addition, deletion, enable/disable operations, and lower layer binding for standalone mode when WAN Manager unification is disabled | `gponmgr_dml_eth_iface.c`, `gponmgr_dml_eth_iface.h` |
| **GPON Controller** | High-level controller orchestrating component initialization sequence, HAL event subscription for hardware notifications, initial configuration transmission, and coordination between state machine and TR-181 layer | `gponmgr_controller.c`, `gponmgr_controller.h` |
| **Link State Machine** | Autonomous state machine managing GPON link lifecycle through states GSM_LINK_DOWN, GSM_VEIP_DISABLED, GSM_VEIP_CONFIGURE, and GSM_LINK_UP with transition logic based on physical media alarms, VEIP operational states, and administrative control | `gponmgr_link_state_machine.c`, `gponmgr_link_state_machine.h` |
| **HAL Interface** | JSON HAL client abstraction providing synchronous RPC operations for querying and configuring GPON hardware through json_hal_client library with connection management, retry logic, and response parsing | `gponmgr_dml_hal.c`, `gponmgr_dml_hal.h` |
| **HAL Parameter Mapping** | Parameter translation layer converting TR-181 parameter paths to JSON HAL schema paths and marshaling data between CCSP types and JSON message formats for getParameters, setParameters, and event subscription operations | `gponmgr_dml_hal_param.c`, `gponmgr_dml_hal_param.h` |

## Component Interactions

RdkGponManager maintains interactions with RDK-B middleware components for TR-181 access, vendor HAL layer for hardware control, system services for configuration persistence, and optionally with WAN Manager for unified interface management.

### Interaction Matrix

| Target Component/Layer | Interaction Purpose | Key APIs/Endpoints |
|------------------------|-------------------|------------------|
| **RDK-B Middleware Components** |
| CcspPandM | TR-181 parameter access and modification, component registration and capability advertisement | `GetParameterValues`, `SetParameterValues`, `GetParameterNames`, `GetParameterAttributes` via CCSP Message Bus |
| WAN Manager | Physical layer status synchronization for unified WAN management when WAN_MANAGER_UNIFICATION_ENABLED | `CosaDmlGponSetPhyStatusForWanManager()` providing interface status INTERFACE_UP/INTERFACE_DOWN |
| **System & HAL Layers** |
| JSON HAL Server | GPON hardware query and configuration through JSON RPC communication | `json_hal_client_init()`, `json_hal_client_run()`, `json_hal_client_get_request()`, `json_hal_client_set_request()`, `json_hal_client_subscribe_event()` |
| systemd | Process lifecycle notification signaling component ready state when ENABLE_SD_NOTIFY enabled | `sd_notify()` with READY=1 status message |

**Events Published by RdkGponManager:**

| Event Name | Event Topic/Path | Trigger Condition | Subscriber Components |
|------------|-----------------|-------------------|---------------------|
| PhysicalMedia Status Change | `Device.X_RDK_ONT.PhysicalMedia.{i}.Status` | Physical media operational status transitions between Up, Down, Unknown, Dormant, NotPresent, LowerLayerDown, Error states | WAN Manager, Monitoring Systems, Logging Services |
| PhysicalMedia Alarm | `Device.X_RDK_ONT.PhysicalMedia.{i}.Alarm` | Alarm condition state change for LOS, LOF, SF, SD, or other physical layer alarms transitioning between Active and Inactive | Link State Machine, WAN Manager, Alarm Management Systems |
| VEIP Administrative State | `Device.X_RDK_ONT.Veip.{i}.AdministrativeState` | VEIP administrative state change between Lock and Unlock triggered by OMCI configuration | Link State Machine, Interface Management |
| VEIP Operational State | `Device.X_RDK_ONT.Veip.{i}.OperationalState` | VEIP operational state transitions affecting interface availability | Link State Machine, WAN Manager, Connectivity Monitoring |
| PLOAM Registration State | `Device.X_RDK_ONT.Ploam.RegistrationState` | ONT registration state change through PLOAM protocol ranging and activation process | WAN Manager, Provisioning Systems, Operational Monitoring |

### IPC Flow Patterns

**Primary IPC Flow - TR-181 Parameter Query:**

```mermaid
sequenceDiagram
    participant Client as TR-181 Client (PandM)
    participant MsgBus as CCSP Message Bus
    participant DML as DML Function Layer
    participant HAL as HAL Client
    participant Server as JSON HAL Server
    participant HW as GPON Hardware

    Client->>MsgBus: GetParameterValues request<br/>Device.X_RDK_ONT.PhysicalMedia.1.RxPower.SignalLevel
    MsgBus->>DML: Dispatch to GponPhyRxpwr_GetParamIntValue()
    Note over DML: Check if cached data is stale<br/>or requires hardware refresh
    DML->>HAL: GponHal_get_Physical_Media()<br/>Request parameter update
    HAL->>HAL: json_hal_client_get_request()<br/>Build JSON getParameters message
    HAL->>Server: JSON RPC call<br/>{"action":"getParameters","name":"Device.X_RDK_ONT.PhysicalMedia.1.RxPower"}
    Server->>HW: Read optical power registers<br/>Hardware-specific access
    HW-->>Server: Register values
    Server->>Server: Format values per schema<br/>Apply unit conversions
    Server-->>HAL: JSON response<br/>{"type":"int","value":-15}
    HAL->>HAL: Parse JSON response<br/>Extract SignalLevel value
    HAL->>DML: Update cached data structure<br/>DML_POWER.SignalLevel = -15
    DML-->>MsgBus: Return value -15
    MsgBus-->>Client: Response (success, value=-15)
```

**Event Notification Flow:**

```mermaid
sequenceDiagram
    participant HW as GPON Hardware
    participant Server as JSON HAL Server
    participant HAL as HAL Client (Subscriber)
    participant Ctrl as GPON Controller
    participant SM as Link State Machine
    participant WanMgr as WAN Manager

    HW->>Server: Hardware interrupt<br/>Physical media LOS alarm activated
    Server->>Server: Detect event condition<br/>Check active subscriptions
    Server->>HAL: publishEvent JSON message<br/>{"action":"publishEvent","name":"Device.X_RDK_ONT.PhysicalMedia.1.Alarm.LOS","value":"Active"}
    HAL->>HAL: Route to registered callback<br/>eventcb_PhysicalMediaAlarmsAll()
    HAL->>Ctrl: Event callback invoked<br/>Update DML data with alarm state
    Ctrl->>Ctrl: GponMgrDml_GetData_locked()<br/>Acquire data structure mutex
    Ctrl->>Ctrl: Update PhysicalMedia.Alarm.LOS = ACTIVE<br/>Modify cached state
    Ctrl->>Ctrl: GponMgrDml_GetData_release()<br/>Release mutex
    Note over SM: State machine periodic evaluation<br/>500ms loop iteration
    SM->>SM: check_gpon_physical_media_alarm_los()<br/>Detect LOS alarm active
    SM->>SM: State transition GSM_LINK_UP -> GSM_LINK_DOWN<br/>Execute gpon_disable_veip_interface()
    alt WAN_MANAGER_UNIFICATION_ENABLED
        SM->>WanMgr: CosaDmlGponSetPhyStatusForWanManager()<br/>Notify INTERFACE_DOWN
        WanMgr-->>SM: Acknowledgment
    else Standalone Mode
        SM->>SM: Gponmgr_eth_setEnableInterface(FALSE)<br/>Disable ethernet gateway interface
    end
    Note over SM: Link down handling complete<br/>Continue monitoring for recovery
```

## Implementation Details

### Major HAL APIs Integration

RdkGponManager integrates with vendor-specific GPON hardware through JSON HAL client-server architecture using standardized schema-based message exchange for hardware abstraction and platform portability.

**Core HAL APIs:**

| HAL API | Purpose | Implementation File |
|---------|---------|-------------------|
| `json_hal_client_init()` | Initialize JSON HAL client library with configuration file path specifying HAL server port and schema location | `gponmgr_dml_hal.c` |
| `json_hal_client_run()` | Start JSON HAL client RPC mechanism establishing connection to vendor HAL server | `gponmgr_dml_hal.c` |
| `json_hal_is_client_connected()` | Query connection status between HAL client and server with retry logic during initialization | `gponmgr_dml_hal.c` |
| `json_hal_client_get_request()` | Send getParameters JSON RPC request to query hardware parameter values by TR-181 path | `gponmgr_dml_hal.c` |
| `json_hal_client_set_request()` | Send setParameters JSON RPC request to configure hardware parameters through TR-181 path | `gponmgr_dml_hal.c` |
| `json_hal_client_subscribe_event()` | Subscribe to hardware event notifications with onChange or interval notification types | `gponmgr_dml_hal.c` |
| `GponHal_Init()` | High-level HAL initialization launching vendor HAL server process, initializing client library, and verifying connection establishment | `gponmgr_dml_hal.c` |
| `GponHal_get_Physical_Media()` | Query all PhysicalMedia instance data including module information, power levels, alarms, and operational status | `gponmgr_dml_hal.c` |
| `GponHal_get_veip()` | Query VEIP instance data including administrative state, operational state, and ethernet flow configuration | `gponmgr_dml_hal.c` |
| `GponHal_get_gem()` | Query GEM port instance data including port IDs, traffic types, frame counts, and VLAN flow parameters | `gponmgr_dml_hal.c` |
| `GponHal_get_gtc()` | Query GTC layer statistics including FEC counters, HEC errors, and frame loss metrics | `gponmgr_dml_hal.c` |
| `GponHal_get_ploam()` | Query PLOAM statistics including registration state, activation counters, and message counts | `gponmgr_dml_hal.c` |
| `GponHal_Event_Subscribe()` | Subscribe to specific hardware events by parameter path with callback function registration | `gponmgr_dml_hal.c` |
| `GponHal_send_config()` | Transmit initial configuration to hardware layer sending power thresholds and VEIP flow settings during component initialization (not persistent across reboots) | `gponmgr_dml_hal.c` |

### Key Implementation Logic

- **State Machine Engine**: Link state machine implemented in `gponmgr_link_state_machine.c` executing periodic 500ms loop evaluating conditions for state transitions between GSM_LINK_DOWN, GSM_VEIP_DISABLED, GSM_VEIP_CONFIGURE, and GSM_LINK_UP states using function pointers for action execution and transition logic. State transition handlers invoke hardware query functions checking VEIP enabled status, physical media alarm conditions, and interface operational states. State transition functions implementing transition logic: `gpon_sm_transition_Start()`, `gpon_sm_transition_LinkDown_to_LinkUp()`, `gpon_sm_transition_VeipDisabled_to_VeipConfigure()`, `gpon_sm_transition_VeipConfigure_to_LinkUp()`, `gpon_sm_transition_LinkUp_to_LinkDown()`.
  
- **Event Processing**: Hardware events received asynchronously through JSON HAL publishEvent messages routed to registered callback functions based on parameter path subscriptions. Event callbacks defined in `gponmgr_controller.c` and `gponmgr_dml_hal.c` include `eventcb_PhysicalMediaStatus()`, `eventcb_PhysicalMediaAlarmsAll()`, `eventcb_VeipAdministrativeState()`, `eventcb_VeipOperationalState()`, `eventcb_PloamRegistrationState()`. Callbacks acquire data structure mutex, update cached values in `GPON_DML_DATA` structure, release mutex, enabling state machine thread to consume updated state during next evaluation cycle.

- **Error Handling Strategy**: JSON HAL operations return ANSC_STATUS_SUCCESS or ANSC_STATUS_FAILURE with error logging through CcspTraceError macro providing function name, line number, and error context. HAL client connection failures trigger retry mechanism with exponential backoff up to HAL_CONNECTION_RETRY_MAX_COUNT (10 attempts) during initialization. Parameter validation failures in DML Validate functions return error without committing to hardware allowing rollback of multi-parameter transactions. JSON parsing errors during HAL response processing logged with message content dump and safe default value return to prevent data corruption.

- **Logging & Debugging**: Component uses RDK Logger framework when FEATURE_SUPPORT_RDKLOG enabled providing categorized logging through CcspTraceInfo, CcspTraceWarning, CcspTraceError macros. State machine state transitions logged at Info level with current state, next state, and transition reason. HAL communication logged at Debug level including JSON message payloads for request/response correlation. Critical failures including HAL connection loss, event subscription failure, or state machine initialization failure logged at Error level with component restart recommendation. 

### Key Configuration Files

| Configuration File | Purpose | Override Mechanisms |
|--------------------|---------|--------------------|
| `/etc/rdk/conf/gpon_manager_conf.json` | HAL client configuration specifying json_hal_server port number and schema file path used by json_hal_client_init | No runtime override, requires file modification and process restart |
| `/etc/rdk/conf/gpon_manager_wan_unify_conf.json` | Alternative HAL configuration used when WAN_MANAGER_UNIFICATION_ENABLED build flag is defined | Selected at compile time via build flag, runtime selection not supported |
| `/etc/rdk/conf/RdkGponManager.xml` | TR-181 object hierarchy definition specifying DML function mappings, parameter types, syntax, and access permissions for Device.X_RDK_ONT namespace | Loaded during DML initialization, modifications require component restart |
| `/etc/rdk/schemas/gpon_hal_schema.json` | JSON HAL schema definition specifying supported actions, parameter paths, data types, and enumeration mappings for GPON HAL protocol validation | Schema path referenced from gpon_manager_conf.json, vendor HAL server must provide matching schema version |

