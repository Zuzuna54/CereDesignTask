# Task: Design and Develop a Unified Data Ingestion SDK/API

## 1. Background and Current State

Our current data infrastructure consists of two primary components:

- **Data Cloud**: This is our core storage layer, acting as the "source of truth." Data stored here is encrypted and immutable. Direct writes are possible via the Data Cloud SDK, which returns a hash of the written data.
- **Indexing Layer**: Built on top of the Data Cloud, this layer allows users to prepare (index, aggregate, correlate) data in a way that's optimized for queries and AI agents.

Currently, data can be written into our system through three distinct pathways:

- **Data Cloud SDK**: Writes data directly to the Data Cloud infrastructure.
  - _Use Case Example_: Drones video streams (initial storage), Drones telemetry (parallel storage).
- **Activity SDK**: Writes data to the Indexing Layer. The Indexing Layer has the capability to then store these events in the Data Cloud.
  - _Use Case Example_: Telegram Mini App quests (user events).
- **HTTP API**: Writes data to the Indexing Layer. Similar to the Activity SDK, the Indexing Layer can persist this data to the Data Cloud.
  - _Use Case Example_: Telegram messages (user events), Drones telemetry (parallel storage), Drones video stream events.

**Key Observation:** The Indexing Layer often acts as an intermediary that also ensures data makes its way to the Data Cloud, particularly for event-driven data from the Activity SDK and HTTP API.

---

## 2. The Challenge and Goal

The current fragmented approach to data ingestion presents several challenges:

- **Complexity for Developers**: Developers need to understand and choose between multiple SDKs/APIs depending on the data type and desired initial destination.
- **Inconsistent Processing Logic**: Decisions about whether data goes to the Data Cloud, Indexing Layer, or both are often embedded in the client application logic.
- **Limited Flexibility**: Modifying data routing or processing rules requires changes in multiple client applications.

**Primary Goal:**

- Design and implement a Unified SDK/API that:
  - Provides a single, consistent interface for onboarding any type of data into our ecosystem.
  - Allows data processing rules to be specified within the metadata accompanying the data payload. This will dictate how and where the data is stored and processed.

---

## 3. Detailed Task Description for the New Developer

You are tasked with designing and subsequently implementing this Unified Data Ingestion SDK/API.

### 3.1. Key Requirements for the Unified SDK/API

- **Single Entry Point**: The SDK/API must offer a unified method (e.g., `writeData(payload, metadata)`) for all data ingestion.
- **Abstraction**: It should abstract the complexities of the underlying Data Cloud SDK, Activity SDK, and HTTP API. Client applications should not need to interact with these directly for ingestion.
- **Payload Agnostic (within reason)**: The SDK should be capable of handling various data formats (e.g., JSON, binary data for streams if applicable, though this might involve wrappers).
- **Metadata-Driven Processing**: The SDK/API must interpret processing rules passed in the metadata object.
- **Idempotency (Consideration)**: Explore mechanisms to handle potential retries or duplicate submissions gracefully, if applicable to the use cases.
- **Return Values**: The SDK/API should return meaningful information, such as a transaction ID, status, and potentially the Data Cloud hash if data is written there directly or indirectly.
- **Language/Platform**: (To be specified by your team, e.g., Python, Go, Node.js for the SDK, or a well-defined REST API spec).

### 3.2. Data Processing Rules in Metadata

The metadata object accompanying each `writeData` call will contain specific instructions for how the data should be handled. Examples of processing rules include:

- **dataCloudWriteMode:**
  - `"direct"`: Write directly to Data Cloud (using Data Cloud SDK).
  - `"batch"`: (If applicable) Aggregate data and write to Data Cloud in batches. This implies the Unified SDK might need a temporary staging/buffering mechanism or coordinate with a separate batching service.
  - `"viaIndex"`: Rely on the Indexing Layer to write to Data Cloud (current behavior for Activity SDK/HTTP API).
  - `"skip"`: Do not write this data to the Data Cloud.
- **indexWriteMode:**
  - `"realtime"`: Write to the Indexing Layer (using Activity SDK or HTTP API).
  - `"skip"`: Do not write this data to the Indexing Layer.
- **correlationId (Optional):** To link related data, e.g., a video stream in Data Cloud and its corresponding event in the Indexing Layer.
- **Other potential rules:** (e.g., priority, timeToLiveInIndex, encryptionKeyAlias if different from default).

**Example Metadata:**

```json
// For Telegram Mini App event
{
  "source": "telegram_mini_app",
  "eventType": "quest_completed",
  "processing": {
    "dataCloudWriteMode": "viaIndex", // Indexing layer handles Data Cloud persistence
    "indexWriteMode": "realtime"
  }
}
```

```json
// For Drone video stream (initial storage)
{
  "source": "drone_alpha_7",
  "dataType": "video_stream_chunk",
  "processing": {
    "dataCloudWriteMode": "direct", // Write directly and immediately to Data Cloud
    "indexWriteMode": "skip" // No indexing for the raw stream itself
  }
}
```

```json
// For Drone video stream event (metadata about the stream)
{
  "source": "drone_alpha_7",
  "dataType": "video_stream_event",
  "streamDataCloudHash": "abc123xyz789", // Link to the raw data
  "processing": {
    "dataCloudWriteMode": "skip", // Event itself might not need separate DC storage if hash is enough
    "indexWriteMode": "realtime" // Index this event
  }
}
```

