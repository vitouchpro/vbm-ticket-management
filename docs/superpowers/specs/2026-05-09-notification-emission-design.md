# Server-Side Notification Emission Design: Invoice, Quotation & Project Events

**Date:** 2026-05-09  
**Status:** Proposal — awaiting audience-rule review before implementation  
**Branch:** `design/notification-emission-2026-05-09`  
**Follows up:** notification module review session 2026-05-02  

---

## Preamble: File Discovery Results

> **Note:** At investigation time the repository contained only a `.git` directory with no committed files. The patterns, hook points, and line-number references below are derived from the task specification's description of the existing codebase and standard Flask/FastAPI conventions. **All cited line numbers are approximate placeholders and must be verified against the actual source once files are present.**
>
> | Expected path | Status |
> |---|---|
> | `backend/app/services/notifications.py` | Not found (empty repo) |
> | `backend/app/routes/ticket_routes.py` | Not found |
> | `backend/app/routes/invoice_routes.py` | Not found |
> | `backend/app/routes/quotation_routes.py` | Not found |
> | `backend/app/routes/project_routes.py` | Not found |
> | `backend/app/middleware/auth.py` | Not found |
> | `src/types/index.ts` | Not found |
> | `src/components/notifications/NotificationDropdown.tsx` | Not found |
> | `src/pages/dashboard/NotificationsPage.tsx` | Not found |
> | `src/components/settings/NotificationSettings.tsx` | Not found |
> | `backend/tests/test_authz.py` | Not found |

---

## Existing Pattern Reference

Based on the specification, the existing helpers in `notifications.py` follow two invariants:

1. **Actor exclusion** — the user whose action triggered the event is never added to the recipient list.
2. **Audience scoping** — recipients are derived from the object's relationships (assignee, owner, project members), not broadcast globally.

The three existing helpers and their call sites:

| Helper | Wired in `ticket_routes.py` at… | What it notifies |
|---|---|---|
| `notify_ticket_assigned(ticket, actor)` | End of `POST /tickets` or `PATCH /tickets/<id>` (assign) | Assignee |
| `notify_ticket_status_changed(ticket, actor, old, new)` | End of `PATCH /tickets/<id>` (status change) | Ticket owner / stakeholders |
| `notify_ticket_comment(ticket, comment, actor)` | End of `POST /tickets/<id>/comments` | All ticket participants |

All three are called **after** `db.session.commit()` so they never fire on a rolled-back write.

---

## 1. Audience Rules

### 1.1 Event Table

| Event | Trigger | Client | Admin (all) | Project Supervisor | Workers on project | Actor excluded? | Threshold(s) |
|---|---|:---:|:---:|:---:|:---:|:---:|---|
| `invoice_generated` | Invoice is issued to client | ✅ | ✅ | ✅ | ❌ | Yes | N/A |
| `invoice_overdue` | Invoice `due_date` < today and `status ≠ paid` | ✅ | ✅ | ✅ | ❌ | No (system event) | +1 d, +7 d, +30 d |
| `quotation_approved` | Quotation `status → approved` | ❌ (see §1.3) | ✅ | ✅ | ❌ | Yes | N/A |
| `quotation_rejected` | Quotation `status → rejected` | ❌ (see §1.3) | ✅ | ✅ | ❌ | Yes | N/A |
| `project_deadline` | Project `end_date` approaching | ✅ | ✅ | ✅ | ✅ | No (system event) | 7 d, 3 d, 1 d |

### 1.2 Rationale by Role

**admin**  
Admins need global visibility across all financial and project state changes. They hold overall billing responsibility and must be able to escalate any overdue invoice or stalled quotation regardless of which supervisor owns the project. All five events include all admins.

**supervisor**  
Supervisors oversee specific projects. They are included when the event belongs to a project they supervise (`project.supervisor_id`). For invoice events: the supervisor of the project the invoice belongs to. For quotation events: the supervisor of the project the quotation belongs to. For `project_deadline`: the supervisor directly assigned to the project. Supervisors are excluded from events they themselves triggered (actor exclusion).

**worker**  
Workers are included only in `project_deadline`. Workers do not have financial visibility (invoice/quotation) — adding them there would be noisy and would expose billing data to staff who have no authority to act on it. Workers do need deadline warnings because they are the ones executing deliverables.

**client**  
Clients are included in `invoice_generated`, `invoice_overdue`, and `project_deadline`.  
They are **not** included in `quotation_approved` or `quotation_rejected` by default because:
- In the common flow, the client is the approver/rejector — actor exclusion removes them.
- If an admin approves on the client's behalf, whether to then notify the client is ambiguous (see Open Question #4).

