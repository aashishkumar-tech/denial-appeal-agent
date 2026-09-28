# 9. Dev B — Work Plan (Platform & Application Engineer)

**Role title:** Platform & Application Engineer
**Owns:** infrastructure, CI/CD, database platform, API, security, dashboard
**Primary languages:** Python 3.11 (FastAPI), TypeScript (React)
**GitHub handle placeholder:** `@devB`

> All names below (modules, endpoints, charts, branches, issues) are the agreed conventions. Use them exactly so the codebase stays consistent.

---

## 10.1 Scope summary

| Area | Included | Not included (Dev A) |
|---|---|---|
| Infrastructure | Oracle VM, k3s, Argo CD, Helm, ingress, TLS | — |
| CI/CD | GitHub Actions, image builds, scans | — |
| Database platform | Migrations, RLS, backups, tuning | Table design |
| API | FastAPI endpoints, auth, RBAC, audit | Business logic (calls Dev A functions) |
| Frontend | React dashboard, viewer, side panel | — |
| Security | Secrets, TLS, scanning, access tests | Prompt safety |

---

## 10.2 Naming conventions (Dev B)

### Backend modules
| Path | Purpose |
|---|---|
| `src/api/main.py` | FastAPI app factory |
| `src/api/routers/` | `auth.py`, `denials.py`, `appeals.py`, `documents.py`, `users.py`, `metrics.py`, `admin.py` |
| `src/api/deps.py` | Shared dependencies: current user, DB session, role checks |
| `src/api/security/` | `passwords.py`, `tokens.py`, `mfa.py`, `permissions.py`, `audit.py` |
| `src/api/schemas/` | Pydantic request/response models |
| `src/db/models.py` | SQLAlchemy models |
| `src/db/session.py` | Engine and session management |
| `src/db/migrations/` | Alembic versions |
| `src/db/rls.sql` | Row-level security policies |

### Frontend structure
| Path | Purpose |
|---|---|
| `frontend/src/pages/` | `LoginPage`, `HomePage`, `QueuePage`, `DenialDetailPage`, `AppealReviewPage`, `TrackingPage`, `MyItemsPage`, `AnalyticsPage`, `AdminPage` |
| `frontend/src/components/` | `DocumentViewer`, `SidePanel`, `FieldHighlight`, `ConfidenceBadge`, `CaseTimeline`, `DeadlineChip`, `RoleGuard` |
| `frontend/src/api/` | Generated client from the OpenAPI spec |
| `frontend/src/hooks/` | `useAuth`, `useCase`, `useDocuments`, `usePanelPrefs` |
| `frontend/src/types/` | Shared TypeScript types |

### Deployment files
| Path | Purpose |
|---|---|
| `deploy/helm/denial-agent/` | Chart: `Chart.yaml`, `values.yaml`, `values-dev.yaml`, `values-prod.yaml` |
| `deploy/helm/denial-agent/templates/` | `api-deployment.yaml`, `frontend-deployment.yaml`, `worker-deployment.yaml`, `postgres-statefulset.yaml`, `ingress.yaml`, `cronjob-backup.yaml`, `sealed-secrets.yaml` |
| `deploy/argocd/` | `app-dev.yaml`, `app-prod.yaml`, `project.yaml` |
| `.github/workflows/` | `ci.yml`, `build-push.yml`, `security-scan.yml` |

**Kubernetes naming:** `denial-agent-api`, `denial-agent-frontend`, `denial-agent-worker`, `denial-agent-postgres`; namespaces `denial-dev`, `denial-prod`.

**Image tags:** `ghcr.io/<org>/denial-agent-api:<git-sha>` (never `latest` in prod).

### Branch names
`feature/api-<resource>`, `feature/ui-<screen>`, `feature/infra-<component>`, `feature/sec-<control>`, `fix/<area>-<issue>`

### Commit scopes
`api`, `db`, `ui`, `deploy`, `ci`, `sec`

---

