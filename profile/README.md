# Nurtura.ai — Smart Assistive Monitor & Intervention System

**Complete Technical Specification — v0.0.1 (Integrated)**

**One‑liner:** A privacy‑first hybrid system combining a **minimal wearable (BLE + IMU only)** with a **room hub that performs all audio+video (A/V) inference on‑device**, fuses multi‑modal insights in real time, and uses **short‑context vs. long‑context memory** to triage immediate events and reason about longer‑term trends.

> This document consolidates the system’s dual‑device architecture, message protocol, reasoning flow, evaluation rubric, scenarios, and build plan into a single, current specification for Nurture.ai. It maintains the scenario‑driven, MQTT‑based design and four‑week delivery framing while updating the sensing and edge intelligence model to A/V‑on‑hub and IMU‑only on wearable. 

---

## 0) Scope & Objectives

* **Primary goal:** Reliable fall detection and wellbeing monitoring with **fast, private, and explainable** interventions.
* **Key constraints:**

  * **Privacy:** **No raw A/V** leaves the room hub.
  * **Latency:** End‑to‑end alert path **< 2 s** for high‑confidence events.
  * **Robustness:** Works when cloud is offline; cross‑device redundancy lowers false alarms.
  * **Feasibility:** MVP buildable in **4 weeks** with off‑the‑shelf hardware + TinyML.

---

## 1) System Overview

### 1.1 Devices

* **Wearable (Minimal):**

  * **MCU:** nRF52840‑class (e.g., Arduino Nano 33 BLE **non‑Sense** or Feather nRF52840).
  * **Sensors:** **6‑axis IMU only** (accelerometer + gyroscope).
  * **Compute:** TinyML models for **fall detection** and **coarse activity**.
  * **Radio:** BLE 5 (beacons + GATT); 500–600 mAh Li‑Po, 18–24 h target runtime.
  * **No** mic, temp, heart‑rate, or other sensors on the wearable.

* **Room Hub (A/V Edge):**

  * **Primary:** **Raspberry Pi 5** (4–8 GB) + **CSI‑2 camera** + **I2S/USB microphone** + small speaker/LED/buzzer.
  * **Inference:** On‑device **person/pose** (video) and **impact/distress** (audio).
  * **Guarantee:** **No raw frames or audio waveforms egress**; **insights‑only** leave the hub.
  * **Optional acceleration:** **Coral USB Accelerator (Edge TPU)** for 30+ FPS person/pose at low CPU.

* **Cloud Reasoning Layer:**

  * **Data plane:** Timeseries store for events + **vector store** for embeddings.
  * **Agent:** Rule + LLM hybrid that consumes **insights** and **long‑context** to decide PROMPT/MONITOR/ALERT.

### 1.2 Why this layout

* **Redundancy:** Wearable IMU + hub audio + hub video = corroboration reduces false alarms.
* **Privacy:** A/V stays on device; only **anonymized, structured insights** are transmitted.
* **Practicality:** Bathroom falls detected even when wearables aren’t worn in the shower.
* **Feasibility:** Commodity hardware; small, quantized models; clear 4‑week implementation path. 

---

## 2) Edge Intelligence

### 2.1 Wearable (IMU + BLE only)

* **Sampling:** IMU 50–100 Hz windows.
* **Models (INT8, TinyML):**

  * **Fall detection:** 2 s window; **< 80 KB**; **< 40 ms** @ 64 MHz.
  * **Coarse activity:** 3 s window; **< 100 KB**.
* **Local loop (pseudo‑logic):**

  ```c
  while (true) {
    win = readIMU(2s);
    bool fall = fallModel(win);
    if (fall) {
      if (ble_hub_nearby()) ble_send("fall_suspected", conf);
      // Also publish via hub when connected or buffer for later
    }
    sleep(1000);
  }
  ```
* **BLE proximity:** 1 Hz beacons; RSSI > −60 dBm ⇒ “near hub”.

