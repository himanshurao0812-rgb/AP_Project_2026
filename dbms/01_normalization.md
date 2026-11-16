# TalkArena — Database Normalization (1NF → 3NF → BCNF)

**AP Capstone · Team T0122** · Deliverable: `dbms/01_normalization.md` (checkpoint CP1)
Project: **TalkArena — AI-Powered Group Discussion Platform** (React · FastAPI · PostgreSQL · WebSockets · WebRTC · AI API)

> **Scope of this document.** The CP1 brief requires *"the data of your AP app taken through 1NF → 2NF → 3NF, and BCNF where possible. Show the tables and the functional dependencies at each step."* This document starts from the raw room-activity export an early version of TalkArena would produce (§1), and normalises it in four steps, stating the functional dependencies (FDs) and the exact violation removed at each step.

---

## 0. What data TalkArena actually stores

A discussion room is where everything happens: users join, chat, optionally start a video call, and may switch on AI Mode so an AI persona joins as a participant. Fact-checks attach to individual messages.

| Data | Where it comes from |
|---|---|
| Users (handle, email, password) | Sign-up |
| Rooms (name, invite code, host, topic, mode) | "Create a discussion room" |
| Membership (who joined which room, role, joined/left time) | Invite link / room controls |
| Messages (text, sender, time) | WebSocket chat |
| AI personas (name, style, model) and which room uses which persona | AI Mode toggle |
| Video sessions and who attended | WebRTC calling |
| Fact-checks (claim, verdict, source) | AI Fact Checking feature |

---

## 1. Step 0 — Un-normalised form (UNF)

The naïve version of TalkArena stores a room the way a spreadsheet would: everything about one room in **one row**, with the participants and the chat packed into single cells.

**`room_activity_export` (UNF)**

| room_id | room_name | topic | topic_category | host_email | mode | participants | chat_log |
|---|---|---|---|---|---|---|---|
| R1 | Avengers Theories | Avengers: Doomsday | Movies | aarav@college.edu | Movie Talk | `Aarav (host); Diya; Rohan` | `[1] Diya @10:02 — "Loki is alive"; [2] AI @10:03 — "What if…"; [3] Rohan @10:05 — "Bold theory"` |
| R2 | DSA Doubts | Binary Trees | Coding | diya@college.edu | Study Mode | `Diya (host); Kabir` | `[1] Kabir @11:10 — "AVL rotation doubt"; [2] AI @11:11 — "Great question"` |

**Why this is not even 1NF:** two attributes hold **repeating groups / multi-valued data** — `participants` (a semicolon-separated list) and `chat_log` (an entire conversation inside one cell). No attribute may hold more than one value.

---

## 2. Step 1 — First Normal Form (1NF)

**Rule applied:** every attribute holds a single atomic value; no repeating groups; each row is identified by a key.

Two repeating groups are pulled out, giving two tables. Because a room now repeats once per message, we number messages **within** a room — that is how the export actually identifies them.

**`room_messages_1nf`** — PK `(room_id, message_no)`

| room_id | message_no | room_name | topic | topic_category | host_email | mode | sender_email | message_text | sent_at |
|---|---|---|---|---|---|---|---|---|---|
| R1 | 1 | Avengers Theories | Avengers: Doomsday | Movies | aarav@college.edu | Movie Talk | diya@college.edu | Loki is alive | 10:02 |
| R1 | 2 | Avengers Theories | Avengers: Doomsday | Movies | aarav@college.edu | Movie Talk | ai@talkarena | What if… | 10:03 |
| R1 | 3 | Avengers Theories | Avengers: Doomsday | Movies | aarav@college.edu | Movie Talk | rohan@college.edu | Bold theory | 10:05 |
| R2 | 1 | DSA Doubts | Binary Trees | Coding | diya@college.edu | Study Mode | kabir@college.edu | AVL rotation doubt | 11:10 |

**`room_members_1nf`** — PK `(room_id, participant_email)`

| room_id | participant_email | participant_name | role | joined_at |
|---|---|---|---|---|
| R1 | aarav@college.edu | Aarav | Host | 09:55 |
| R1 | diya@college.edu | Diya | Member | 10:00 |
| R1 | rohan@college.edu | Rohan | Member | 10:01 |