## 10.3 API endpoint naming (agreed contract)

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/auth/login` | Password + MFA → access + refresh token |
| POST | `/auth/refresh` | New access token |
| POST | `/auth/logout` | End session |
| POST | `/auth/change-password` | First login / rotation |
| GET | `/denials` | Filtered, role-scoped list |
| GET | `/denials/{denial_id}` | Detail + score + explanation |
| POST | `/denials/{denial_id}/run-agent` | Trigger agent draft |
| GET | `/appeals/{appeal_id}` | Draft + citations |
| POST | `/appeals/{appeal_id}/approve` | Approve and submit |
| POST | `/appeals/{appeal_id}/reject` | Reject with reason |
| POST | `/appeals/{appeal_id}/outcome` | Record result |
| POST | `/documents/upload` | Upload files |
| GET | `/documents/{document_id}` | Metadata + fields |
| GET | `/documents/{document_id}/pages/{page}` | Page image (short-lived URL) |
| PATCH | `/documents/{document_id}/fields` | Save user corrections |
| GET | `/cases/{case_id}/documents` | Timeline / side panel list |
| GET | `/me/items` | My cases, approvals, queries |
| GET | `/metrics` | Role-scoped KPIs |
| GET/POST | `/admin/users` | User management |
| GET | `/admin/audit-log` | Audit trail |

**Conventions:** plural resources, snake_case IDs, ISO-8601 dates, cursor pagination (`?cursor=&limit=`), errors as `{"error": {"code", "message"}}`.

---

## 10.4 Task backlog (issue IDs)

### Phase 1 — Data onboarding support (Weeks 1–4)
| ID | Task | Est. | Depends on |
|---|---|---|---|
| `P1-01` | Repo, folder skeleton, `.gitignore`, CODEOWNERS, branch protection | 1d | — |
| `P1-02` | `docker-compose.yml` (PostgreSQL + pgvector) + `.env.example` | 1d | P1-01 |
| `P1-03` | Schemas `ref`/`core`/`rag`/`ml`/`eval`/`ops` + first Alembic migration | 2d | P1-02 |
| `P1-04` | CI workflow: ruff, mypy, pytest, `pip-audit`, secret scan | 1d | P1-01 |
| `P1-14` | Document file storage service + `core.documents` metadata | 2d | P1-03 |

### Phase 1b — Platform foundation (Weeks 3–5)
| ID | Task | Est. |
|---|---|---|
| `P1b-01` | Oracle Always Free VM provisioned and hardened (firewall, SSH keys, updates) | 1d |
| `P1b-02` | k3s installed; `denial-dev` and `denial-prod` namespaces | 1d |
| `P1b-03` | Argo CD installed; `app-dev.yaml`, `app-prod.yaml` | 2d |
| `P1b-04` | Helm chart for api, frontend, worker, postgres | 3d |
| `P1b-05` | GitHub Actions: build Arm64 images → GHCR; update image tag in values | 2d |
| `P1b-06` | Sealed Secrets controller + sealed secret workflow | 1d |
| `P1b-07` | Traefik ingress + cert-manager (Let's Encrypt) + DNS | 1d |
| `P1b-08` | Backup CronJob (`pg_dump` → Oracle Object Storage) + restore test | 2d |
| `P1b-09` | Uptime Kuma + health/readiness probes | 1d |

### Phase 4 — API + Dashboard (Weeks 3–16)
| ID | Task | Est. |
|---|---|---|
| `P4-01` | FastAPI skeleton, config, DB session, error handling | 2d |
| `P4-02` | User/team/assignment models + admin user seeding | 2d |
| `P4-03` | Password hashing, lockout, forced first-login change | 2d |
| `P4-04` | JWT access + refresh tokens, session timeout | 2d |
| `P4-05` | TOTP MFA enrolment and verification | 3d |
| `P4-06` | Role and permission layer (`permissions.py`) | 3d |
| `P4-07` | Row-level security policies (`rls.sql`) + tests | 3d |
| `P4-08` | Audit log middleware | 2d |
| `P4-09` | `/denials` list and detail endpoints (role-scoped) | 3d |
| `P4-10` | `/appeals` approve / reject / outcome endpoints | 3d |
| `P4-11` | `/documents` upload, validation, virus scan, page images | 4d |
| `P4-12` | `/metrics` role-scoped KPI endpoint | 2d |
| `P4-13` | `/admin` users and audit endpoints | 2d |
| `P4-14` | React app shell, routing, auth context, `RoleGuard` | 3d |
| `P4-15` | `LoginPage` with MFA | 2d |
| `P4-16` | `QueuePage` with filters, sorting, deadline colours | 4d |
| `P4-17` | `DenialDetailPage` with "why this decision" panel | 3d |
| `P4-18` | `AppealReviewPage` with citation jump-to-source | 4d |
| `P4-19` | `DocumentViewer` (PDF.js) + field highlights | 4d |
| `P4-20` | `SidePanel`: toggle, dock left/right, resize, sync, saved prefs | 3d |
| `P4-21` | `TrackingPage` + follow-up alerts | 2d |
| `P4-22` | `MyItemsPage` + notifications | 2d |
| `P4-23` | `AdminPage`: users, thresholds, audit viewer | 3d |
| `P4-24` | `HomePage` KPI dashboard | 2d |

### Phase 5–7
| ID | Task |
|---|---|
| `P5-04` | Access tests: user A cannot read/search/export user B's data |
| `P5-05` | OWASP ZAP scan; fix high/critical findings |
| `P5-06` | Load test: queue with 10K open items < 2s |
| `P5-07` | Backup restore drill |
| `P6-03` | Production release via Argo CD; admin account creation |
| `P6-04` | User guide + support runbook |
| `P7-xx` | `AnalyticsPage`; platform patching cadence |

---

## 10.5 Security controls owned by Dev B

| Control | Implementation |
|---|---|
| Password storage | bcrypt, cost ≥ 12 |
| Lockout | 5 failures → 15-minute lock |
| MFA | TOTP, required for all users |
| Tokens | JWT access 15 min, refresh 7 days, rotation on use |
| Authorisation | Deny by default; permission matrix in `permissions.py` |
| Data isolation | API filtering **plus** PostgreSQL RLS |
| Secrets | Sealed Secrets; nothing in Git or images |
| Transport | HTTPS only, HSTS enabled |
| File access | Short-lived signed URLs, no public objects |
| Audit | Login, view, approve, reject, export, download |
| Scanning | `pip-audit`, `npm audit`, secret scan, OWASP ZAP in CI |

---

## 10.6 Definition of done (every Dev B task)
- [ ] Tests passing (`pytest` for API, `vitest` for UI)
- [ ] Endpoint documented in the OpenAPI spec
- [ ] Permission check and audit entry for every new endpoint
- [ ] Helm values updated for new config or secrets
- [ ] No secrets committed; sealed secrets used
- [ ] Docs updated in the same pull request
- [ ] Reviewed and approved by Dev A

---

## 10.7 Quality gates Dev B owns
| Metric | Target |
|---|---|
| Access tests (cross-user data) | 100% pass |
| Queue load, 10K items | < 2s |
| Availability | ≥ 99.5% |
| High/critical security findings | 0 open |
| Backup restore drill | Passes before go-live |
| Infra cost | $0 (free tier) |