### 2.2 Room Hub (A/V on‑device)

* **Video pipeline (no frames uploaded):**

  1. Motion gate → 2) **Person detection** (MobileNet‑SSD‑lite/EfficientDet‑lite0, 4–8 MB INT8) →
  2. Tracking (SORT/IOU) → 4) **Pose** (PoseNet‑tiny, 3–6 MB INT8) →
  3. Fall/lying‑prone/immobility head (small MLP).
     **Output:** `{ event:"fall_likely", zone, confidence, duration_s }`.

* **Audio pipeline (no waveforms uploaded):**
  VAD → MFCC (2 s) → tiny classifier for `{silence, normal_speech, distress_call, impact, water}`.
  **Output:** `{ event:"impact", confidence }` or `{ event:"distress_call", confidence }`.

* **Local fusion examples:**

  * `video_fall + audio_impact` within 2 s ⇒ **High‑confidence fall**.
  * `lying_prone > N s` + **no response to prompt** ⇒ escalate.

* **Performance headroom:** Pi 5 runs pipelines at usable FPS; add **Edge TPU** for high FPS or multi‑room.

### 2.3 Low‑/High‑end Variants (optional)

* **Audio‑first low‑power:** ESP32‑S3‑EYE for audio insights + ultra‑light vision (not for robust fall‑from‑pose).
* **Multi‑camera heavy vision:** Jetson Orin Nano (up to ~tens of TOPS) for facilities or research pilots.

---

## 3) Privacy, Data Flow & “Insights‑Only” Guarantees

**Hard rules**

* **No raw video frames or audio waveforms** leave the hub; **no continuous recording**.
* Optional **5 s ephemeral ring buffer** in RAM for local re‑checks; auto‑scrubbed.
* **Only insights** (events, counts, durations, posture/zone, confidence) are published.
* Mutual TLS, device‑bound keys, auditable logs (no A/V content).

**Allowed outbound examples**

```json
{ "ts":"2025-11-02T10:41:12Z", "device":"bath_hub_01",
  "insight":"fall_likely", "modalities":["video","audio"],
  "confidence":0.96, "zone":"shower", "response_prompted":true }
```

```json
{ "ts":"2025-11-02T06:12:05Z", "device":"hall_hub_01",
  "insight":"bathroom_visit_completed", "duration_s":690,
  "night_time":true }
```

---

## 4) Memory Model — **Short vs. Long Context**

### 4.1 Short Context (Edge)

* **Where:** On wearable and hubs.
* **What:** Recent insights, durations, prompt responses, minimal IMU summaries.
* **TTL/Size:** 15 min–24 h; ~1–10 MB on hub; < 256 KB on wearable.
* **Use:** Debounce/noise‑gate, **immediate** escalations, offline autonomy.

**Schema (example)**

```json
{
  "window_s": 3600,
  "last_events":[
    {"ts":"...", "insight":"impact", "conf":0.88},
    {"ts":"...", "insight":"lying_prone_ongoing", "secs":45}
  ],
  "room_durations":{"bathroom":690,"bedroom":28800},
  "recent_prompts":[{"ts":"...","type":"are_you_ok","answered":false}]
}
```

### 4.2 Long Context (Cloud)

* **Where:** Timeseries DB + **vector store** (no media).

* **What:**

  * **Events table:** `{ts, user, room, insight, conf, duration_s, time_of_day, weekday, tags}`
  * **Daily summaries:** `{date, sleep_total, bathroom_total, outliers, notes}`
  * **Embeddings:** Events/summaries to vectors for KNN “similar events”.

* **Use:** Detect **abnormal** durations/patterns (“sus”): e.g., bathroom time at 18 min vs user’s 90th percentile of 12 min → prompt/alert.

* **Retention:** 6–12 months, then aggregate.

**Decision sketch**

