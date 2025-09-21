# Product Requirements Document (PRD)

## Real-Time Collaborative Whiteboard

**Platform:** Web (Next.js + React) + Optional Native iPad App (Swift + PencilKit)

**Prepared:** 2025-09-21

---

## 1. One-line summary
A low-latency, infinite canvas for distributed teams to ideate, draw, and map ideas together — real-time collaborative editing (CRDT-based) with excellent Apple Pencil support on iPad.

## 2. Goals and success metrics
**Primary goals**
- Deliver a delightful, low-latency whiteboard experience that supports real-time collaboration.
- Provide excellent Apple Pencil fidelity on iPad (pressure, tilt, coalesced events, double-tap where available).
- Offer a scalable, offline-friendly sync model enabling teams to work concurrently with minimal conflicts.

**Success metrics**
- Time-to-first-draw < 3s after board open.
- Median local render frame latency < 32ms.
- Median collaborator visibility latency < 150–300ms under typical network conditions.
- Concurrent session success rate ≥ 99.5% for up to 50 users/board initially.
- DAU/MAU for boards > 20% and mean session length > 15 minutes.
- < 0.5% of sessions report sync divergence or data loss.

---

## 3. Target users / personas
- **Product Managers & Facilitators:** remote workshops, retrospectives, roadmaps.
- **Designers & Engineers:** sketching ideas, flows, quick wireframes.
- **Teachers & Educators:** virtual whiteboarding in classes.
- **Enterprise Teams:** planning sessions, decision records; require SSO, audit logs.

---

