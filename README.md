# GRIT Interview Ops

A responsive React + TypeScript prototype for NIAT GRIT Level Clearance Interviews. The demo is intentionally credential-free and persists all actions in `localStorage`. A production-ready PostgreSQL/Supabase schema and atomic booking RPC are included.

## Run
```bash
npm install
npm run dev
```
Open the local Vite URL. Use the top-right role selector; no passwords are required. `npm test` runs lifecycle guard tests and `npm run build` verifies the production bundle.

## Evaluator walkthrough
1. **Student / Ananya Iyer:** book an open slot; refresh to confirm persistence and duplicate prevention.
2. **Student / Arjun Nair:** verify booking is unavailable until all required skills are complete.
3. **Interviewer / Aarav Menon:** add or block availability; open an assigned interview and submit PASS, FAIL, or RETAKE LATER with feedback and internal notes.
4. **Student:** switch to the affected learner to see the result, feedback, notification, and modified badge state.
5. **Admin / Ops:** filter `Eligible · not booked`, inspect workload, and view audit events.
6. **Admin / Ops:** select Sara Khan (BADGE_EARNED), enter reason and evidence, confirm the verified decision, then switch back to Sara to see NULLIFIED. Allegations alone never nullify.
7. Use **Reset** to restore all seeded scenarios.

## Demo scenarios
- Before result: Rohan Verma (BOOKED)
- Pass/badge earned: Sara Khan
- Fail: Dev Patel
- Retake: Ishita Bose
- Pending result: Vikram Singh
- Not eligible: Arjun Nair
- Already nullified: Aditya Kulkarni

## Architecture
- **UI:** React 19, TypeScript, responsive CSS system, accessible native controls, confirmation modal and toast feedback.
- **Demo persistence:** versioned browser `localStorage`; deterministic reset seed with 10 students, 3 interviewers, 4 tracks, open/booked/blocked slots.
- **Production path:** replace the state adapter with Supabase Auth + typed queries. Execute `supabase/migrations/001_schema.sql`; expose only the `book_interview_slot` security-definer RPC for booking writes; enable RLS by role.
- **Concurrency:** slot row is locked `FOR UPDATE`; status, expiry and eligibility are rechecked server-side; `UNIQUE(slot_id)`, partial unique active-booking index and unique idempotency key resolve races and double clicks.
- **Time:** all database times are UTC `timestamptz`; UI displays `Asia/Kolkata` explicitly.

## Product assumptions (configurable)
- Interview duration is 60 minutes and timestamps are displayed in IST.
- Cancellation reopens the slot in the demo. In production, cancellation cutoffs and notification escalation are configuration values.
- `FAIL` is terminal by default; `RETAKE LATER` creates retake eligibility. Ops may configure whether failed learners receive another attempt.
- A booked slot cannot be silently removed. Production cancellation creates a replacement workflow and notifies the learner.
- Pending results are escalated to Ops after a configurable SLA, recommended 24 hours.

## Edge-case behavior
| Case | Behavior |
|---|---|
| Two students select one slot | Atomic RPC locks the row; one commit succeeds, the other receives `BOOKING_CONFLICT`. |
| Double click / retry | Unique idempotency key returns/blocks the duplicate request. |
| Ineligible student | Server verifies completed required skills/state before insert; UI explains missing skills. |
| Overlap | PostgreSQL exclusion constraint rejects interviewer time overlap. |
| Multiple active bookings | Partial unique index on `student_id WHERE active`. |
| Availability withdrawn after booking | Existing booking is preserved; Ops must cancel/reschedule with notification. |
| No show | Authorized interviewer/Ops transitions BOOKED to NO_SHOW; retake policy is explicit. |
| Forgotten result | Pending-results queue and SLA reminder/escalation. |
| No/full slots | Empty state; no synthetic confirmation; Ops sees capacity gap. |
| Refresh | Browser state persists; production state is read from PostgreSQL. |
| Malpractice allegation | Case remains ALLEGED. Only authorized VERIFIED decision transitions BADGE_EARNED to NULLIFIED and writes history/audit. |

## Official reference
The public “How GRIT works” page describes Tracks, Badges, required Skills/Levels and a final Level Clearance Interview. Existing policies not stated there are treated above as configurable assumptions: https://www.grit.niatindia.com/how-grit-works

## Files
- `src/main.tsx`: complete role-based product demo
- `src/stateMachine.ts`: valid lifecycle transitions
- `src/stateMachine.test.ts`: lifecycle tests
- `supabase/migrations/001_schema.sql`: normalized schema + atomic booking function
- `docs/GRIT_Product_Writeup.pdf`: two-page product document
- `docs/REQUIREMENTS_CHECKLIST.md`: QA traceability and test results