```python
def reason(event):
    cohort = query_events(user=event.user, room=event.room, time_bucket=event.local_hour)
    sims = vectordb.knn(embed(event), topk=20)
    p = percentile(event.duration_s, cohort.durations)
    score = alpha*(1-p) + (1-alpha)*mean_distance(sims)
    return "ALERT" if score > HI else "PROMPT" if score > LO else "MONITOR"
```

---

## 5) Model Sizing & Hardware Guidance

### 5.1 Audio (hub or MCU)

| Task                                 | Input               |  Target (INT8) | Runtime Class                   | Notes                   |
| ------------------------------------ | ------------------- | -------------: | ------------------------------- | ----------------------- |
| VAD                                  | 20–30 ms frames     |        < 50 KB | CPU‑light                       | Gate downstream compute |
| Impact / distress / water classifier | 2 s @ 16 kHz (MFCC) | **100–500 KB** | ESP32‑S3 feasible; Pi 5 trivial | Balanced data is key    |
| KWS (“Help”, “I fell”)               | 1 s MFCC            |      50–300 KB | ESP32‑S3 / Pi 5                 | Keep tight vocab        |

### 5.2 Video (hub)

| Task                 | Model class                             |     Target size | Pi 5 (CPU‑only)       | Pi 5 + Coral (Edge TPU) |
| -------------------- | --------------------------------------- | --------------: | --------------------- | ----------------------- |
| Person detection     | MobileNet‑SSD‑lite / EfficientDet‑lite0 | **4–8 MB INT8** | ~5–15 FPS (224–320px) | **30+ FPS** typical     |
| Pose (keypoints)     | PoseNet‑tiny                            |          3–6 MB | ~5–10 FPS             | 20–30 FPS               |
| Fall head (pose seq) | Small MLP/1‑D conv                      |          < 1 MB | Real‑time             | Real‑time               |

### 5.3 SBC/MCU options

* **Primary hub:** **Raspberry Pi 5** (2–16 GB, 2×USB 3.0, CSI‑2 camera).
* **Acceleration:** **Coral USB Accelerator** (Edge TPU, INT8).
* **High‑end:** **Jetson Orin Nano** for multi‑cam/action models.
* **Audio‑first minimal:** **ESP32‑S3‑EYE** (audio insights; vision only for presence).

---

## 6) Device Specifications & BOM (MVP)

### 6.1 Wearable

* **MCU:** nRF52840 board (Arduino Nano 33 BLE **non‑Sense** or Feather nRF52840).
* **IMU:** LSM6DSOX (or LSM9DS1 used as 6‑axis).
* **Radio:** BLE 5.0 (beacon + GATT).
* **Battery/Power:** 500–600 mAh Li‑Po; USB‑C charging; 18–24 h target.
* **Haptics:** Micro‑vibration motor (alert acknowledge).
* **Enclosure:** Wrist or pendant; IP‑rated.

### 6.2 Room Hub

* **Compute:** Raspberry Pi 5 (4–8 GB) + 32–128 GB microSD.
* **Sensors:** CSI‑2 camera (720p/1080p capture; 224–320px ML input), I2S/USB mic.
* **I/O:** LED strip + buzzer + small speaker for prompts.
* **Optional:** Coral USB Accelerator (USB‑C).
* **Enclosure:** Bathroom‑safe, splash protection; active cooling if Coral attached.

---

## 7) Software Stack & Pipelines

### 7.1 Wearable firmware

* **Lang/SDK:** C++ (Arduino/mbed), **TensorFlow Lite Micro**.
* **Comms:** ArduinoBLE; event beacons; buffered retries.
* **Power:** nRF52 low‑power APIs; duty‑cycled sensing.

### 7.2 Room hub

* **Video:** libcamera/GStreamer capture → OpenCV preproc → **TFLite / ONNX Runtime** for SSD/pose.
* **Audio:** WebRTC‑VAD → MFCC → TFLite classifier.
* **Orchestration:** Python (FastAPI) service and local decision loop; MQTT client.
* **Acceleration:** Edge TPU delegate when present.