## 4. Scope (MVP vs later phases)
**MVP (Phase A)**
- Infinite pan/zoom canvas
- Freehand pen, eraser, basic shapes, text boxes
- Local-first immediate rendering with pressure support (on supported browsers / devices)
- Real-time sync (CRDT via Yjs) with WebSocket provider
- Presence cursors, basic awareness (who's online)
- Save/load board snapshots (JSON + periodic PNG export)
- Simple share link + role: owner/editor/viewer

**Phase B (Collaboration & Productivity)**
- Undo/redo syncing
- Sticky notes, image upload, templates
- Comments & mentions, simple chat
- CRDT optimizations & versioned snapshots
- Export to PNG/PDF and Google Drive integration

**Phase C (Scale & Enterprise)**
- Offline sync & conflict reconciliation UI
- Tablet / iPad native app with PencilKit (high-fidelity Pencil support)
- SSO (SAML/SCIM), audit logs, team admin features
- Large-board performance (tiling, lazy-load), 500+ concurrent users/board scaling

---

## 5. Key features & priority
### Must-have (MVP)
- Infinite zoomable canvas
- Local immediate rendering of strokes (client-side)
- Compact stroke model (vector with pressure/timestamp) and batching
- Real-time CRDT sync using Yjs
- Presence (avatars, cursors)
- Save/load snapshots; basic sharing & permissions

### Should-have (Phase B)
- Undo/redo across collaborators
- Comments, reactions, lightweight chat
- Templates and media attachments (images)
- Export & basic integrations (Google Drive, Slack)

### Nice-to-have (Phase C+)
- Native iPad app with PencilKit + Metal renderer
- Hover/preview cursor for supported hardware
- Advanced shape recognition and auto-layout tools
- Workspace/organization features (folders, teams, templates marketplace)

---

## 6. Apple Pencil & iPad-specific requirements
- Support pressure (`pressure` / `UITouch.force`) and map to stroke thickness.
- Support coalesced and predicted events for smooth strokes.
- Support tilt/azimuth for brush angle on native app.
- Handle Pencil 2 double-tap using `UIPencilInteraction` (native) for tool toggle.
- Implement palm rejection when stylus detected; allow explicit finger-mode toggle.
- Web fallback: use Pointer Events (`pointerType === 'pen'`) and `getCoalescedEvents()`; document browser limitations.

Acceptance criteria for Pencil support:
- Pressure mapping is fluid across range on supported devices.
- No visible gaps or jitter (coalesced/predicted sampling used).
- Palm-rejection prevents accidental marks while writing.
- Double-tap toggles between pen/eraser or last-tool on Pencil 2 (native only).

---

## 7. UX / User flows (summary)
**Create a board**
1. User clicks "New board" on dashboard. 2. Choose blank or template. 3. Board opens; user can draw immediately.

**Invite collaborators**
1. Click Share → choose role (view/edit) → copy link or invite email. 2. Collaborators open link and join; presence indicators appear.

**Live collaboration**
- Each user sees others’ cursors and in-progress strokes (live ink). Comments attached to objects.

**Save / Export**
- Autosave every 5–10s; manual snapshot and export to PNG/PDF.

---

## 8. Data model (simplified)
```json
Board {
  id: string,
  title: string,
  ownerId: string,
  visibility: 'private'|'team'|'public',
  createdAt: timestamp,
  updatedAt: timestamp,
  latestSnapshotUrl: string
}

Object (stroke/shape/image) {
  id: string,
  boardId: string,
  type: 'stroke'|'shape'|'text'|'image'|'sticky',
  properties: {...},
  zIndex: number,
  createdBy: userId,
  lastModified: timestamp
}

Event {
  id: string,
  boardId: string,
  opType: 'strokeStart'|'strokeAppend'|'strokeEnd'|'objectUpdate',
  payload: {...},
  timestamp: number,
  authorId: string
}

Presence {
  boardId: string,
  userId: string,
  cursor: {x,y},
  selection: [...],
  lastSeen: timestamp
}
```

Notes: primary source-of-truth is CRDT state (Yjs). Append-only event log used for analytics, auditing and replays; periodic snapshots persisted to object storage.

---

## 9. Sync & architecture (recommended)
**High-level**
- Clients run Yjs CRDT instance for the board state.
- Transport: `y-websocket` provider over WebSocket (or `y-webrtc` for P2P fallbacks).
- Presence: Yjs Awareness module + Redis for server-side ephemeral presence if needed.
- Server: lightweight WebSocket relay (y-websocket or custom) + API service for boards, auth, snapshots.
- Persistence: periodic server-side snapshots of Yjs state saved to S3; metadata in Postgres.
- Event streaming: Kafka/Redis Streams for analytics and background processing (snapshotting, export generation).

**Why CRDT (Yjs)**
- Offline-first friendliness and automatic merges.
- Small messages and well-tested ecosystem for rich collaboration.

---

## 10. API surface (examples)
```
POST /api/boards                  # create
GET  /api/boards/:id              # metadata
GET  /api/boards/:id/snapshot     # latest snapshot
POST /api/boards/:id/share        # create invite
GET  /api/boards/:id/history      # snapshots & events
POST /api/auth/login              # auth
```
Real-time events travel over WebSocket (y-websocket) instead of REST for collaborative ops.

---

## 11. Non-functional requirements
- **Latency:** local render latency < 32ms per frame; collaborator updates median < 150–300ms.
- **Scalability:** initially support 50 concurrent users per board; design to scale to 500+ with sharding/tiling.
- **Availability:** 99.9% uptime for core services.
- **Durability:** snapshot + event logs with backups; RPO/RTO targets in enterprise plan.
- **Security:** TLS everywhere, auth via OAuth2/OIDC, RBAC server-side checks, optional SAML for enterprise.

---

## 12. Monitoring & telemetry
- Track: update latency, presence counts, autosave failures, sync divergence events, active users, board sizes.
- Tools: Prometheus + Grafana, Sentry, OpenTelemetry traces.
- Alerts: high median latency, queue/backlog growth, error rate spikes.

---

## 13. Testing & QA
**Acceptance tests**
- Functional: stroke fidelity, presence, save/load, share permissions.
- Performance: render frame latency, large board interactions, concurrency tests (multiple simulated clients).
- Cross-platform: Safari on iPad pointer & pressure, Chrome/Edge on desktop, native iPad app (if implemented).

**Manual QA checklist for Apple Pencil**
- Pressure mapping, tilt influence, double-tap behavior (Pencil 2), palm rejection, coalesced events rendering.

**E2E & automated**
- Playwright/Cypress for UI flows (create/invite/save/export).
- Synthetic load tests for the realtime server (k6) simulating many clients per board.

---

## 14. Security & privacy
- Default boards private; share-by-link ephemeral tokens with expiry.
- RBAC enforced server-side for every API & event message.
- Data encryption in transit (TLS) and at rest (S3 encryption, DB encryption where available).
- Enterprise compliance options: data residency, SSO (SAML), audit logs, SCIM provisioning.

---

## 15. Roadmap & milestones (12-week plan)
**Phase A — MVP (Weeks 1–4)**
- Core canvas, drawing tools, local rendering, basic Yjs sync, presence, save/load, share link.

**Phase B — Collaboration (Weeks 5–8)**
- CRDT/undo, comments, media/templates, export, role-based sharing.

**Phase C — Scale & Native (Weeks 9–12)**
- Offline sync, iPad native app (PencilKit), performance optimizations, SSO & audit logs.

---

## 16. Risks & mitigations
- **Sync divergence / merge issues**: use mature CRDT libs (Yjs); build reconciliation UI and extensive integration tests.
- **Large board performance**: implement viewport tiling, object culling, and progressive snapshotting.
- **Apple Pencil browser limitations**: provide a native iPad app path for pro users; document known Safari limitations.
- **Security leakage through share links**: make links short-lived and revokable; default private boards.

---

## 17. Go-to-market & business model
- **Freemium:** Free tier with board and collaborator limits; paid plans for team features (SSO, audit logs, unlimited boards, storage).
- **Enterprise:** SAML/SCIM, data residency, SLAs, dedicated support.
- **Partnerships & integrations:** Google Workspace, Slack, Microsoft Teams, and templates marketplace.

---

## 18. Open questions
- Do we prioritize native iPad app in parallel with web MVP or defer until Phase C?
- Which analytics & observability budget is allocated (managed vs self-hosted)?
- Do we require strict data residency for initial customers?

---

## 19. Appendix
**Glossary**: CRDT, Yjs, WebSocket, WebRTC, PencilKit, coalesced events, autosave.

**Related docs**: Architecture sketch, Tech-stack README, UX comps (to be attached).

---

*End of PRD*