**1NF is satisfied, but the design is still bad:** `room_name`, `topic`, `topic_category`, `host_email` and `mode` are repeated on **every message row** of a room. That is redundant storage, and it means the same fact can be updated inconsistently ("Avengers Theories" renamed in one row and not another) — an **update anomaly**.

---

## 3. Step 2 — Second Normal Form (2NF)

**Rule applied:** 1NF **and** every non-key attribute depends on the **whole** primary key, not part of it (no partial dependency).

### 3.1 `room_messages_1nf` → violation

Key = `(room_id, message_no)`. The FDs are:

```
room_id, message_no  →  sender_email, message_text, sent_at        (depends on the whole key ✓)
room_id              →  room_name, topic, topic_category,
                        host_email, mode                            (depends on PART of the key ✗)
```

`room_id` alone determines the room's name, topic, host and mode — a **partial dependency**. Everything about the *room* was being stored once per *message*.

### 3.2 `room_members_1nf` → violation

Key = `(room_id, participant_email)`:

```
room_id, participant_email  →  role, joined_at      (whole key ✓)
participant_email           →  participant_name     (part of the key ✗)
```

The member's **name** is a fact about the **person**, not about their membership of this room.

### 3.3 Decomposition (2NF)

```
rooms_2nf    (room_id, room_name, topic, topic_category, host_email, host_name, mode)
messages_2nf (room_id, message_no, sender_email, sender_name, message_text, sent_at)
members_2nf  (room_id, participant_email, role, joined_at)
people_2nf   (participant_email, participant_name)
```

**Effect:** a room's name is now stored **once**, so a rename cannot disagree with itself. The message table is smaller and no longer repeats the room's details.

---

## 4. Step 3 — Third Normal Form (3NF)

**Rule applied:** 2NF **and** no non-key attribute depends on another non-key attribute (no transitive dependency).

### 4.1 `rooms_2nf` → violations

Key = `room_id`. Non-key attributes: `room_name, topic, topic_category, host_email, host_name, mode`.

```
room_id    →  host_email          (direct)
host_email →  host_name           (transitive!)  room_id → host_email → host_name
room_id    →  topic               (direct)
topic      →  topic_category      (transitive!)  room_id → topic → topic_category
room_id    →  mode                (lookup value repeated as free text)
```

Three problems: the host's **name** is a fact about a **user**; the topic's **category** is a fact about a **topic**; `mode` is a small closed list ("Study Mode", "Debate", "Movie Talk") repeated as text in every room.

### 4.2 `messages_2nf` → violation

Key = `(room_id, message_no)`:

```
room_id, message_no → sender_email
sender_email        → sender_name      (transitive!)
```

### 4.3 `people_2nf` → violation

Key = `participant_email`. Once a user profile grows (`handle`, `password_hash`, `created_at`, `is_active`), keeping them keyed by email forces a user's identity to change if their email changes — the classic reason an email is a **candidate key, not a primary key**.

### 4.4 Decomposition (3NF)

```
users           (user_id, handle, email, full_name, password_hash, created_at, is_active)
topics          (topic_id, topic_name, category)
discussion_modes(mode_id, mode_name, description)
rooms           (room_id, room_name, invite_code, host_user_id, topic_id, mode_id,
                 ai_enabled, status, created_at, ended_at)
messages        (message_id, room_id, author_user_id, persona_id, body, sent_at)
room_members    (room_id, user_id, role, joined_at, left_at)
```

Plus the tables introduced by the AI, video and fact-checking features (normalised the same way — each was a repeating group or a transitive dependency in the export):

```
ai_personas       (persona_id, persona_name, mode_id, model_name, system_prompt)
room_ai_persona   (room_id, persona_id)
video_sessions    (session_id, room_id, started_at, ended_at, recording_url)
video_participants(session_id, user_id, joined_at, left_at)
fact_checks       (check_id, message_id, claim_text, verdict, source_url, checked_at, persona_id)
```

**3NF status: achieved.** No non-key attribute determines another non-key attribute in any of the 11 tables.

---

## 5. Step 4 — Boyce-Codd Normal Form (BCNF)

**Rule applied:** every **determinant** is a **candidate key**.

For 10 of the 11 tables this holds trivially, because each has a single surrogate key that is the only determinant:

| Table | Determinants | Candidate keys | BCNF? |
|---|---|---|---|
| `users` | `user_id`, `handle`, `email` | `user_id`, `handle`, `email` | ✅ all determinants are keys |
| `topics` | `topic_id`, `topic_name` | `topic_id`, `topic_name` | ✅ |
| `discussion_modes` | `mode_id`, `mode_name` | both | ✅ |
| `ai_personas` | `persona_id` | `persona_id` | ✅ |
| `rooms` | `room_id`, `invite_code` | both | ✅ |
| `messages` | `message_id` | `message_id` | ✅ |
| `room_members` | `(room_id, user_id)` | that pair only | ✅ |
| `room_ai_persona` | `(room_id, persona_id)` | that pair only | ✅ |
| `video_sessions` | `session_id` | `session_id` | ✅ |
| `video_participants` | `(session_id, user_id)` | that pair only | ✅ |
| `fact_checks` | `check_id` | `check_id` | ✅ |

### 5.1 One table that is 3NF but **not** BCNF — and how we fixed it

TalkArena's rule is: **"an AI persona is built for exactly one discussion mode"** (the *Study Buddy* persona only ever runs in Study Mode), and **"a room uses at most one persona per mode"**. If we stored the persona directly with the room:

**`room_ai` (3NF, but not BCNF)**

| Attribute | Meaning |
|---|---|
| `room_id` | the room |
| `mode_id` | the room's discussion mode |
| `persona_id` | the persona the AI Mode toggle selected |

Functional dependencies:

```
room_id, mode_id → persona_id      (a room has one persona per mode)
persona_id       → mode_id         (a persona belongs to exactly one mode)
```

Candidate keys: `(room_id, mode_id)` and `(room_id, persona_id)`.

**The BCNF test fails on the second FD:** `persona_id → mode_id` has a determinant (`persona_id`) that is **not a superkey**. Note it *passes* 3NF, because `mode_id` is **prime** (part of the candidate key `(room_id, mode_id)`) and 3NF only forbids non-prime attributes depending on non-keys. This is exactly the case where 3NF and BCNF disagree.

**Decomposition (BCNF):**

```
ai_personas     (persona_id, persona_name, mode_id, model_name, system_prompt)   -- persona_id → all
room_ai_persona (room_id, persona_id)                                            -- (room_id, persona_id) → nothing
```

`persona_id → mode_id` now lives inside `ai_personas`, where `persona_id` **is** the key — so the BCNF violation is gone, and a persona's mode can no longer be stated twice.

**The honest trade-off (important).** BCNF removed the redundancy but also removed the database's ability to enforce *"the persona's mode must equal the room's mode"* declaratively — the room's `mode_id` now lives in `rooms` and the persona's in `ai_personas`. We accept this and enforce the rule in the application layer (validated when the AI toggle is switched on, and re-checked by a trigger before insert into `room_ai_persona`). This is documented rather than hidden: **a normalisation that silently drops a business rule is worse than a redundant column.**

---

## 6. Final schema — tables, keys and functional dependencies

| Table | Primary key | Candidate keys | Functional dependencies | Normal form |
|---|---|---|---|---|
| `users` | `user_id` | `user_id`, `handle`, `email` | `user_id → handle, email, full_name, password_hash, created_at, is_active`; `handle → user_id`; `email → user_id` | BCNF |
| `topics` | `topic_id` | `topic_id`, `topic_name` | `topic_id → topic_name, category` | BCNF |
| `discussion_modes` | `mode_id` | `mode_id`, `mode_name` | `mode_id → mode_name, description` | BCNF |
| `ai_personas` | `persona_id` | `persona_id` | `persona_id → persona_name, mode_id, model_name, system_prompt` | BCNF |
| `rooms` | `room_id` | `room_id`, `invite_code` | `room_id → room_name, invite_code, host_user_id, topic_id, mode_id, ai_enabled, status, created_at, ended_at` | BCNF |
| `room_members` | `(room_id, user_id)` | same | `(room_id, user_id) → role, joined_at, left_at` | BCNF |
| `room_ai_persona` | `(room_id, persona_id)` | same | `(room_id, persona_id) → (no non-key attributes)` | BCNF |
| `messages` | `message_id` | `message_id` | `message_id → room_id, author_user_id, persona_id, body, sent_at` | BCNF |
| `video_sessions` | `session_id` | `session_id` | `session_id → room_id, started_at, ended_at, recording_url` | BCNF |
| `video_participants` | `(session_id, user_id)` | same | `(session_id, user_id) → joined_at, left_at` | BCNF |
| `fact_checks` | `check_id` | `check_id` | `check_id → message_id, claim_text, verdict, source_url, checked_at, persona_id` | BCNF |