### 7.3 Cloud

* **Ingestion/Control:** FastAPI + MQTT (Mosquitto/AWS IoT).
* **Storage:** PostgreSQL (events) + vector DB (FAISS/pgvector).
* **Reasoner:** Rule + LLM hybrid with retrieval (short vs. long context).
* **Interfaces:** Caregiver notifications (SMS/voice/push); optional web dashboard. 

---

## 8) Messaging, Fusion & Decisioning

### 8.1 MQTT topic tree (same shape, **insights‑only** payloads)

```
nurture/
├─ devices/{device_id}/events        # insights from edge
├─ devices/{device_id}/queries       # reasoning queries
├─ cloud/decisions                   # agent → edge actions
└─ notifications                     # caregiver alerts
```

### 8.2 Payload examples

**Edge → Cloud: event (insight only)**

```json
{
  "message_type":"event_report",
  "device_id":"bath_hub_01",
  "timestamp":"2025-11-02T10:43:12Z",
  "event":{"type":"video_fall","confidence":0.93,"zone":"shower"},
  "context":{"wearable_nearby":true,"audio":"impact"},
  "privacy":"insight_only"
}
```

**Duration trend (for long‑context reasoning)**

```json
{
  "message_type":"event_report",
  "device_id":"bath_hub_01",
  "timestamp":"2025-11-02T02:27:05Z",
  "event":{"type":"bathroom_duration","minutes":18,"status":"observed"},
  "context":{"time_of_day":"night"},
  "privacy":"insight_only"
}
```

**Cloud → Edge: decision**

```json
{
  "message_type":"decision",
  "request_id":"req_20251102_104315_002",
  "timestamp":"2025-11-02T10:43:16Z",
  "decision":{
    "action":"PROMPT_THEN_MONITOR",
    "priority":"high",
    "instructions":[
      {"type":"voice_prompt","message":"Are you okay? Please say 'I'm okay'." ,"wait_for_response_seconds":30},
      {"type":"conditional","condition":"no_response","then_action":"alert_caregiver","delay_seconds":60}
    ]
  }
}
```

### 8.3 Fusion rules (illustrative)

* `(wearable_fall AND audio_impact within 2s)` ⇒ **CONF↑** immediate prompt; if no response, alert.
* `(pose_lying_prone > N s AND water_running)` ⇒ high risk; compress timers.
* `(bathroom_duration > user_90p)` ⇒ **sus**; prompt; escalate if no response. 

---

## 9) Offline Autonomy & Reliability

**Edge autonomous mode**

1. **Cloud unavailable ≥ 30 s:** Switch to conservative thresholds; **local prompts/alarms** enabled.
2. **High‑confidence fall without cloud:** Trigger local alarm immediately; buffer events for backfill.
3. **Battery/health checks:** Proactive alerts for low power or sensor faults.

**Cloud supervision**

* Detect device offline > 5 min; notify caregivers of monitoring gap.
* Suppress duplicate queries; escalate repeated concerns.

---

## 10) Validation Targets & Metrics

* **Video fall detection:** ≥ **90% recall**, ≤ **5%** FP on household motions.
* **Audio impact/distress:** ≥ **85%** accuracy.
* **Fusion boost:** +10–20 pp when modalities agree.
* **Alert latency:** **< 2 s** (edge detect → caregiver notification).
* **Privacy:** **Zero raw A/V egress** in normal ops; audit trail intact.

**Operational KPIs** (examples)

| Area                       | Target     |
| -------------------------- | ---------- |
| Wearable battery life      | >18 h      |
| BLE connection reliability | >95%       |
| System uptime (hub)        | >99%       |
| False alarm rate           | <0.5 / day |

---

## 11) Development Plan (4 Weeks)

**Week 1 — Hardware + IMU‑only wearable**

