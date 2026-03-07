# TOOLS.md - Local Notes

This file contains environment-specific instructions for tools available to the Executive Assistant.

Only include information that is specific to this environment.

---

# Google Calendar (Source of Truth)

All scheduling information comes from **Google Calendar**.

The system accesses Google Calendar using the **`gog` CLI**.

## Rules

- ALWAYS use the `gog` CLI for calendar queries
- Execute commands using the **shell tool**
- NEVER use `openclaw calendar`
- NEVER read `/workspace/calendars/*`
- NEVER rely on cached calendar files
- Google Calendar is the single source of truth

Default account:

taylortigertravels@gmail.com

---

# Calendar Query Modes

The assistant supports two calendar query modes: **Short Calendar** and **Full Calendar**.

---

# Short Calendar (Default)

Use **Short Calendar** unless the user explicitly asks for a full calendar.

Short Calendar includes only the main scheduling calendars:

- taylortigertravels@gmail.com (Primary)
- lvlivingston@gmail.com
- bm5qte2k9nthj2oh1319l90ob8@group.calendar.google.com (Work Schedule)
- f6i777aua4kfrr52hlajmi443s@group.calendar.google.com (Fun Stuff)
- 987b998df9a858cad963419a99df21de4a0e551250586dfd6d840dc66055703e@group.calendar.google.com (Don't Forget)

Combine the results from these calendars into **one chronological summary**.

Short Calendar is used for questions like:

- What is on my calendar today?
- What do I have Monday?
- What meetings do I have?
- What's my schedule?

Short Calendar **does NOT include**:

- sendmeyogastuff@gmail.com
- agnihaus@gmail.com
- birthdays
- health calendars
- holiday calendars
- sports calendars
- moon phase calendars

---

# Full Calendar

Use **Full Calendar** only when the user explicitly asks for it.

Examples:

- Show my full calendar
- Includes sendmeyogastuff@gmail.com
- Includes agnihaus@gmail.com
- Include birthdays
- Include holidays
- Everything on March 9
- Show all events

Full Calendar includes **all calendars**, including:

- taylortigertravels@gmail.com
- lvlivingston@gmail.com
- bm5qte2k9nthj2oh1319l90ob8@group.calendar.google.com (Work Schedule)
- f6i777aua4kfrr52hlajmi443s@group.calendar.google.com (Fun Stuff)
- 987b998df9a858cad963419a99df21de4a0e551250586dfd6d840dc66055703e@group.calendar.google.com (Don't Forget)
- i54mobrkuoc3f7urkmt8ionp8c@group.calendar.google.com (Birthday)
- 0g49j61et3hfb89cap0mqfrnbc@group.calendar.google.com (Health)
- sendmeyogastuff@gmail.com
- agnihaus@gmail.com

It may also include:

- holiday calendars
- sports calendars
- moon phase calendars

---

# Calendar Command Pattern

To retrieve events from a calendar:
`gog calendar events CALENDAR_ID --from "YYYY-MM-DD 00:00" --to "YYYY-MM-DD 23:59" --plain --no-input`

Example:
`gog calendar events bxxxxxxxx@group.calendar.google.com --from "2026-03-09 00:00" --to "2026-03-09 23:59" --plain --no-input`

---

# Expected Behavior

When the user asks about their calendar or schedule, determine the date range requested\*\*.

Examples of user requests:

- "Give me short calendar."
- "What's on the calendar today?"
- "What is on my calendar tomorrow?"
- "What do I have (day)?"
- "What is my schedule for (month) (date)?"
- "What meetings do I have today?"
- "Show my calendar this week."

The assistant should:

1. Determine the correct date or date range.
2. Use Short Calendar unless the user explicitly asks for the full calendar.
3. Query each calendar using the gog calendar events command.
4. Merge all results from the selected calendars.
5. Sort events chronologically.
6. Return a clear schedule summary.

Example command pattern:
`gog calendar events CALENDAR_ID --from "YYYY-MM-DD 00:00" --to "YYYY-MM-DD 23:59" --plain --no-input`

Example behavior:

User asks:
`What's tomorrow's schedule?`

Assistant should:

- determine tomorrow’s date
- query the Short Calendar list
- merge the events
- return a summarized schedule.

If the user explicitly asks for Full Calendar, then include all calendars.

---

# Useful gog Commands

List available calendars:
`gog calendar calendars --plain --no-input`

Get events for a specific calendar and day:
`gog calendar events CALENDAR_ID --from "YYYY-MM-DD 00:00" --to "YYYY-MM-DD 23:59" --plain --no-input`

Example:
`gog calendar events lvlivinston@gmail.com --from "2026-03-09 00:00" --to "2026-03-09 23:59" --plain --no-input`

Check availability:
`gog calendar freebusy primary --from "YYYY-MM-DD HH:MM" --to "YYYY-MM-DD HH:MM" --no-input`
