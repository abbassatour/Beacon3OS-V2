
# BeaconVault (Powered by BeaconOS) × UGREEN AI NAS Developer Friction Log (Executive Summary)

> Technical Evaluation & Developer Experience Log for UGREEN Hackathon 2026 on AI NAS (Open Innovation Track)
> Target Platform: UGREEN AI NAS iDX6011 Pro • Operating System: UGOS • Skill Standard: Uliya

---

### Structured Friction Log Entries (Engineering Evaluation)

#### 1. UGOS Container Sandbox Isolation vs. `ug-filesystem` CLI Path Resolution

- **Specific Task Attempted:** Access and persist user document scans, medication image inputs, and SQLite vault snapshots inside the authorized NAS shared volume `/volume1/beacon_vault/` from within the BeaconVault containerized skill runtime.
- **Steps Taken:** Implemented declarative tool calls invoking `ug-filesystem` CLI utilities using standard absolute Linux filesystem paths (`/volume1/beacon_vault/inbox/prescription.jpg`) inside the UGOS container environment.
- **Expected vs. Actual Result:** Expected `ug-filesystem` to seamlessly resolve authorized absolute volume paths. Actual result: The UGOS container isolation layer maps storage volumes to localized mount points (`/mnt/storage/...`), resulting in `ENOENT (No such file or directory)` and permission validation rejections when paths were passed without explicit container-relative translation.
- **Severity Rating:** High (Blocks autonomous document ingestion and local file indexing).
- **Workaround Used:** Engineered an internal path canonicalization utility inside the BeaconVault coordinator that intercepts all filesystem parameters, resolves UGOS container mount aliases dynamically via environment variables (`UGOS_VOLUME_ROOT`), and enforces strictly validated relative sandbox paths.
- **Actionable Suggestion:** The UGOS Skill Developer Package and `ug-filesystem` CLI should natively support virtual volume aliases (e.g., `volume://beacon_vault/inbox/...`) that automatically resolve host-to-container volume bindings regardless of deployment morphology.

---

#### 2. NPU/GPU Compute Contention on iDX6011 Pro During Concurrent Agent Ingestion

- **Specific Task Attempted:** Simultaneously transcribe an incoming voice memo via `knowledge-agent` (`ug-knowledge`) while analyzing a high-resolution prescription label using local on-device VLM via `document-agent` (`ug-document`).
- **Steps Taken:** Dispatched asynchronous concurrent Uliya delegation calls (`delegate: document-agent` and `delegate: knowledge-agent`) targeting on-device acceleration runtimes on the iDX6011 Pro hardware.
- **Expected vs. Actual Result:** Expected hardware acceleration queues to interleave execution smoothly within the sub-200ms latency budget. Actual result: Concurrent execution saturated on-device GPU/NPU memory channels, triggering a thread lockout that degraded OCR inference time from ~140ms to over 2100ms and dropped incoming WebSocket keep-alive frames from the client nodes.
- **Severity Rating:** High (Violates real-time eyes-free accessibility thresholds).
- **Workaround Used:** Implemented a prioritized asynchronous FIFO queue in the BeaconVault coordinator skill. Time-critical visual inspections (medications, walking hazards) are assigned P0 priority with exclusive hardware access, while background voice memo indexing and vector embedding generation are deferred to P2 background threads.
- **Actionable Suggestion:** UGOS should expose a hardware QoS (Quality of Service) scheduler API within the Skills Hub runtime allowing skills to flag latency-critical tasks versus background batch tasks for the onboard NPU/GPU.

---

#### 3. Streamable HTTP (Spec 2025-11-25) Local LAN Relay vs. Docker Host Bridge Isolation

- **Specific Task Attempted:** Stream real-time camera frames and voice intent payloads between mobile BeaconOS nodes and the BeaconVault NAS Hub over local Wi-Fi with sub-2ms network latency.
- **Steps Taken:** Deployed the BeaconVault MCP Server using unified Streamable HTTP (`POST /mcp` for JSON-RPC 2.0 and `GET /mcp` for Server-Sent Events) inside a standard UGOS Docker bridged network container.
- **Expected vs. Actual Result:** Expected mobile and smart TV clients on the local subnet to automatically discover and connect to the NAS via UDP multicast discovery (`port 8491`). Actual result: Docker default bridge networking isolated the container from host subnet broadcasts; UDP probe packets were dropped, and browser clients on the LAN triggered CORS/DNS rebinding security exceptions (`403 Forbidden`).
- **Severity Rating:** High (Prevents zero-configuration local device discovery).
- **Workaround Used:** Configured the BeaconVault service container with host networking mode (`--net=host`), bound the internal server to `0.0.0.0:8000`, and implemented FastAPI middleware that explicitly validates and whitelists private LAN IP spaces (`192.168.x.x`, `10.x.x.x`) against DNS rebinding rules.
- **Actionable Suggestion:** UGREEN Skills Hub should provide a built-in local mDNS / SSDP relay service that automatically advertises installed UGOS skills to the local area network without requiring full host network container privileges.