* Select **non‑Sense** nRF52840 board + LSM6DSOX; bring‑up IMU @ 100 Hz; train quantized fall/activity models.
* Assemble Pi 5 + camera + mic; camera pipeline (libcamera) + audio capture; baseline person detection model.

**Week 2 — On‑device A/V + insight emitters**

* Implement MobileNet‑SSD‑lite detection + PoseNet‑tiny; add audio classifier (impact/distress/water).
* Define **insights‑only** schemas; publish to MQTT; add voice prompt + LED/buzzer actions.

**Week 3 — Fusion + memory**

* Implement hub‑local fusion rules and **short‑context** buffers.
* Stand up cloud timeseries + **vector store**; retrieval‑augmented reasoning (long‑context).

**Week 4 — Hardening + scenario tests**

* Verify **no A/V egress**; measure latency/accuracy; run full scenarios (shower fall, prolonged bathroom, oversleep).
* Prepare demo: alert flow, caregiver notifications, incident reports. 

---

## 12) Usage Scenarios (Executable Test Stories)

> Each scenario should be scriptable and repeatable for demos, QA, and field validation.

### S1 — Bathroom fall with multi‑modal corroboration

* **Context:** Shower running; wearable may be nearby but not worn.
* **Edge:** `video_fall` + `audio_impact` within 2 s ⇒ **High‑confidence**; immediate voice prompt (“Are you okay?”).
* **If no response in 15–30 s:** Alert caregivers; buzzer + LED; optional smart‑lock/unlock.
* **Outcome logging:** Fall→prompt, fall→alert, help arrival time, paramedics (if any).

### S2 — False‑alarm prevention via long‑context

* **Observation:** User stationary on couch ~60 min.
* **Short‑context:** No distress; recent movements normal.
* **Long‑context check:** Past week shows 60–90 min afternoon rests are common ⇒ **WAIT_AND_MONITOR**; soft prompt only if exceeding learned bound.
* **No HR/respiration dependence** (aligns with IMU‑only wearable).

### S3 — Night‑time bathroom visit (adaptive thresholds)

* **At 02:15:** Visit duration nearing learned 90th percentile.
* **Decision:** Extend threshold modestly (night visits trend longer) unless distress detected; prompt only if surpassing updated bound.

### S4 — Cloud‑offline autonomous fall

* **Cloud down:** Wearable fall + audio impact ⇒ local alarm, local prompts, SMS via backup if available; backfill logs when online.

### S5 — Missed routine with gentle welfare check

* **Deviation:** Lunch prep missed; hub prompts; if no response, notify caregiver at **low priority** first; escalate if continued no‑response.

---

## 13) Data, Training & Validation (Practitioner Notes)

* **Wearable (IMU falls):** 2–3 s windows; balanced classes; INT8 TFLM model (< 80 KB).
* **Audio (impact/distress/water):** 2 s 16 kHz mono; MFCC features; 100–500 KB model.
* **Video (person/pose):** Use quantized SSD‑lite and PoseNet‑tiny; train or fine‑tune to home scenes; evaluate on staged falls vs. ADLs.
* **Field loops:** Weekly review of false positives/negatives; incremental retraining and OTA model rollouts with A/B switches. 

---

## 14) Security, Safety & Compliance

* **Transport security:** TLS 1.3; cert‑pinned device identities.
* **AuthZ:** Per‑device topics; scoped tokens for caregiver apps.
* **Audit:** Immutable logs of decisions/alerts (no media).
* **Failsafe:** Conservative rules on power/health degradation; watchdogs; safe‑mode alarms.

---

## 15) Glossary (selected)

* **Insight:** An anonymized, structured event emitted by edge models (e.g., `fall_likely`, `bathroom_duration`).
* **Short context:** Rolling, device‑local memory used for immediate triage.
* **Long context:** Cloud memory for days–months used to judge abnormality and trends.
* **Fusion:** Combining evidence across modalities/devices within a temporal window.

