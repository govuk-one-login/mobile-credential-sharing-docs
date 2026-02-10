# Verifier Solution Architecture

## Executive Summary

This document outlines the architectural patterns and orchestration logic
required to implement an ISO 18013-5 compliant mDL Verifier. It maps the
complete transaction lifecycle from managing device permissions to the final
cryptographic validation of a mobile ID. It uses an Orchestrator-driven pattern
that centralises and enforces app logic state via a passive state machine.

## The Mental Model: Orchestrator & Session

The architecture separates **Execution** (doing things) from **State** (tracking
the User's position in the flow).

1. `Orchestrator`
    - **Role**: The business logic of the app.
    - **Responsibility**: Owns the hardware services (Bluetooth, Crypto,
      Camera). It initiates all actions (for example, calling
      `bluetooth.connect()`).
    - **State Control**: Observes the results of these actions and attempts to
      transition the Session to the appropriate next state.
    - **Associated Jira tickets**:
      - `VerifierOrchestrator` implementation:
        - Android: DCMAW-18157
        - iOS: DCMAW-18159
2. **VerifierSession (the map)**  
    - **Role**: Passive Finite State Machine (FSM).
    - **Responsibility**: Enforces the ISO 18013-5 sequence. It doesn't perform
      work or side effects.
    - **The Guardrail**: Exposes a `transition(to: State)` method and validates
      that the requested transition is legal based on the current state
      (for example, allowing `Scanning → Connecting`, but blocking
      `Scanning → Success`).
    - **Associated Jira tickets**:
      - `VerifierSession` implementation:
        - Android: DCMAW-18158
        - iOS: DCMAW-18160

## Lifecycle & Ephemerality

The `VerifierSession` is a **Single-Use Object**. It corresponds 1:1 with a
specific cryptographic session.

1. **Terminal States**: Once the session reaches `.success`, `.failed`, or
  `.cancelled`, it's immutable. It can't be reset or rewound.
2. **Handling Retries**: To restart a flow (for example, after a Bluetooth
   drop), the `Orchestrator` must **discard** the current session instance and
   instantiate a new one. This ensures that a fresh set of Ephemeral Keys and
   generates a new Session Transcript for every attempt, strictly adhering to
   the ISO 18013-5 security model.

## Architectural Flow

The verification process divides into four distinct and unidirectional,
sequential phases:

1. **Pre-flight Checks**: Authorises required capabilities:
   - Bluetooth
   - Camera
   - **Associated Jira tickets**:
     - Listen to / standardise app permission state:
       - Android: DCMAW-18019
       - iOS: DCMAW-18021
2. **Device Engagement**: Capturing the QR code to extract the Holder’s identity
   and initialise session cryptography.
3. **Transport & Data**: Establishing the secure Bluetooth Low Energy (BLE)
   tunnel, sending the request, and retrieving the encrypted response.
4. **Completion**: Validating the cryptographic trust chain and tearing down
   the connection.

## System Components

- Orchestrator
- VerifierSession
- PrerequisiteGate
- BluetoothTransport
- CameraService
- CryptoService

## Verifier Session

Verifier Session is a state machine, deciding what screen should show.
For example, the permissions needed, scanning in progress, connected, reading,
success, or failure of the User journey and triggering one-off effects or
actions such as launching permission prompt, starting a system activity.

<details>
<summary>Android VerifierSessionState sealed class</summary>

```kotlin
sealed class  VerifierSessionState {
    data object NotStarted : VerifierSessionState()
    data class Preflight(
        val missingPermissions: Set<String>
    ) : VerifierSessionState()
    data object ReadyToScan : VerifierSessionState()
    data object Connecting : VerifierSessionState()
    data object ProcessingEngagement : VerifierSessionState()
    data object Verifying: VerifierSessionState()
    sealed class Complete( val reason: String) : VerifierSessionState() {
        data class Success(
            val data: DeviceResponse
        ) : Complete("Successful journey")
        data class Failed(
            val error: SessionError
        ) : Complete(error.message, error.exception)
        data object Cancelled : Complete("Journey cancelled by User")
    }
}
```

</details>

<details>
<summary>iOS VerifierSessionState enum</summary>

```swift
enum VerifierSessionState: Equatable {
 case notStarted
 case preflight(missing: Set<Permission>)
 case readyToScan
 case processingEngagement
 case connecting
 case verifying
 case success(data: DeviceResponse)
 case failed(error: SessionError)
 case cancelled
}
```

</details>

| Diagram Phase | State | UI Responsibility |
| --- | --- | --- |
| Startup | Initialising | Show a skeleton loader or generic "Starting…" state. |
| 1. Pre-flight Checks | Preflight | If this state is active, the capabilities aren't determined. Render a view explaining to the user the need for the missing permission. |
| 1. Pre-flight Checks | ReadyToScan | Confirms permissions, start the Camera. |
| 2. Device Engagement | ProcessingEngagement | Indicate to the user that a scan was successfully completed. |
| 3. Transport &amp; Data | Connecting | This covers the BLE connection + Session Establishment. |
| 3. Transport &amp; Data | Verifying | This covers Data Retrieval. |
| 4. Completion | Success | Display the relevant data. |
| 4. Completion | Failed | Handle specific errors, such as prompting to fix settings, or show a generic error. |
| Interruption &amp; Cancellation | Cancelled | Dismiss the flow. |

## Startup

The `Orchestrator` is a long-lived object that persists across the app lifecycle
(or screen lifecycle).

When the user initiates a transaction, the `Orchestrator` instantiates a fresh
`VerifierSession` in the `Initialising` state.

This session instance is ephemeral: it lives only for the duration of this
specific transaction. Discards the specific session object if the transaction
fails or completes.

```mermaid
sequenceDiagram
    autonumber
    
    participant User
    participant UI as View
    participant OR as Orchestrator
    participant VS as VerifierSession
    participant PG as PrerequisiteGate
    participant CAM as CameraService
    participant BLE as BluetoothTransport
    participant CRY as CryptoService

    User->>UI: Tap "Scan ID"
    UI->>OR: startVerification()
    
    OR->>OR: create VerifierSession
    OR->>VS: init(state: .initialising)
    VS-->>OR: state = Initialising
```

## 1. Pre-flight Checks

The flow for Verification requires several device capabilities to work,
including access to **Camera** (for scanning a QR code to initiate a session),
**Bluetooth** (for data exchange) and, on Android, **Location** (for pairing).

With the `VerifierSession` in `Initialising` state, the `Orchestrator` calls the
`PrerequisiteGate` to check firstly that device capabilities are present, and
then that the User has granted permission to access them.

The `PrerequisiteGate` returns with a set of missing capabilities, if any. The
`Orchestrator` transitions the `VerifierSession` into a state of
`Preflight(missing: {<Capability>})`.

By passing this as a set, this enables the View and ViewModel to present an
on-boarding flow with the correct number of steps. For example, if the set
contains both Camera and Bluetooth as missing permissions, the view can prepare
on-boarding that presents these sequentially with explanations for each.

The Orchestrator loops through each permission, retrying each check until
they're granted.

Once the `PrerequisiteGate` determines that there are no missing capabilities,
the `Orchestrator` transitions the `VerifierSession` to a `ReadyToScan` state.

```mermaid
sequenceDiagram
    autonumber
    
    participant User
    participant UI as View
    participant OR as Orchestrator
    participant VS as VerifierSession
    participant PG as PrerequisiteGate
    participant CAM as CameraService
    participant BLE as BluetoothTransport
    participant CRY as CryptoService

    Note over User, OR: 1. Pre-flight Checks

    %% 1. Batch Check
    OR->>PG: checkCapabilities()
    PG-->>OR: missing: {Camera, Bluetooth}

    alt permissions missing
        OR->>VS: transition(to: .preflight(missing))
        VS-->>OR: state = Preflight
        OR->>UI: render(state: .preflight)
        
        %% 2. Unified On-boarding
        UI-->>User: Show On-boarding Flow (Sequence of Steps)
        
        %% 3. Sequential Resolution
        loop For each capability
            User->>UI: Tap "Enable Capability"
            UI->>OR: resolve(capability)
            OR->>PG: requestAuthorization(capability)
            PG-->>User: OS System Prompt
            User-->>PG: Grant / Deny
            PG-->>OR: status: .authorized
            
            %% Re-check to confirm progress or completion
            OR->>PG: checkCapabilities()
            PG-->>OR: missing: RemainingSet
            OR->>UI: updateProgress(remaining)
        end

    else all permissions granted
        OR->>VS: transition(to: .readyToScan)
        VS-->>OR: state = ReadyToScan
        OR->>UI: render(state: .readyToScan)
    end
```

## 2. Device Engagement

Completing the pre-flight checks then begins the Device Engagement.

The `Orchestrator` instructs the `CameraService` to start scanning. When the
user scans a presented QR code, the `CameraService` returns the detected code
to the `Orchestrator`, which then stops the camera.

With this raw string, the `Orchestrator` transitions the `VerifierSession` to
`ProcessingEngagement`.

The `Orchestrator` passes the string to the `CryptoService` to decode and
validate the engagement structure.

If the string is a valid mDL format, the `CryptoService` generates an ephemeral
key pair for the Verifier, and combines its Public Key with the raw Device
Engagement bytes to form the Session Transcript. Upon success, the
`CryptoService` holds the symmetric session keys (`SKReader` and `SKDevice`)
required to encrypt the initial Bluetooth request.

Notice that the `Orchestrator` calls the `CryptoService` before interacting with
Bluetooth. This is because mDL requires encryption from the very first message.

With the keys established, the `Orchestrator` transitions the `VerifierSession`
to the `Connecting` state.

```mermaid
sequenceDiagram
    autonumber
    
    participant User
    participant UI as View
    participant OR as Orchestrator
    participant VS as VerifierSession
    participant PG as PrerequisiteGate
    participant CAM as CameraService
    participant BLE as BluetoothTransport
    participant CRY as CryptoService

    Note over User, OR: 2. Device Engagement

    OR->>CAM: startScanning()
    OR->>UI: showCameraPreview()
    UI-->>User: Show Camera Feed
    
    User->>CAM: Point camera at QR Code
    activate CAM
    CAM-->>CAM: Detect Pattern
    CAM-->>OR: qrCodeString
    deactivate CAM
    
    OR->>CAM: stopScanning()
    
    OR->>VS: transition(to: .processingEngagement)
    VS-->>OR: state = ProcessingEngagement
    OR->>UI: render(state: .processingEngagement)
    Note right of UI: eg. "Reading code…"
    
    OR->>CRY: decode(qrCodeString)
    
    alt Invalid QR Code
        CRY-->>OR: error(invalidFormat)
        OR->>VS: transition(to: .failed) 
        VS-->>OR: state = Failed
        OR->>UI: showError("Invalid QR Code")
    else Valid mDL QR Code
        CRY-->>OR: deviceEngagement
        Note left of CRY: Contains BLE UUID,<br/>Device Public Key,<br/>Version
        
        OR->>CRY: createSessionKeys(deviceEngagement)
        CRY-->>OR: sessionKeys (SKReader, SKDevice)
        Note left of CRY: Derived from SessionTranscript<br/>& Ephemeral Keys
        
        Note right of OR: We now have the UUID<br/>to dial the device
        OR->>VS: transition(to: .connecting)
        VS-->>OR: state = Connecting
    end
```

## 3. Transport & Data

With the `VerifierSession` in the `Connecting` state, the `Orchestrator` begins
the transport phase.

This includes two key components according to the ISO specification:

1. **Session Establishment**: the act of sending the first message to “open” the
   secure tunnel.
2. **Data Retrieval**: the entire phase of asking for data and getting a
   response.

### 3.1. Transport Layer Setup

The `Orchestrator` instructs the `BluetoothTransport` layer to initiate a
connection using the specific Service's Universally Unique Identifier (UUID)
extracted from the QR code. The transport layer scans for the advertising
device and establishes a raw BLE connection.

```mermaid
sequenceDiagram
    autonumber
    
    participant User
    participant UI as View
    participant OR as Orchestrator
    participant VS as VerifierSession
    participant PG as PrerequisiteGate
    participant CAM as CameraService
    participant BLE as BluetoothTransport
    participant CRY as CryptoService

    rect rgb(210, 245, 210)

    OR->>BLE: connect(uuid: deviceEngagement.uuid)
    activate BLE
    Note right of BLE: Finding Device…
    BLE-->>OR: connected
    deactivate BLE
     
    end
```

### 3.2. Session Establishment

Once connected, the `Orchestrator` requests the `CryptoService` to construct the
`SessionEstablishment` message.

The ISO standard defines a single message structure called
`SessionEstablishment` that carries both:

1. The Verifier’s Public Key (`eReaderKey`)—so the Holder can generate their
   own shared secret.
2. The Encrypted Request (`DeviceRequest`)—the actual attributes we're asking
   for.

The `Orchestrator` sends this combined payload via the `BluetoothTransport` to
the Holder.

> **Note:** By binding the session keys to the original QR code (via the Session
Transcript), we ensure we're connecting to the exact scanned device.
>

```mermaid
sequenceDiagram
    autonumber
    
    participant User
    participant UI as View
    participant OR as Orchestrator
    participant VS as VerifierSession
    participant PG as PrerequisiteGate
    participant CAM as CameraService
    participant BLE as BluetoothTransport
    participant CRY as CryptoService
    
    rect rgb(210, 245, 210)

    OR->>CRY: generateSessionEstablishment(items: [.familyName, .ageIsOver18])
    activate CRY
    Note left of CRY: Constructs 'SessionEstablishent'<br/>(Ephemeral Key + Encrypted DeviceRequest)
    CRY-->>OR: sessionEstablishmentPayload
    deactivate CRY

    OR->>BLE: send(sessionEstablishmentPayload)
    
    end
```

### 3.3. Data Retrieval

The `Orchestrator` waits for the Holder to consent and respond. Upon receiving
the `SessionData` payload, the `BluetoothTransport` passes the encrypted bytes
to the `Orchestrator`. The `Orchestrator` transitions the `VerifierSession` to
`Verifying` and passes the data to `CryptoService`.

The service performs three actions:

1. **Decryption**: Unlocks the `DeviceResponse` using the derived session keys.
2. **Trust Validation**: Verifies the digital signature of the
   Mobile Security Object (MSO) against the Trusted Issuer Root CA.
3. **Integrity Check**: Confirms that the hashes of the received data elements
   match the signed digests in the MSO.

Once processing is complete, the `Orchestrator` triggers the transport layer to
disconnect.

```mermaid
sequenceDiagram
    autonumber
    
    participant User
    participant UI as View
    participant OR as Orchestrator
    participant VS as VerifierSession
    participant PG as PrerequisiteGate
    participant CAM as CameraService
    participant BLE as BluetoothTransport
    participant CRY as CryptoService
    
    rect rgb(210, 245, 210)

    activate BLE
    Note right of BLE: Waiting for Holder Consent…
    BLE->>OR: didReceive(sessionData)
    deactivate BLE
    
    OR->>VS: transition(to: .verifying)
    VS-->>OR: state = Verifying
    OR->>UI: render(state: .verifying)
    Note right of UI: "Verifying Signature…"
    
    OR->>CRY: processResponse(sessionData)
    activate CRY
    Note left of CRY: Decrypts Response,<br/>Validates Issuer Signature<br/>& Checks Integrity
    CRY-->>OR: deviceResponse (Decrypted & Validated)
    deactivate CRY
    
    Note right of OR: Teardown
    OR->>BLE: disconnect()
    
    end
```

### 3.4. Completion

The `Orchestrator` concludes the session by transitioning to a final state:

- **Success**: Returns the decrypted identity object for display if verification
  passes.
- **Failure**: Returns a specific error if signature validation fails or the
  connection drops.

```mermaid
sequenceDiagram
    autonumber
    
    participant User
    participant UI as View
    participant OR as Orchestrator
    participant VS as VerifierSession
    participant PG as PrerequisiteGate
    participant CAM as CameraService
    participant BLE as BluetoothTransport
    participant CRY as CryptoService    

    rect rgb(210, 245, 210)

    alt Valid Identity
      OR->>VS: transition(to: .success(IdentityData))
      VS-->>OR: state = Success
      OR-->>UI: showResult(IdentityData)
    else Verification Failed
      OR->>VS: transition(to: .failed(Reason))
      VS-->>OR: state = Failed
      OR-->>UI: showError(Reason)
    end
    
    end
```

## 4. Interruption & Cancellation

The cancellation flow handles user-initiated interruptions at any stage of the
verification process.

When cancellation occurs, the `Orchestrator` must exit and tear down
transports, and wipe all ephemeral session keys from memory.

The `Orchestrator` transitions the `VerifierSession` to a final `Cancelled`
state, signalling the UI to dismiss the verification view.

```mermaid
sequenceDiagram
    participant User
    participant UI as View
    participant OR as Orchestrator
    participant VS as VerifierSession
    participant PG as PrerequisiteGate
    participant CAM as CameraService
    participant BLE as BluetoothTransport
    participant CRY as CryptoService

    Note over User, OR: 4. Interruption & Cancellation

    User->>UI: Tap Cancel
    UI->>OR: cancelVerification()
    
    %% Transition First
    OR->>VS: transition(to: .cancelled)
    VS-->>OR: state = Cancelled
    
    %% Teardown Effects
    par Resource Teardown
        OR->>CAM: stopScanning()
        OR->>BLE: disconnect()
        Note right of BLE: Release radio
        
        OR->>CRY: wipeEphemeralKeys()
        Note left of CRY: Wipe Keys, Transcript &<br/>Message Counters
    end
    
    OR->>UI: render(state: .cancelled)
    UI-->>User: Dismiss Flow
```
