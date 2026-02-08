Call Flow 1: Appointment Booking – Slot Selected From Available Options
1.	Inbound Call & Greeting
1.1 Patient calls the healthcare contact number.
1.2 Google IVA Playbook bot greets the caller and identifies itself as the Appointment Scheduling Bot.
1.3 IVA asks the caller to confirm if they are calling to book an appointment.
2.	Intent Confirmation
2.1 Patient confirms they want to book an appointment.
2.2 IVA proceeds to patient validation.
3.	Patient Validation
3.1 IVA asks the caller to provide: 
o	Patient ID
o	Last four digits of SSN
3.2 IVA invokes a Cloud Run function for validation.
3.3 Cloud Run queries the BigQuery master patient table.
3.4 Validation result is returned to IVA.
4.	Validation Outcome Handling
o	If patient is valid: IVA continues to appointment scheduling.
o	If patient is invalid:
4.1 IVA informs the caller that the provided details could not be validated.
4.2 IVA allows the caller to retry (configurable retry attempts).
4.3 If validation still fails, IVA plays a relevant message and asks the caller to call back later, then ends the call.
5.	Department Selection
5.1 IVA asks the patient to select the department (e.g., Radiology, Cardiology, Ophthalmology).
5.2 Patient selects the department.
6.	Fetch Available Appointment Slots
6.1 IVA invokes a Cloud Run function to retrieve appointment availability.
6.2 Cloud Run queries the department-specific appointment slots table in BigQuery.
6.3 Available slots are returned to IVA.
7.	Slot Announcement & Selection
7.1 IVA announces the available appointment slots to the patient.
7.2 Patient selects one of the announced slots.
8.	Appointment Confirmation
8.1 IVA confirms the selected slot with the patient.
8.2 IVA invokes Cloud Run to update BigQuery and mark the slot as Booked.
8.3 IVA provides appointment confirmation details to the patient.
9.	Call Closure
9.1 IVA thanks the patient and ends the call.
________________________________________
Call Flow 2: Appointment Booking – Requested Slot Is Already Booked
1.	Inbound Call to Slot Retrieval
Steps 1 through 6 are identical to Call Flow 1 (greeting, intent confirmation, patient validation, department selection, and slot retrieval).
2.	Slot Announcement
2.1 IVA announces the available appointment slots retrieved from BigQuery.
3.	Patient Requests a Different Slot
3.1 Patient requests a slot that is not part of the announced options.
3.2 IVA invokes Cloud Run to verify the requested slot availability in BigQuery.
4.	Slot Availability Check
o	If requested slot is already booked:
4.1 IVA informs the patient that the requested slot is unavailable.
4.2 IVA announces the currently available slots again.
4.3 IVA asks the patient to choose from the announced options.
5.	Slot Selection From Available Options
5.1 Patient selects one of the available slots.
6.	Appointment Confirmation
6.1 IVA confirms the selected slot with the patient.
6.2 IVA updates the department-specific BigQuery table to book the slot.
6.3 IVA provides appointment confirmation to the patient.
7.	Call Closure
7.1 IVA thanks the patient and ends the call.
________________________________________
Exception Handling (IVA-Only, No Live Agent)
1. No Available Slots for Selected Department
1.	IVA informs the patient that no immediate slots are available for the selected department.
2.	IVA invokes Cloud Run to retrieve the next three available future slots from BigQuery.
3.	IVA announces the next three available slots to the patient.
4.	Patient selects a slot.
5.	IVA books and confirms the appointment.
6.	IVA ends the call.
________________________________________
2. Backend or System Error (Cloud Run / BigQuery)
1.	IVA plays a standard system error message.
2.	IVA asks the patient to call back later.
3.	IVA ends the call.
 
Assumptions for Appointment Scheduling IVA Flow
1. Patient Validation Assumptions
1.	Patient validation is performed using Patient ID + Last 4 digits of SSN.
2.	Validation is executed via a Cloud Run function, which queries BigQuery as the system of record.
3.	BigQuery contains a master patient table with a unique Patient ID as the primary key.
4.	If validation fails, IVA will allow configurable retry attempts (e.g., 2–3 retries) before terminating the call.
________________________________________
2. Department Mapping Assumptions
1.	Departments (e.g., Radiology, Cardiology, Ophthalmology) are predefined and aligned with backend data in BigQuery.
2.	Each department has a dedicated appointment slots table in BigQuery.
________________________________________
3. Appointment Slot Retrieval Assumptions
1.	Appointment slots are stored in BigQuery with at least the following attributes: 
o	Slot Date
o	Slot Time
o	Slot Status (Available / Booked)
2.	Cloud Run function retrieves only slots with status = Available from the selected department’s table.
3.	Slots are returned in chronological order (earliest to latest).
________________________________________
4. “Next Three Available Slots” Calculation Assumptions
1.	When no immediate slots are available, Cloud Run retrieves the next three future available slots for the selected department.
2.	The logic is: 
o	Sort by Slot Date ASC
o	Then Slot Time ASC
o	Return the first three records where status = Available
3.	If fewer than three slots exist, IVA will announce all available future slots.
________________________________________
5. Slot Selection and Booking Assumptions
1.	When a patient selects a slot, IVA invokes Cloud Run to update the corresponding department’s BigQuery table and mark the slot as Booked.
2.	Slot booking is assumed to be atomic (handled via transaction logic in Cloud Run or BigQuery to avoid race conditions).
3.	IVA confirms the booking only after receiving a successful update response.
________________________________________
6. Slot Conflict Handling Assumptions
1.	If a patient requests a slot outside the announced list, IVA validates the slot against the department-specific appointment table.
2.	If the slot is already booked, IVA will: 
o	Inform the patient the slot is unavailable
o	Re-announce available slots
o	Prompt the patient to choose from the available options
________________________________________
7. Error Handling Assumptions
1.	If Cloud Run or BigQuery is unavailable, IVA plays a generic system error message and requests the patient to call back later.
2.	Error logs and telemetry are assumed to be captured in Cloud Logging / Monitoring.
________________________________________
8. Security & Compliance Assumptions
1.	Patient data access via Cloud Run and BigQuery follows HIPAA-compliant security controls (IAM roles, encryption, audit logging).
2.	SSN digits are not stored in IVA context logs beyond runtime validation.