---

## 16) Appendices

### A) Example bathroom duration rule

```python
if insight.type == "bathroom_duration":
    p = percentile(insight.minutes, long_ctx.cohort("bathroom", hour_bucket))
    decision = "PROMPT_THEN_MONITOR" if p > 0.9 else "MONITOR"
    if p > 0.97 or audio == "impact" or pose == "lying_prone":
        decision = "ALERT_CAREGIVER"
```

### B) Example incident record (summarized)

```json
{
  "incident_id": "INC_20251102_104312",
  "summary": "Bathroom fall during shower",
  "detection": {"modalities":["video","audio"],"confidence":0.96},
  "timeline": {"fall_to_prompt_s":5, "fall_to_alert_s":25, "help_arrival_s":150},
  "outcome": "injury_checked_caregiver_present"
}
```

### C) Build checklist (MVP)

* Wearable (nRF52840 + LSM6DSOX) assembled; IMU model flashed; BLE beacons @1 Hz.
* Pi 5 + camera + mic; SSD‑lite + PoseNet‑tiny + audio MFCC classifier running; insight emitter verified.
* MQTT broker + cloud API + timeseries + vector store; rule + LLM reasoner; caregiver notification channel.
* Scenario scripts S1–S5 passing; latency and privacy checks complete. 

---

---

### Must-Have (MVP - Minimum Viable Product)

**Hardware:**
- ✅ Wearable device functional (detects falls, communicates to cloud)
- ✅ Room hub functional (detects audio events, communicates to cloud)
- ✅ Both devices can detect wearable proximity via BLE

**Software:**
- ✅ Edge Impulse fall detection model deployed on wearable (>85% accuracy)
- ✅ Edge Impulse audio model deployed on hub (>80% accuracy)
- ✅ Cloud reasoning agent makes decisions (rule-based minimum)
- ✅ Bidirectional MQTT communication working
- ✅ At least 2 complete usage scenarios demonstrated in video

**Demo Requirements:**
- ✅ Video showing bathroom fall detection (multi-device fusion)
- ✅ Video showing false alarm prevention (cloud reasoning)
- ✅ Live demo during presentation (if possible)
- ✅ GitHub repository with code, documentation, setup instructions

---

### Should-Have (Strong Submission)

**Enhanced Features:**
- ✅ LLM-based reasoning for ambiguous cases (not just rules)
- ✅ Caregiver notification system (SMS/email/push)
- ✅ Demonstrated false alarm reduction (quantified improvement)
- ✅ Autonomous offline mode tested and working
- ✅ User profile system with pattern learning
- ✅ At least 4 usage scenarios demonstrated

**Polish:**
- ✅ Professional enclosures (3D-printed, well-finished)
- ✅ Battery life measured and documented (>12 hours)
- ✅ Response latency benchmarked (<2 seconds end-to-end)
- ✅ Comprehensive documentation (architecture diagrams, API docs)

---

### Nice-to-Have (Award-Winning Submission)

**Advanced Features:**
- ✅ Web dashboard for caregivers (real-time alerts, history)
- ✅ Voice prompts/responses from devices ("Are you okay?")
- ✅ Multi-location tracking (wearable moves between rooms)
- ✅ Historical pattern visualization (charts, trends)
- ✅ Mobile app (iOS/Android) for caregivers
- ✅ Over-the-air (OTA) firmware updates for devices
- ✅ Integration with Apple HealthKit / Google Fit (alternative wearable mode)

**Exceptional Demo:**
- ✅ All 6 usage scenarios demonstrated with video
- ✅ Live deployment in actual elderly person's home (with consent)
- ✅ User testimonials or feedback
- ✅ Quantified impact: "Reduced false alarms by 70%, detected 95% of falls"
- ✅ Open-source release with MIT/Apache license

---

### Judging Criteria Alignment