### 6.1 Relationships and cardinality

| Relationship | Cardinality | Resolved by |
|---|---|---|
| users ⇄ rooms | **M:N** | `room_members` (composite PK) |
| users → rooms (as host) | 1:N | `rooms.host_user_id` |
| rooms → messages | 1:N | `messages.room_id` |
| ai_personas → messages | 1:0..N | `messages.persona_id` (nullable) |
| ai_personas ⇄ rooms | **M:N** | `room_ai_persona` |
| rooms → video_sessions | 1:N | `video_sessions.room_id` |
| users ⇄ video_sessions | **M:N** | `video_participants` (composite PK) |
| messages → fact_checks | 1:N | `fact_checks.message_id` |
| topics → rooms | 1:N | `rooms.topic_id` |
| discussion_modes → rooms | 1:N | `rooms.mode_id` |

### 6.2 One deliberate denormalisation — `messages` uses an exclusive arc

A message is written by a human **or** by an AI persona. Storing both `author_user_id` and `persona_id` as nullable columns and allowing exactly one is technically two optional FKs in one row, which a pure normal-form analysis would split into two tables (`user_messages`, `ai_messages`). We keep **one** `messages` table because:

1. Chat is read as **one time-ordered stream** per room; splitting it would make the core query a `UNION ALL` of two tables sorted together — slower and more complex, on the hottest path in the app.
2. A CHECK constraint enforces the rule declaratively: `num_nonnulls(author_user_id, persona_id) = 1` — so the "one author" invariant is still guaranteed by the database, not just by application code.

This is the same reasoning used in our DE project for derived columns: **denormalisation is acceptable when it is deliberate, documented and constrained.**

---

## 7. Summary of what each step removed

| Step | Violation removed | Example in TalkArena | Tables before → after |
|---|---|---|---|
| **UNF → 1NF** | Repeating groups / multi-valued cells | `participants = "Aarav; Diya; Rohan"`, `chat_log = "[1] Diya…[2] AI…"` | 1 → 2 |
| **1NF → 2NF** | Partial dependency | `room_id → room_name` stored once per **message** | 2 → 4 |
| **2NF → 3NF** | Transitive dependency | `room_id → topic → topic_category`; `user_id → email → full_name`; free-text `mode` | 4 → 6 (+5 feature tables) |
| **3NF → BCNF** | Determinant that is not a superkey | `persona_id → mode_id` in the old `room_ai` table | 11 tables, all BCNF |

**Result: 11 tables, every one in BCNF**, no update/insert/delete anomaly — a room's name is stored once, a user's email once, an AI persona's mode once.

---

## 8. Where each table is used in the app

| Feature in the README | Tables touched |
|---|---|
| "Create and join discussion rooms" | `rooms`, `room_members`, `topics`, `discussion_modes` |
| "Implement group chat" (WebSockets) | `messages` |
| "Add AI participation" | `ai_personas`, `room_ai_persona`, `messages.persona_id` |
| "Add video calling" (WebRTC) | `video_sessions`, `video_participants` |
| "Different discussion modes" / "AI personalities" | `discussion_modes`, `ai_personas` |
| "AI fact checking" | `fact_checks` |
| "Analyzing popular topics, active rooms, user engagement" | aggregates over `rooms`, `messages`, `room_members` |

---

## 9. Next deliverables (build on this document)

| File | Content | Checkpoint |
|---|---|---|
| `dbms/03_schema.sql` | `CREATE TABLE` for all 11 tables, keys + constraints, runs on PostgreSQL | CP2 |
| `dbms/02_er_diagram.png` | Entities, attributes, PK/FK, cardinality | CP2 |
| `dbms/04_queries.sql` | Every query the app uses, marked `-- READ`/`-- WRITE` with ids | CP3 |
| `dbms/05_query_timings.md` + `dbms/plans/` | 3 runs per query, average, scan type | CP4 |
