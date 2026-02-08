This revised **Low-Level Design (LLD)** includes the comprehensive **Session Parameter Map**. These parameters act as the "memory" of the bot, ensuring data collected in the Flow is available to the Playbook and the backend services without redundancy.

---

## 1. Solution Architecture Overview

The project follows a **Hybrid Bot Architecture**. A deterministic **Flow** handles secure authentication, while a generative **Playbook** manages the dynamic conversation of scheduling.

---

## 2. Session Parameter Map (The "Memory" of the Bot)

These parameters are stored in the `$session.params` object and are passed between services to maintain state.

| Parameter Name | Data Type | Originating Source | Purpose |
| --- | --- | --- | --- |
| **`patient_id`** | String (Regex) | `AuthenticationFlow` | The unique ID (e.g., P123) used for BigQuery lookups. |
| **`ssn4`** | String (Regex) | `AuthenticationFlow` | Last 4 digits of SSN for identity verification. |
| **`authenticated`** | Boolean | Cloud Function | Set to `true` if ID/SSN match; controls the Flow-to-Playbook transition. |
| **`patient_name`** | String | BigQuery | Fetched during validation; used for the Playbook's personalized greeting. |
| **`selected_dept`** | String | Playbook | Captured when the user selects Cardiology, Radiology, etc. |
| **`selected_slot_id`** | String | Playbook | The ID of the slot the user chose from the announced list. |
| **`retry_count`** | Integer | `AuthenticationFlow` | Tracks failed login attempts to trigger a "Call Support" exit after 3 tries. |

---

## 3. End-to-End Service Data Flow

### Phase A: Identification & Secure Authentication (Flow)

1. **Greeting:** The bot identifies as the "Appointment Scheduling Bot" and triggers the `intent.book_appointment`.
2. **Form Filling:** The bot collects `patient_id` and `ssn4`. These are saved to session parameters immediately.
3. **Validation Call:** Dialogflow sends these parameters to the Cloud Function `/validatePatient` endpoint.
4. **Database Query:** The function queries the `master_patient` table.
5. **Parameter Update:** The function returns `authenticated: true` and `patient_name: "John Doe"`. Dialogflow updates the session parameters with these values.

### Phase B: The Handoff (Transition Logic)

1. **Condition Check:** In the `ValidationResultPage`, a transition route checks: `IF $session.params.authenticated = "true"`.
2. **Playbook Invitation:** The bot transitions to the **Appointment_Booking_Playbook**.
3. **Input Mapping:** The `patient_id` and `patient_name` are mapped as **Input Parameters** to the Playbook, ensuring the LLM knows who it is talking to.

### Phase C: Scheduling & Conflict Resolution (Playbook)

1. **Personalized Greeting:** The Playbook uses `$session.params.patient_name` to say: *"Hello John, I've verified your records. Which department do you need today?"*
2. **Slot Retrieval:** Upon department selection, the Playbook calls the `GetSlotsTool`.
3. **Atomic Booking:** When a slot is picked, the Playbook calls the `BookingTool`. If a `409 Conflict` occurs (meaning someone else grabbed the slot), the Playbook logic handles the apology and re-lists available times.

---

## 4. Configuration & Naming Standards

### Google Cloud Function (Node.js)

* **Resource Name:** `appointment-bot-backend`
* **Entry Point:** `appointmentWebhook`
* **URL:** `https://[REGION]-[PROJECT].cloudfunctions.net/appointmentWebhook/`

### Dialogflow CX Components

| Configuration Item | Suggested Name | Note |
| --- | --- | --- |
| **Flow Name** | `AuthenticationFlow` | Contains Start, CollectDetails, and Validation pages. |
| **Playbook Name** | `Appointment_Booking_Playbook` | Generative agent with Goal-based instructions. |
| **Entity (patient_id)** | `patient_id` | Regexp: `^P[0-9]{3,8}$`. |
| **Entity (ssn4)** | `ssn4` | Regexp: `^[0-9]{4}$`. |
| **Webhook Name** | `PatientValidationWebhook` | Points to the `/validatePatient` sub-path. |

---

## 5. Important Implementation Notes

* **Parameter Passing:** In the Playbook settings, ensure "Track Session Parameters" is enabled so the `selected_dept` and `selected_slot_id` are available for debugging.
* **Atomic Updates:** The BigQuery update logic must be atomic (`UPDATE ... WHERE status = 'Available'`) to ensure data integrity in high-volume environments.
* **Data Privacy:** Use Dialogflow CX **Data Redaction** for the `ssn4` parameter to ensure sensitive data is not stored in plain text logs.

**Next Step:** Would you like me to generate the **BigQuery SQL Table schemas** to match these session parameters and department lists?
