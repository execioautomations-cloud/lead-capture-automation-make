## Demo

[▶️ Watch the 3-minute walkthrough](YOUR_LOOM_LINK)

![Make scenario](make-scenario.png)
![Airtable records](airtable.png)
![Slack alerts](slack.png)
1	# Lead Capture Automation — Google Forms → Airtable → Slack
2	
3	Automated intake pipeline that captures inbound quote requests, deduplicates them
4	against an existing CRM table, validates the data, and alerts the sales team to
5	high-value opportunities in real time.
6	
7	Built with **Make.com** (no-code orchestration), **Airtable** (data store) and
8	**Slack** (notifications).
9	
10	---
11	
12	## The problem
13	
14	An agency receives quote requests through a Google Form. Requests were being
15	copied into Airtable by hand, which meant:
16	
17	- Leads sat unprocessed until someone did the data entry
18	- High-value enquiries weren't flagged, so follow-up was slow
19	- Repeat enquiries from the same client created duplicate records
20	- Nobody knew when a submission was incomplete
21	
22	## The solution
23	
24	A Make.com scenario that runs on a schedule, processes each submission, and
25	routes it down one of three mutually exclusive paths.
26	
27	---
28	
29	## Architecture
30	
31	```
32	[Google Forms: Watch Responses]        (module 4, limit 3 per run)
33	            │
34	            ▼
35	[Airtable: Search by email]            (module 24 — raw API call)
36	   filterByFormula: LOWER({Email}) = lower(trim(submitted_email))
37	            │
38	            ▼
39	        [ROUTER]  (module 19)
40	            │
41	   ┌────────┼─────────────────────────────┐
42	   │        │                             │
43	   ▼        ▼                             ▼
44	INCOMPLETE  NEW LEAD                  EXISTING LEAD
45	(26, 28)    (2, 12)                   (21, 22)
46	
47	INCOMPLETE     : email OR budget missing
48	                 → create record, Status = "Needs Review"
49	                 → Slack ⚠️ alert
50	
51	NEW LEAD       : 0 search results AND email exists AND budget exists
52	                 → create record, Status = "New"
53	                 → if budget >= 5000: Slack 🔥 high-value alert
54	
55	EXISTING LEAD  : search returned a record ID AND email exists AND budget exists
56	                 → update existing record, append to Notes
57	                 → Slack 🔁 repeat-enquiry alert (always, regardless of value)
58	```
59	
60	---
61	
62	## Real-world problems handled
63	
64	This is the part that matters. A happy-path version of this automation takes
65	twenty minutes to build. The following edge cases are what make it survive
66	contact with real users.
67	
68	### 1. Currency symbols in a numeric field
69	
70	Google Forms short-answer fields always return **text**. Users type `9000 $`,
71	`$9,000`, `9k`. Writing that into an Airtable Number field fails, and — worse —
72	a numeric filter comparison against dirty text silently evaluates false, so the
73	high-value alert never fires and the lead is lost without any error.
74	
75	Handled at two layers:
76	
77	- **Source**: Google Form response validation set to Number
78	- **Pipeline**: `parseNumber(replace(value; "/[^0-9.]/g"; ""))` strips any
79	  non-numeric character before parsing
80	
81	Fixing it at the source is the cheaper fix; the pipeline layer defends against
82	the form being edited later.
83	
84	### 2. Missing fields disappear entirely
85	
86	When a Google Form question is left blank, the key is **absent from the JSON**,
87	not present-and-empty. Any mapping that assumes it exists resolves to nothing.
88	
89	Handled with `ifempty(value; fallback)` on every optional field, plus a
90	dedicated validation route so incomplete submissions are flagged as
91	`Needs Review` rather than stored as half-empty records nobody notices.
92	
93	### 3. Duplicate submissions
94	
95	The same client submitting twice created two records, so sales called twice.
96	
97	Handled with an **upsert**: search Airtable by email before writing, then branch
98	to create-or-update. New enquiry details are appended to the existing record's
99	Notes field rather than overwriting it.
100	
101	### 4. Email case sensitivity
102	
103	`SALES@EXAMPLE.COM` and `sales@example.com` are the same person. A naive string
104	comparison treats them as different and defeats the deduplication entirely.
105	
106	Both sides are normalised before comparison — Airtable's `LOWER()` on the stored
107	value, and Make's `lower(trim(...))` on the incoming value.
108	
109	### 5. Timezone drift on date fields
110	
111	Google returns timestamps in UTC (`2026-10-01T20:45:56Z`). Stored raw, an
112	evening submission from India (UTC+5:30) records as **the following day**.
113	
114	Handled with `formatDate(createTime; "YYYY-MM-DDTHH:mm:ssZ"; "Asia/Kolkata")`.
115	
116	### 6. Overlapping router routes
117	
118	Make executes **every** matching route, not just the first. A submission with an
119	email but no budget initially matched both "New lead" and "Incomplete",
120	producing two records for one submission.
121	
122	Fixed by making the three route filters mutually exclusive — the valid routes
123	require both email and budget to exist; the incomplete route fires when either
124	is missing.
125	
126	---
127	
128	## Error handling
129	
130	| Module | Directive | Rationale |
131	|---|---|---|
132	| Airtable Create Record (new lead) | Slack alert → **Ignore** | Alert the team, continue with remaining submissions |
133	| Airtable Update Record | Slack alert → **Ignore** | Same |
134	| Airtable Create Record (incomplete) | Slack alert → **Ignore** | Same |
135	| Airtable Search | **Break** | A failed search means create-vs-update can't be decided safely — park the bundle for retry rather than guess and risk a duplicate |
136	
137	**Scenario settings:**
138	
139	- *Allow storing of incomplete executions* — **enabled**. Failed bundles are
140	  queued and can be re-run after the underlying problem is fixed, rather than
141	  being lost.
142	- *Consecutive errors before deactivation* — 3
143	
144	The choice of directive per module is deliberate: the question asked at each
145	step was *"what is the safest way for this specific step to fail?"*
146	
147	---
148	
149	## Setup
150	
151	### Airtable — table `Requests`
152	
153	| Field | Type | Options |
154	|---|---|---|
155	| Name | Single line text | |
156	| Email | Email | |
157	| Service Needed | Single select | Web Design, SEO, Consulting, Unknown |
158	| Budget | Number | |
159	| Status | Single select | New, Contacted, Closed, Needs Review |
160	| Received At | Date (with time) | |
161	| Notes | Long text | |
162	| Phone | Single line text | |
163	
164	### Google Form
165	
166	Questions: Full Name, Email, Phone Number, Country, Service Needed,
167	Estimated Budget.
168	
169	Name, Email, Service Needed and Estimated Budget are set to **Required**;
170	Estimated Budget additionally uses **Response validation → Number**.
171	
172	### Make.com
173	
174	Import the blueprint, then reconnect:
175	
176	- Google (Forms read)
177	- Airtable (personal access token, scoped to the base)
178	- Slack (see limitation below)
179	
180	Update the Airtable base/table IDs and the Slack channel ID to match your own.
181	
182	---
183	
184	## Known limitations
185	
186	Listed honestly, because undocumented limitations become someone else's
187	production incident.
188	
189	1. **`replace()` relies on implicit behaviour.** Two expressions omit the
190	   third (replacement) argument. Tested with input `7000 $`, which correctly
191	   stored as the number `7000` — Make treats the missing argument as an empty
192	   string. This works, but depends on undocumented behaviour that could change.
193	   The argument should be stated explicitly as `""`.
194	
195	2. **Slack uses a user connection, not a bot.** Messages post as an individual
196	   user. For production this should be a bot connection, otherwise the
197	   automation breaks if that user leaves the workspace and every alert appears
198	   to come from a person.
199	
200	3. **Polling, not instant.** Google Forms is polled on a schedule, so there is a
201	   delay between submission and processing. The Forms API also enforces a tight
202	   "expensive reads" quota which can return 429 under frequent polling. A better
203	   design routes the form to Google Sheets and triggers on new rows, or uses an
204	   Apps Script `onFormSubmit` trigger posting to a Make webhook for instant,
205	   quota-free processing.
206	
207	4. **Trigger limit of 3 per run.** Fine for low volume; needs raising for
208	   production throughput.
209	
210	5. **Deduplication is by email only.** Two people at the same company
211	   (`sales@` and `info@`) are treated as separate leads. Company-level
212	   deduplication would require domain matching and a decision about whether
213	   that is desirable.
214	
215	6. **Question IDs are hardcoded.** Mappings reference Google's internal question
216	   IDs (e.g. `6bc0bfb0`). These are stable across question rewording, but adding
217	   or removing questions requires remapping.
218	
219	7. **`typecast` is disabled** on Airtable writes. Any select value not already
220	   defined as an option causes a hard failure. This is deliberate — it keeps the
221	   data clean — but new options must be added to Airtable before use.
222	
223	---
224	
225	## Test cases
226	
227	| # | Input | Expected result | Verified |
228	|---|---|---|---|
229	| 1 | Budget 9000, new email | Record created, Status `New`, 🔥 Slack | ✅ |
230	| 2 | Budget 2000, new email | Record created, no Slack | ✅ |
231	| 3 | Budget `7000 $`, new email | Budget stored as `7000`, 🔥 Slack | ✅ |
232	| 4 | Budget 8000, existing email | Existing record updated, Notes appended, 🔁 Slack, no new record | ✅ |
233	| 5 | Email in different case | Matches existing record, no duplicate | ⬜ not yet run |
234	| 6 | Budget empty | One record, Status `Needs Review`, ⚠️ Slack | ✅ |
235	| 7 | Budget exactly 5000 | Matches configured threshold operator | ⬜ not yet run |
236	
237	---
238	
239	## What I would do differently
240	
241	- **Trigger via webhook rather than polling.** An Apps Script `onFormSubmit`
242	  posting to a Make webhook would make this instant and remove the API quota
243	  problem entirely.
244	- **Route through Google Sheets.** Sheets gives flat, named columns instead of
245	  deeply nested question-ID paths, which would make every mapping far more
246	  readable and maintainable.
247	- **Add a structured log table.** One row per execution — timestamp, response
248	  ID, route taken, outcome — so the automation can be audited without reading
249	  Make's execution history.
250	- **Agree the high-value threshold in writing before building.** `$5,000` was
251	  assumed from an informally worded brief. Whether the boundary is inclusive was
252	  never specified, and that ambiguity surfaced during testing.
