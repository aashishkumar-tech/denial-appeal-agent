# 4. Dashboard

The dashboard is where users see denials, check what the AI did, approve letters and follow results.

## 4.1 User roles
| Role | Can do |
|---|---|
| **Reviewer (billing staff)** | Work queue, review, edit, approve or reject drafts |
| **Manager** | Everything a reviewer does, plus team KPIs and assigning work |
| **Clinical / CDI** | Answer documentation queries |
| **Admin** | Users, thresholds, rules config |
| **Viewer / Leadership** | Read-only KPIs and reports |

## 4.2 Screens

### 1. Home / KPI overview
- Total denied $, recovered $, overturn rate, open appeals
- Deadlines due in 7 and 14 days
- Trend chart: denials and recoveries by month
- Top denial reasons, top payers

### 2. Work queue
- Table: claim ID, payer, reason, amount, P(overturn), expected $, deadline, status
- Sorted by priority by default
- Filters: payer, category, status, assigned user, deadline
- Colors: red = deadline ≤ 14 days, amber = needs a human decision

### 3. Denial detail
- Claim and denial information
- **Why this decision:** rules applied and ML score with the top factors
- Evidence panel: notes and policy criteria, marked found or missing
- Button: **Run agent / Regenerate draft**

### 4. Appeal review
- Draft letter on the left, source documents on the right
- Click a citation to jump to the source text
- Actions: **Approve**, **Edit**, **Reject (with reason)**
- Edits are saved and used to improve the templates

### 5. Tracking
- Submitted appeals with status: Submitted → Pending → Overturned / Upheld
- Follow-up dates and overdue alerts
- Record the outcome and amount recovered

### 6. Root-cause analytics
- Denials by category, payer, CPT, department
- Preventable denials (for example, repeated missing authorizations)
- Export to CSV or PDF

### 7. My items (self-service)
- "My assigned cases", "My pending approvals", "My documentation queries"
- Search by claim ID or patient ID
- Notifications for new assignments and approaching deadlines

### 8. Admin / Settings
- Thresholds (min $, probability cut-offs, deadline alert days)
- Denial rules and templates
- Users and roles
- Audit log viewer

## 4.3 Case status flow
```
New → Scored → [Write-off | Corrected claim | Needs review | Drafting]
Drafting → Doc query (if gap) → Drafting
Drafting → In review → Approved → Submitted → Overturned / Upheld → Closed
                     → Rejected → Closed or Re-draft
```

## 4.4 Technology
| Layer | Tool | Why |
|---|---|---|
| Front end | React (TypeScript) | Scales, rich viewer and side panel, fine-grained UI per role |
| Back end | FastAPI | Serves all data; enforces login, roles and data filtering |

## 4.5 Non-functional requirements
- Login with separate app accounts; role-based access on every screen
- Patient identifiers masked unless the role needs them
- Queue loads in under 2 seconds for 10K open items
- Every approve, reject and edit written to the audit log

## 4.6 Login and user access

### Rule: each user sees only their own data
Every user logs in, and every screen, search, export and API call is filtered to the data their role and assignments allow. Filtering happens **on the server**, not only by hiding things on screen.

### Login flow
```
Open dashboard → Login → Identity verified → Role + team loaded
→ Every request carries the user's token → Server filters data → Screen shows only allowed data
```

**Decision: separate app accounts** (not company SSO).

- Username + password login handled by FastAPI; users stored in PostgreSQL.
- Successful login (password + MFA code) returns a short-lived access token (JWT, e.g. 15 minutes) and a refresh token.
- Every API call checks the token, the user's role and the data they can access.

### Account management
| Item | Rule |
|---|---|
| Account creation | Only Admin creates accounts and assigns role + team (no self sign-up) |
| First login | Temporary password; user must change it at first login |
| Password policy | Minimum 12 characters, not reused (last 5), not the username |
| Failed logins | Account locked for 15 minutes after 5 failed attempts |
| Password reset | Admin-triggered reset or emailed one-time link (expires in 30 minutes) |
| MFA | Required for all users (authenticator app code) |
| Leavers | Admin deactivates the account; sessions end immediately; data stays for audit |
| Reviews | Admin reviews active accounts and roles every quarter |

### Data visibility by role
| Role | Sees | Cannot see |
|---|---|---|
| **Reviewer** | Only cases assigned to them | Other users' cases, admin settings |
| **Manager** | All cases for their own team(s); team KPIs | Other teams' cases |
| **Clinical / CDI** | Only documentation queries assigned to them, plus the notes needed to answer | Financial details, other cases |
| **Viewer / Leadership** | Aggregated KPIs and reports only (no patient-level data) | Individual cases, patient data |
| **Admin** | Users, roles, thresholds, rules, audit log | Patient-level case data (unless also given a work role) |

