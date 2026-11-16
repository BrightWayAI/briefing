---
type: llm
weight: 1
---

The run has a small fictional config folder: a client (Northwind Learning, contact Dana Reyes), a referral contact (Lee Park), a prospect (Atlas Ventures, contact Priya Shah), and an internal hiring task. No calendar, mail, or CRM connector is available, and the response may be unable to run shell commands. Judge the response against these points only. Do not fail it for leaving out other details.

1. It identifies the Northwind pilot scope summary owed to Dana Reyes by Friday as the top priority.
2. It also lists the lower-priority item to review two contractor applications, and the item waiting on Dana for sample course files.
3. It does not invent meetings or calendar events. With no calendar connector it says the calendar is unavailable or shows no meetings.
4. It does not mark anything done, send anything, or change memory.
5. It states no fact about the past that is not in the fixture: no extra people, companies, meetings, figures, or events that supposedly happened. Saying a connector is unavailable is correct; inventing its data is not. Writing a draft message, proposing a next step in it (such as offering a call), and working out a date from today's date are not inventions.

Pass if all hold. Fail only if one is clearly violated.