---

#### 4. Uliya Declarative Orchestration (`SKILL.md`) vs. Non-Visual Fallback Handling

- **Specific Task Attempted:** Handle ambiguous, low-lighting, or obscured medication prescription photos purely through spatial voice prompts without prompting a visual UI confirmation dialogue.
- **Steps Taken:** Declared an autonomous tool-calling pipeline in `SKILL.md` with fallback branches when `ug-document` confidence scores fall below 0.70.
- **Expected vs. Actual Result:** Expected the agent to invoke the `waiting_user` state with an audio-first clarification intent. Actual result: Standard Uliya agent templates expect visual interactive forms for `waiting_user` states, which leaves a blind user stranded with no auditory feedback or clarification prompt.
- **Severity Rating:** Medium (Accessibility UX barrier for non-visual workflows).
- **Workaround Used:** Structured the BeaconVault coordinator to intercept low-confidence OCR results before triggering a raw `waiting_user` state, compiling an explicit conversational audio prompt (e.g., *"The label is partially obscured by glare; please tilt the bottle slightly right and hold steady"*) transmitted immediately over the speech audio bus.
- **Actionable Suggestion:** Expand the Uliya Skill Development Standard to officially specify multimodal interaction profiles (`interaction_mode: voice_first | headless | visual`), enabling declarative audio prompts during parameter clarification states.

---

#### 5. `ug-surveillance` RTSP Event Latency vs. Immediate Acoustic Actuation

- **Specific Task Attempted:** Detect an arriving visitor or delivery courier at the front entrance via security camera feeds and immediately trigger the living room TV alert and phone acoustic locator in < 200ms.
- **Steps Taken:** Configured `security-agent` to query `ug-surveillance` event timelines and trigger cross-device actuation via the BeaconVault local WebSocket relay upon motion detection.
- **Expected vs. Actual Result:** Expected near-instantaneous event notification. Actual result: Standard polling intervals against the surveillance timeline created a 1.5 to 3.0 second latency window between physical visitor arrival and skill invocation, causing delayed verbal announcements.
- **Severity Rating:** Medium (Compromises real-time perimeter awareness).
- **Workaround Used:** Bypassed batch timeline polling by hooking directly into the camera's native RTSP webhook / MQTT motion stream within UGOS, using `ug-photo` only for on-demand keyframe scene description upon trigger.
- **Actionable Suggestion:** Provide native event-driven webhook callbacks within `ug-surveillance` so that skills can subscribe to live camera motion events with zero-polling overhead.

---

#### 6. Air-Gapped Network Severance vs. Hybrid Ledger Attestation

- **Specific Task Attempted:** Maintain full assistive intelligence, alarm scheduling, local document retrieval, and emergency SOS radar when external WAN internet connectivity is severed.
- **Steps Taken:** Cut WAN uplink on the local Wi-Fi router during an emergency SOS test and verified whether BeaconVault executes autonomous offline failover.
- **Expected vs. Actual Result:** Expected all local features to operate uninterrupted, buffering non-custodial telemetry in local SQLite. Actual result: Certain backend libraries attempted remote DNS lookups and stalled async execution loops, resulting in a 5-second UI freeze before local fallback kicked in.
- **Severity Rating:** Medium (Safety-critical resilience issue).
- **Workaround Used:** Enforced aggressive 200ms timeouts on all external networking calls, isolated the Drift SQLite local vault engine, and ensured that emergency radar signals dispatch across local LAN sockets immediately while queueing external blockchain ledger attestations for background retry.
- **Actionable Suggestion:** Include an explicit "Air-Gapped Local Verification Mode" in the UGREEN Skill Creation Tool to test skills under total simulated WAN disconnectivity prior to submission.