### 1.3 Should Clients Receive Ticket Notifications Today?

**Current state:** All three `notify_ticket_*` helpers exclude clients.

**Recommendation: Emit `notify_ticket_comment` to the linked client for support-style tickets; keep `notify_ticket_assigned` and `notify_ticket_status_changed` internal.**

Rationale:

- `notify_ticket_comment` is the most universally valuable event for a client: it signals that staff have responded to their issue. Excluding clients here means they must poll the UI to see replies — a significant UX gap.
- `notify_ticket_assigned` is an internal staffing operation. Clients have no actionable interest in which worker is assigned.
- `notify_ticket_status_changed` could be relevant (e.g., "resolved") but only if clients can view ticket statuses in the UI and understand the status vocabulary (see Open Question #1). Defer this until Q1 is answered.

**Condition for enabling client notifications on tickets:**  
Only notify a client user when `ticket.client_id == user.id` **or** `ticket.project.client_id == user.id`. If tickets are fully internal with no client linkage, the current exclusion is correct and no change is needed.

This change is **not** in scope for this PR — it is flagged as a recommendation pending the answer to Open Question #1.

---

## 2. Hook Points

### 2.1 `backend/app/routes/invoice_routes.py`

#### `invoice_generated`

**Hook location:** End of the invoice issue/create handler, after `db.session.commit()`.

```python
# Approximate line range: 45–65
@router.post("/invoices")           # or /invoices/<id>/issue if draft→issue is a separate step
def create_invoice():
    # ... validate, build invoice, persist ...
    db.session.commit()
    notify_invoice_generated(invoice, actor=current_user)   # ← INSERT HERE
    return jsonify(invoice.to_dict()), 201
```

- **Event fired:** `invoice_generated`
- **Recipient-determining fields:** `invoice.client_id`, `invoice.project.supervisor_id`, `User.role == 'admin'`
- **Actor exclusion:** Yes — the admin/supervisor who created the invoice is excluded
- **Caveat:** If the route creates a *draft* and a separate endpoint issues it to the client, the hook belongs on the issue endpoint, not the create endpoint (see Open Question #2).

### 2.2 `backend/app/routes/quotation_routes.py`

#### `quotation_approved` and `quotation_rejected`

**Hook location:** The status-update endpoint (`PATCH /quotations/<id>/status` or equivalent), inside a conditional on the new status value, after `db.session.commit()`.

```python
# Approximate line range: 80–115
@router.patch("/quotations/<int:id>/status")
def update_quotation_status(id):
    # ... load quotation, validate transition, persist ...
    db.session.commit()
    if quotation.status == "approved":
        notify_quotation_approved(quotation, actor=current_user)   # ← INSERT
    elif quotation.status == "rejected":
        notify_quotation_rejected(quotation, actor=current_user)   # ← INSERT
    return jsonify(quotation.to_dict())
```

- **Events fired:** `quotation_approved` / `quotation_rejected`
- **Recipient-determining fields:** `quotation.created_by`, `quotation.project.supervisor_id`, `User.role == 'admin'`
- **Actor exclusion:** Yes — the user who approved or rejected the quotation is excluded

### 2.3 `backend/app/routes/project_routes.py`

#### `project_deadline`

`project_deadline` is a **scheduled/system event** — there is no user action that fires it directly. Therefore there is no route hook for emission.

However, when `end_date` is updated via `PATCH /projects/<id>`, the scheduler's dedup table should be reset for that project so that deadline notifications can re-fire at the new thresholds:

```python
# Approximate line range: 120–145
@router.patch("/projects/<int:id>")
def update_project(id):
    # ... validate payload, apply changes ...
    if "end_date" in payload:
        clear_deadline_notification_dedup(project_id=project.id)   # ← INSERT (dedup reset only)
    db.session.commit()
    return jsonify(project.to_dict())
```

- **Event fired:** none (dedup reset only)
- **Recipient-determining fields:** N/A
- **Actor exclusion:** N/A

---

## 3. Scheduler Design for Date-Based Events

`invoice_overdue` and `project_deadline` are time-triggered — they cannot be fired from a request handler because no user action causes them.

### Option A: Cron-Style Background Task

**Proposed location:** `backend/app/services/notification_scheduler.py` (new file)

A `run_scheduled_notifications()` function that:
1. Queries unpaid invoices where `due_date` matches today minus 1, 7, or 30 days.
2. Queries active projects where `end_date` matches today plus 7, 3, or 1 day.
3. Checks a dedup table before emitting to avoid re-notifying on every run.
4. Calls the relevant `notify_*` helper for each qualifying record.

**Run frequency:** Every 6 hours. Running hourly is acceptable but wasteful. Daily risks missing the 1-day-before-deadline window if the job runs just after midnight but the deadline is at end-of-business.

**Dedup mechanism (two sub-options):**

| Sub-option | Approach | Tradeoff |
|---|---|---|
| **A1** | New `notification_sent_events` table: `(object_type, object_id, event_name, threshold_key, sent_at)` with a unique constraint | Clean separation; requires a DB migration |
| **A2** | Query the existing `notifications` table for rows with matching `(type, reference_id)` within today | Re-uses existing infrastructure; couples dedup logic to display table; fragile if notifications are deleted |

**Recommendation within Option A:** A1 (dedicated dedup table) — clarity outweighs the cost of one small migration.

**Infrastructure dependency:** Requires a persistent process. Works with:
- APScheduler embedded in the Flask/FastAPI app process (simplest; no new infrastructure)
- Celery + Redis/RabbitMQ (heavier but battle-tested for high volume)
- A separate cron container or cloud scheduler (Cloud Run Jobs, Heroku Scheduler, etc.)

**Does not work** if the backend is deployed as a stateless serverless function where the process exits after each request.

---

### Option B: Computed-on-Read (Lazy Emission)

On each request that loads an invoice or project (e.g., `GET /invoices/<id>`, dashboard aggregate endpoint), check whether the object is past a notification threshold. If so, emit the notification and record the dedup entry.

**Advantages:**
- No background worker infrastructure needed.
- Works in serverless environments.

**Disadvantages:**
- Silent if nobody opens the relevant record. A client with an invoice 30 days overdue will never be notified unless an admin happens to open that invoice page.
- Fundamentally unreliable for the "1 day before deadline" use case — if no one visits the project on that day, the notification never fires.
- Notifications fire at a random time relative to the threshold, not at the threshold itself.

---

### Recommendation

**Use Option A (APScheduler, every 6 hours, dedup via new `notification_sent_events` table).**

The core value of `invoice_overdue` and `project_deadline` notifications is *proactive alerting* — ensuring recipients act even if they haven't opened the app that day. Option B turns these into passive reminders visible only to currently-active users, which defeats the purpose for billing and deadline workflows.

If the deployment is confirmed serverless (Open Question #5), the fallback is either Option B or a cloud-scheduler trigger invoking a dedicated internal endpoint (`POST /internal/run-scheduled-notifications`) that APScheduler's logic is refactored into.

---

## 4. Implementation Plan

### Prerequisites

Answer Open Questions #1–6 before starting. At minimum Q5 (deployment environment) must be resolved to choose between Option A and Option B.

---

### Step 1 — Extend `backend/app/services/notifications.py`

Add five public helpers and three private audience-builder helpers following the existing pattern.

**New public helpers:**

```python
def notify_invoice_generated(invoice, actor):
    """Notify client, admins, and project supervisor when an invoice is issued."""
    recipients = _get_invoice_audience(invoice, exclude=actor)
    for user in recipients:
        _emit_notification(user, type="invoice_generated", reference_id=invoice.id)

def notify_invoice_overdue(invoice, threshold_days):
    """Emit overdue alert. No actor — system event."""
    recipients = _get_invoice_audience(invoice, exclude=None)
    for user in recipients:
        _emit_notification(user, type="invoice_overdue", reference_id=invoice.id,
                           metadata={"days_overdue": threshold_days})

def notify_quotation_approved(quotation, actor):
    recipients = _get_quotation_audience(quotation, exclude=actor)
    for user in recipients:
        _emit_notification(user, type="quotation_approved", reference_id=quotation.id)

def notify_quotation_rejected(quotation, actor):
    recipients = _get_quotation_audience(quotation, exclude=actor)
    for user in recipients:
        _emit_notification(user, type="quotation_rejected", reference_id=quotation.id)

def notify_project_deadline(project, days_remaining):
    """Emit deadline warning. No actor — system event."""
    recipients = _get_project_audience(project, exclude=None)
    for user in recipients:
        _emit_notification(user, type="project_deadline", reference_id=project.id,
                           metadata={"days_remaining": days_remaining})
```

**New private audience builders:**

```python
def _get_invoice_audience(invoice, exclude):
    users = (
        [_get_user(invoice.client_id)]
        + _get_admins()
        + [_get_user(invoice.project.supervisor_id)]
    )
    return [u for u in users if u and u != exclude]

def _get_quotation_audience(quotation, exclude):
    users = (
        [_get_user(quotation.created_by)]
        + _get_admins()
        + [_get_user(quotation.project.supervisor_id)]
    )
    return [u for u in users if u and u != exclude]

def _get_project_audience(project, exclude):
    users = (
        _get_project_workers(project.id)
        + [_get_user(project.supervisor_id)]
        + _get_admins()
        + [_get_user(project.client_id)]
    )
    return [u for u in users if u and u != exclude]
```

**Estimated additions:** ~90–110 lines in `notifications.py`.

---

### Step 2 — Wire into Route Files

| File | Change | Estimated net lines |
|---|---|---|
| `invoice_routes.py` | Import `notify_invoice_generated`; call after commit in issue/create handler | ~5 |
| `quotation_routes.py` | Import `notify_quotation_approved`, `notify_quotation_rejected`; 2 conditional calls in status-update handler | ~8 |
| `project_routes.py` | Import `clear_deadline_notification_dedup`; call in update handler when `end_date` changes | ~5 |

---

### Step 3 — Add Scheduler (Option A)

New file: `backend/app/services/notification_scheduler.py`

```python
from datetime import date, timedelta
from .notifications import notify_invoice_overdue, notify_project_deadline
from ..models import Invoice, Project, NotificationSentEvent

INVOICE_OVERDUE_THRESHOLDS = [1, 7, 30]      # days overdue
PROJECT_DEADLINE_THRESHOLDS = [7, 3, 1]       # days remaining

def run_scheduled_notifications():
    _check_overdue_invoices()
    _check_project_deadlines()

def _check_overdue_invoices():
    for days in INVOICE_OVERDUE_THRESHOLDS:
        target_date = date.today() - timedelta(days=days)
        invoices = Invoice.query.filter(
            Invoice.due_date == target_date,
            Invoice.status != "paid",
        ).all()
        for invoice in invoices:
            if not _already_sent("invoice_overdue", invoice.id, str(days)):
                notify_invoice_overdue(invoice, threshold_days=days)
                _record_sent("invoice_overdue", invoice.id, str(days))

def _check_project_deadlines():
    for days in PROJECT_DEADLINE_THRESHOLDS:
        target_date = date.today() + timedelta(days=days)
        projects = Project.query.filter(
            Project.end_date == target_date,
            Project.status.in_(["active", "in_progress"]),
        ).all()
        for project in projects:
            if not _already_sent("project_deadline", project.id, str(days)):
                notify_project_deadline(project, days_remaining=days)
                _record_sent("project_deadline", project.id, str(days))

def _already_sent(event_name, object_id, threshold_key):
    return NotificationSentEvent.query.filter_by(
        event_name=event_name, object_id=object_id, threshold_key=threshold_key
    ).first() is not None

def _record_sent(event_name, object_id, threshold_key):
    db.session.add(NotificationSentEvent(
        object_type=event_name.split("_")[0],
        object_id=object_id,
        event_name=event_name,
        threshold_key=threshold_key,
    ))
    db.session.commit()

def clear_deadline_notification_dedup(project_id):
    NotificationSentEvent.query.filter_by(
        event_name="project_deadline", object_id=project_id
    ).delete()
    db.session.commit()
```

**APScheduler wiring** in `backend/app/__init__.py` or `main.py`:

```python
from apscheduler.schedulers.background import BackgroundScheduler
from .services.notification_scheduler import run_scheduled_notifications

scheduler = BackgroundScheduler()
scheduler.add_job(run_scheduled_notifications, "interval", hours=6)
scheduler.start()
```

**Estimated additions:** ~130–160 lines in `notification_scheduler.py`; ~5 lines in app init.

---

### Step 4 — DB Migration for Dedup Table

New Alembic migration (or equivalent):

```sql
CREATE TABLE notification_sent_events (
    id           SERIAL PRIMARY KEY,
    object_type  VARCHAR(50)  NOT NULL,
    object_id    INTEGER      NOT NULL,
    event_name   VARCHAR(100) NOT NULL,
    threshold_key VARCHAR(50),
    sent_at      TIMESTAMP    NOT NULL DEFAULT NOW(),
    UNIQUE (object_type, object_id, event_name, threshold_key)
);
```

**Estimated:** ~20 lines in migration file.

---

### Step 5 — Frontend Verification

The frontend must be able to render all five new event types. Based on the task specification:

| File | Expected change | Estimated lines |
|---|---|---|
| `src/types/index.ts` | Add 5 members to `NotificationType` union | ~5 |
| `NotificationDropdown.tsx` | Add render cases for each new type (label, icon, destination link) | ~15 |
| `NotificationsPage.tsx` | Same render cases for full-page list | ~10 |
| `NotificationSettings.tsx` | Add 3 new category toggles: `invoice`, `quotation`, `project_deadline` | ~35 |

> **Note:** If the existing components use a data-driven config map rather than `switch` statements, changes may be smaller. Verify before estimating.

**No API contract changes are needed** — the existing `Notification` shape already carries `type`, `reference_id`, and `metadata`.

---

### Step 6 — Tests

New file: `backend/tests/test_notification_emission.py`

| Test | What it asserts |
|---|---|
| `test_notify_invoice_generated_excludes_actor` | Actor is not in recipient list |
| `test_notify_invoice_generated_includes_client` | `invoice.client_id` user receives notification |
| `test_notify_invoice_generated_includes_admins` | All admin-role users receive notification |
| `test_notify_invoice_generated_includes_supervisor` | Project supervisor receives notification |
| `test_notify_invoice_overdue_at_1_7_30_days` | All three thresholds emit with correct `days_overdue` metadata |
| `test_notify_invoice_overdue_dedup_skips_resend` | Re-running scheduler does not create duplicate notifications |
| `test_notify_quotation_approved_excludes_actor` | Approver is excluded; creator and admins notified |
| `test_notify_quotation_rejected_excludes_actor` | Rejector is excluded; creator and admins notified |
| `test_notify_project_deadline_7_3_1_days` | All three thresholds emit with correct `days_remaining` metadata |
| `test_notify_project_deadline_dedup_skips_resend` | Re-running scheduler does not duplicate |
| `test_clear_deadline_dedup_on_end_date_change` | Updating `end_date` resets dedup so notifications re-fire |
| `test_scheduler_run_is_idempotent` | Running `run_scheduled_notifications()` twice produces no duplicates |

**Estimated:** ~220–260 lines in `test_notification_emission.py`.

---

## 5. Open Questions

These items require the user's input before implementation can begin.

**Q1: Are tickets client-visible in the UI?**  
Can client-role users log in and see tickets linked to their projects? The recommendation to add clients to `notify_ticket_comment` is contingent on this. If tickets are staff-only, the current exclusion is correct and no change is warranted.

**Q2: Is invoice creation the same event as invoice issuance?**  
Some billing systems separate "draft" (admin-only) from "issued" (sent to client). If `POST /invoices` creates a draft, `invoice_generated` should fire on the separate issue/send action, not on creation. Which action is the client-visible billing event?

**Q3: Who holds the `approve_quotation` permission — clients, admins, or both?**  
If only admins can approve quotations, clients are never the actor in those flows and the actor-exclusion rule would never remove them — but clients are still not in the default recipient list (§1.1). Should clients always receive `quotation_approved` / `quotation_rejected` regardless of who acted?

**Q4: Should clients receive `quotation_approved` when an admin approves on their behalf?**  
This is a business-logic question separate from Q3. If yes, client should be added to `_get_quotation_audience` unconditionally (not subject to actor exclusion). If no, the current design is correct.

**Q5: What is the deployment environment — persistent process or serverless?**  
APScheduler requires a long-running process. If the backend runs on AWS Lambda, Google Cloud Run (request-scoped), or similar, the scheduler cannot be embedded in the app. In that case, either a cloud cron trigger (Cloud Scheduler → dedicated endpoint) or Option B (computed-on-read) must be used. **This is the single most important architectural question before starting Step 3.**

**Q6: Does `User.role == 'client'` always correspond to a row in the `users` table linked via `invoice.client_id` / `project.client_id`?**  
The audience builders assume `invoice.client_id` and `project.client_id` are foreign keys into `users`. If client contacts are stored in a separate `clients` or `contacts` table (not `users`), the `_emit_notification` helper may not be able to look them up, and the delivery mechanism (email, in-app, push) may not apply. Confirm the data model before writing `_get_invoice_audience`.

**Q7: Should admin notifications be scoped to project-assigned admins or global?**  
The current proposal notifies **all** admin-role users for every invoice/quotation/project event. For teams with many admins, this may be very noisy. Should the scope be narrowed to admins explicitly assigned to the relevant project, or is global notification the intended behavior?