```json
// For Drone telemetry
{
  "source": "drone_beta_1",
  "dataType": "telemetry_packet",
  "processing": {
    "dataCloudWriteMode": "direct", // Or "batch" if desired
    "indexWriteMode": "realtime"
  }
}
```

### 3.3. Supporting Existing Use Cases

The new Unified SDK/API must seamlessly support all current data ingestion patterns:

- **Telegram Mini App Quests:**
  - Data sent via Unified SDK.
  - Metadata: `{ "processing": { "dataCloudWriteMode": "viaIndex", "indexWriteMode": "realtime" } }`
  - Unified SDK routes to Indexing Layer (which then writes to Data Cloud).
- **Telegram Messages:**
  - Data sent via Unified SDK.
  - Metadata: `{ "processing": { "dataCloudWriteMode": "viaIndex", "indexWriteMode": "realtime" } }`
  - Unified SDK routes to Indexing Layer (which then writes to Data Cloud).
- **Drones Telemetry:**
  - Data sent via Unified SDK.
  - Metadata: `{ "processing": { "dataCloudWriteMode": "direct", "indexWriteMode": "realtime" } }` (or `"batch"` for Data Cloud)
  - Unified SDK writes to Data Cloud SDK and Indexing Layer HTTP API/Activity SDK.
- **Drones Video Streams:**
  - **Stream Data:** Sent via Unified SDK.
    - Metadata: `{ "processing": { "dataCloudWriteMode": "direct", "indexWriteMode": "skip" } }`
    - Unified SDK writes to Data Cloud SDK.
  - **Stream Events:** Sent via Unified SDK (after stream data is stored).
    - Metadata: `{ "processing": { "dataCloudWriteMode": "skip", "indexWriteMode": "realtime" }, "streamDataCloudHash": "returned_hash_from_stream_write" }`
    - Unified SDK writes to Indexing Layer HTTP API/Activity SDK.

---

## 4. Suggestions for Solution and Architecture

### 4.1. Unified SDK/API - Core Logic

This component will be the new "front door."

- **Interface:** A well-defined function/endpoint (e.g., `ingest(data, metadata)`).
- **Rules Engine:** A module within the SDK/API responsible for parsing the processing rules from the metadata.
- **Dispatcher:** Based on the parsed rules, this module will decide which underlying system(s) to call:
  - Data Cloud SDK
  - Activity SDK (or its internal logic if the Unified SDK replaces it)
  - HTTP API for the Indexing Layer
- **Orchestration:** For rules requiring multiple actions (e.g., write to Data Cloud and Index), the dispatcher needs to manage this. Consider error handling and atomicity if required (e.g., what if writing to Data Cloud succeeds but Indexing fails?). For simpler cases, sequential calls might be sufficient.
- **Batching** (if `dataCloudWriteMode: "batch"` is implemented):
  - This is a more complex feature. The Unified SDK might:
    - Buffer data locally (risky for stateless SDK instances, better for stateful services).
    - Send data to a dedicated microservice responsible for batching and writing to Data Cloud.
    - Utilize a message queue (e.g., Kafka) that a batch processor consumes.

### 4.2. Implementation Considerations

- **Phased Rollout:**
  - Start by implementing the Unified SDK to support the most common or critical use cases.
  - Gradually migrate existing applications to use the new SDK.
- **SDK vs. Service:**
  - **SDK:** A library integrated directly into client applications. Simpler to start, but updates require client app redeployments.
  - **Service (API Gateway Pattern):** A standalone service (e.g., a REST API) that client applications call. This service then implements the unified logic. Centralizes logic, easier to update, but introduces a network hop. A hybrid approach is also possible (a thin SDK that calls the central service).
  - Given the need for potential batching and centralized rule interpretation, a central service approached by a lightweight SDK might be more robust and scalable in the long run.
- **Configuration Management:** How will the Unified SDK/API know the endpoints for the Data Cloud SDK, Indexing Layer API, etc.? This should be configurable.
- **Testing:** Rigorous testing will be crucial, covering all combinations of processing rules and data types.
- **Monitoring and Logging:** Implement comprehensive logging to track data flow, rule execution, and any errors.

---

## 5. Expected Outcome (from this specific task)

- **Detailed Architecture Document:**
  - Finalized architecture diagram.
  - Detailed design of the Unified SDK/API (or service), including its internal components (Rules Interpreter, Dispatcher).
  - Clear specification of the metadata structure, particularly the processing rules and their accepted values.
  - Data flow diagrams for each of the existing use cases, showing how they would work with the new Unified SDK/API.
  - Error handling strategies.
  - Scalability and performance considerations.
  - Deployment strategy.
- **Roadmap and Implementation Plan:**
  - Breakdown of implementation into phases/milestones.
  - Technology stack choices (if not already decided).
  - Estimated effort for each phase.
  - Plan for testing and QA.
  - Plan for migrating existing applications.

This document should serve as the blueprint for the development team.

---

## 6. Next Steps (for the new developer)

- **Deep Dive:** Thoroughly understand the existing Data Cloud SDK, Activity SDK, and HTTP API functionalities and limitations.
- **Clarify Ambiguities:** Discuss any unclear requirements or edge cases with the team (e.g., exact batching requirements, atomicity needs).
- **Propose Initial Design:** Draft the architecture document and present it for review.
- **Iterate:** Incorporate feedback and refine the design.
- **Develop Roadmap:** Once the design is approved, create the detailed implementation plan.