### Permissions by action
| Action | Reviewer | Manager | Clinical | Viewer | Admin |
|---|---|---|---|---|---|
| View own assigned cases | ✅ | ✅ | ✅ (queries) | ❌ | ❌ |
| View team cases | ❌ | ✅ | ❌ | ❌ | ❌ |
| Run agent / edit draft | ✅ | ✅ | ❌ | ❌ | ❌ |
| Approve / reject appeal | ✅ | ✅ | ❌ | ❌ | ❌ |
| Assign / reassign cases | ❌ | ✅ | ❌ | ❌ | ❌ |
| Answer documentation query | ❌ | ❌ | ✅ | ❌ | ❌ |
| View KPIs | Own only | Team | ❌ | All (aggregated) | ❌ |
| Export data | Own cases | Team | ❌ | Aggregated | ❌ |
| Manage users, roles, settings | ❌ | ❌ | ❌ | ❌ | ✅ |
| View audit log | ❌ | Team | ❌ | ❌ | ✅ |

### Data model for access
| Table | Key fields |
|---|---|
| `users` | user_id, username, name, email, role, team_id, active, password_hash, must_change_password, failed_attempts, locked_until, last_login |
| `teams` | team_id, name, manager_user_id |
| `assignments` | case_id, assigned_user_id, team_id, assigned_by, assigned_at |
| `audit_log` | log_id, user_id, action, case_id, timestamp, details |

- Every case has an owner (`assigned_user_id`) and a `team_id`; access is checked against these.
- **Row-level security** in PostgreSQL is a second layer, so data stays filtered even if an API check is missed.

### Security rules
- Passwords hashed (bcrypt); never stored or logged in plain text
- Token signing secret kept in environment variables / key vault, never in code
- Deny by default: a new role or user sees nothing until access is granted
- Session timeout after inactivity (e.g. 15 minutes)
- Patient identifiers masked unless the role needs them
- HTTPS for all traffic
- Log logins, failed logins, case views, approvals and exports
- Access tests: automated tests confirm that user A can't open, search or export user B's cases

## 4.7 Document upload and viewer

### Upload flow
```
Upload → Validate (type, size, virus scan) → Store encrypted → OCR
→ Classify document type → Extract fields → Link to case → Ready to view
```

| Item | Rule |
|---|---|
| Allowed types | PDF, JPG, PNG, TIFF, DOCX, EDI (835/837), ZIP of these |
| Upload options | Drag and drop, multiple files, ZIP for a full medical record |
| Size limit | Configurable (e.g. 50 MB per file) |
| Linking to a case | User picks the case, or the system suggests one from the claim ID / patient ID it reads |
| Status shown | Uploaded → Processing → Ready / Failed (with reason) |
| Who can upload | Reviewer, Manager (own/team cases); Clinical (for their queries only) |

### Viewing options
| View | What it shows |
|---|---|
| **Document viewer** | Original PDF or scan, page by page, zoom and rotate |
| **Field highlights** | Boxes on the page around extracted fields (amount, codes, dates) |
| **Extracted fields panel** | Field values with confidence colors (green = sure, amber = check, red = not found); user can correct them |
| **Evidence highlights** | The exact lines cited in the appeal letter; clicking a citation jumps to that page |
| **Case timeline** | All documents on a case in date order |
| **Search** | Text search across the user's own documents |

### Side panel (optional, user choice)
Users can open their document in a **side panel on one side of the screen** while they keep working in the main area (work queue, case detail, appeal review).

```
┌────────────────────────────────┬──────────────────────┐
│ Main area                      │ Document side panel  │
│ (queue / case / appeal draft)  │ (original page +     │
│                                │  highlights)         │
└────────────────────────────────┴──────────────────────┘
```

- **Toggle:** a "Show document" button opens or closes the panel; it is closed by default.
- **Position and size:** left or right side; drag to resize; option to expand to full screen.
- **Stays in sync:** clicking a field, citation or evidence item in the main area scrolls the panel to that page and highlights it.
- **Switch documents:** a dropdown in the panel lists every document on the current case.
- **Remembered:** each user's panel preference (open/closed, side, width) is saved to their profile.
- **Small screens:** the panel opens as a full-screen overlay instead.

### Corrections and learning
- Field corrections are saved with who changed what and when.
- Corrections are added to the evaluation and training set to improve extraction.

### Security
- Users see and upload documents only for cases they can access (section 4.6).
- Files stored encrypted in file/object storage; PostgreSQL keeps only metadata and a reference.
- Files open through short-lived links; no permanent public URLs.
- Downloads can be turned off by role; if allowed, they are logged.
- Every upload, view, correction and download goes to the audit log.

### Build approach
- React with PDF.js viewer: clickable highlights, zoom, rotate
- Resizable, dockable side panel (left/right) synced with the main area