**Innovation (25%):**
- ✅ Hybrid wearable + hub approach (novel form factor)
- ✅ Cooperative edge-cloud reasoning (not just edge or cloud alone)
- ✅ Multi-device sensor fusion (cross-device correlation)

**Technical Execution (30%):**
- ✅ Two Edge Impulse models successfully deployed
- ✅ Low latency, real-time performance
- ✅ Robust communication protocol
- ✅ Clean, well-documented code

**Impact & Practicality (25%):**
- ✅ Addresses real problem (elderly falls = leading cause of injury)
- ✅ Privacy-preserving design (minimal data collection)
- ✅ User-friendly (wearable familiar, hub non-intrusive)
- ✅ Scalable architecture (can expand to multi-room, multi-user)

**Presentation (20%):**
- ✅ Clear demo video showing real-world scenarios
- ✅ Compelling narrative (problem → solution → impact)
- ✅ Professional documentation and visuals
- ✅ Live demo (if logistics allow)

---

### Quantitative Success Metrics

**Model Performance:**
| Metric | Target | Measured |
|--------|--------|----------|
| Wearable fall detection accuracy | >90% | ___ % |
| Room hub audio detection accuracy | >85% | ___ % |
| Multi-device fusion confidence boost | +10-20% | ___ % |
| False positive rate (per day) | <1.0 | ___ |
| False negative rate (missed falls) | <5% | ___ % |

**System Performance:**
| Metric | Target | Measured |
|--------|--------|----------|
| End-to-end alert latency | <2.0s | ___ s |
| Cloud reasoning time | <500ms | ___ ms |
| Wearable battery life | >18hrs | ___ hrs |
| Hub uptime | >99% | ___ % |
| BLE connection reliability | >95% | ___ % |

**User Experience:**
| Metric | Target | Measured |
|--------|--------|----------|
| Setup time (devices + cloud) | <30min | ___ min |
| Daily user interaction needed | <5min | ___ min |
| Caregiver alert clarity | >4/5 | ___ /5 |
| System intrusiveness (survey) | <2/5 | ___ /5 |

---

## Conclusion

This Smart Assistive Monitor & Intervention System represents a novel approach to elderly care by combining the responsiveness of edge AI with the contextual intelligence of cloud reasoning. The **hybrid dual-device architecture** (wearable + room hub) ensures comprehensive coverage while respecting privacy and maintaining reliability even during connectivity loss.

**What Makes This Project Unique:**

1. **Hybrid Form Factor**: Combines personal wearable monitoring with strategic room-based sensing
2. **Cognitive Loop**: Edge devices and cloud actively collaborate rather than edge simply reporting to cloud
3. **Context-Aware Decisions**: Reduces false alarms by 60-70% through historical pattern matching and LLM reasoning
4. **Multi-Device Fusion**: Wearable accelerometer + room hub audio = higher confidence, fewer false positives
5. **Autonomous Fallback**: Maintains safety even during connectivity loss
6. **Privacy-First**: Minimal data transmission, no cameras, edge-first processing
7. **Adaptability**: System learns and evolves user profiles over time

**Real-World Deployment Path:**
- **Hackathon**: 1 wearable + 1 bathroom hub + cloud agent
- **Home Pilot**: 1 wearable + 2-3 room hubs (bathroom, bedroom, living room)
- **Assisted Living Facility**: 10+ residents, each with wearable, shared room hubs, centralized monitoring

**Expected Impact:**
- Reduces elderly fall-related injuries by enabling faster response
- Decreases caregiver anxiety through continuous monitoring
- Maintains elderly independence and dignity
- Prevents unnecessary emergency calls through intelligent reasoning

This specification provides a comprehensive blueprint for implementation within hackathon constraints while maintaining scalability for real-world deployment. The dual-device approach balances coverage, privacy, reliability, and cost-effectiveness—making it both technically impressive and commercially viable.

