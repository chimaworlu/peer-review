# Peer Code Reviews

## Review 1 — ShaDowCRYT / PropAPI
**PR reviewed:** https://github.com/ShaDowCRYT/PropAPI/pull/1
**What they built:** A GET /api/v1/agents/:id/listings endpoint that returns all property listings belonging to one agent, with pagination and sorting. Built with Express and Prisma. The same PR also updates their background jobs documentation.

---

### What they did well

**1. Agent existence check before listings query**
The code checks if the agent exists before running any pagination or listing query. If the agent does not exist the request fails immediately with a 404 without touching the listings table. This is the correct order — validate the parent resource exists before fetching its children.

**2. Parallel database queries with Promise.all**
The total count and the listings fetch run in parallel using Promise.all instead of sequentially. This halves the database round trips for every paginated request. One query does not wait for the other to finish.

**3. Explicit sort field allowlist**
The ALLOWED_SORT_FIELDS array is declared right before it is used. It is easy to see exactly which fields can be sorted and adding a new one is one line. No magic strings buried in a shared utility.

**4. Background jobs documentation is thorough**
The TASK2-BACKGROUND-JOBS.md is genuinely detailed. The design decisions section explains atomic claiming with FOR UPDATE SKIP LOCKED, jitter in backoff, and why processJob is not awaited. The defence prep section answers real questions with specific line references. The evidence table lists what each test proved, not just that tests were run.

---

### One specific thing that could be improved

In src/routes/agents.ts, the listings query uses where: { agentId: id } with no filter for soft-deleted or inactive listings. If the Listing model has a deletedAt or isActive field, this endpoint would return deleted or inactive listings to callers. Check the Listing schema and if either field exists add deletedAt: null or isActive: true to the where clause.

---

### One question to ask in a defence

The Promise.all runs the count query and the listings query at the same time. If a listing is inserted between the moment the count query runs and the moment the listings query runs, the total count and the actual data will be inconsistent. In what scenario would this matter for a property listings API, and would you fix it or accept it?


## Review 2 — angel-madueme / peerup-api
**PR reviewed:** https://github.com/angel-madueme/peerup-api/pull/1
**What they built:** A fix to the seed script for a study group API. Every seeded session was previously assigned the same start time (12:00 PM). This PR introduces a set of six realistic study session time slots and distributes them across the 400 seeded sessions.

---

### What they did well

**1. Declarative time slots in one place**
The sessionTimeSlots array defines all six time slots in a single location. Adding or changing a slot is one line. The values are readable as real times — 9:00, 11:00, 14:00 — not magic numbers buried in arithmetic.

**2. Clean, focused diff**
The change touches only the seed script. No runtime code, no schema changes, no routes. The PR does exactly one thing and nothing else. That makes it easy to review and easy to revert if needed.

**3. Honest PR description**
The author noted clearly that the seed had not been run against the database before the PR was opened. That kind of honesty in a PR description is the right behaviour — it flags what the reviewer should verify rather than hiding it.

---

### One specific thing that could be improved

The time slot is selected using index % sessionTimeSlots.length. Because study groups repeat every 60 sessions and there are 6 slots, 60 is divisible by 6. This means each study group consistently receives the same slot across all its seeded sessions — session group 0 always gets 9:00, group 1 always gets 11:00, and so on. The fix is to include the group index or use Math.floor(index / 60) % sessionTimeSlots.length so that each round through the groups gets a different slot. This was also flagged by another reviewer in the PR with a working code suggestion.

---

### One question to ask in a defence

The last two time slots are 18:00 and 19:30. A session starting at 18:00 lasts 90 minutes and ends at 19:30 — exactly when the next slot begins. Was that intentional to create back-to-back sessions with no gap, or is it an oversight? The earlier slots each have a 30-minute gap between the end of one session and the start of the next.

---

## Review 3 — jay-inventor007 / engineering-task-1-consumable-api
**PR reviewed:** https://github.com/jay-inventor007/engineering-task-1-consumable-api/pull/1
**What they built:** Full-text search on a property listings API. A new database migration adds a generated search_vector column and a GIN index to the listings table. A new ?q= query parameter on GET /listings and GET /agents/:id/listings runs a full-text search across title, description, address, city, and state. The consumer page gets a debounced search box.

---

### What they did well

**1. Database-level full-text search with a GIN index**
The search runs against a generated tsvector column with a GIN index. This is the correct PostgreSQL approach — the search_vector is computed once at write time and indexed, so search queries do not scan every row and do not recalculate the vector on every read. A LIKE query on each column would be dramatically slower at scale.

**2. websearch_to_tsquery with parameterized input**
The query uses websearch_to_tsquery with a parameterized value. This handles quoted phrases, punctuation, and multi-word searches without SQL injection risk and without requiring the caller to know PostgreSQL's tsquery syntax. A caller can type "duplex ikoyi" and it works as expected.

**3. Debounced search on the consumer**
The consumer debounces the search input before calling the API. This means the API is not called on every keystroke. That is the correct approach for any search input connected to a network call.

**4. Consistent filter composition**
The search filter is added to the same filter builder as minPrice, maxPrice, and minBedrooms. It composes cleanly with existing filters. A caller can combine ?q=duplex&minPrice=5000000 and both filters apply together.

---

### One specific thing that could be improved

In consumer/src/App.tsx, when the trimmed search input drops to exactly one character the useEffect returns early and does not fire a new request. This is correct — the API requires at least two characters. But it leaves the previous filter value active. If a user searched for "duplex" and then backspaced to "d", the input shows "d" but the page still shows duplex results. The fix is to also clear the active search filter when the trimmed query length is less than 2, not just skip the request. This was flagged by the other reviewer in the PR with a reproduction step.

---

### One question to ask in a defence

The GIN index is on the search_vector column, which is generated from title, description, address, city, and state. If a user searches for a postcode or a specific price range using the q parameter, the search will return no results because postcodes and numbers are not in the tsvector. How would you communicate this limitation to API consumers, and would you consider adding numeric fields to the search vector?
