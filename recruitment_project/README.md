The Recruitment Observations Pipeline captures unstructured signals from recruiters' daily work (hiring manager conversations, market intel, candidate feedback,
sourcing discoveries) and turns them into structured, queryable, and surfaced data. Before this system, these observations lived in people's heads or
scattered across chat threads. Now they flow from a Google Sheet into a Supabase database and back out to a dashboard — all automated via n8n.

Two workflows, one database (Ingestion.json), one dashboard (Retrieval.json).

Ingestion Pipeline

Google Sheet (Observations tab)
  → n8n picks up new rows via Google Sheets Trigger
  → Validates required fields and enum values
  → Maps human-readable names (Ana Reyes, Acme Corp - Technology Transformation)
    to UUIDs for database integrity
  → Inserts into Supabase `observations` table
  → Writes status back to the sheet ("inserted" or "error: ...")
  
Dashboard Pipeline

n8n Schedule Trigger (every 2 minutes)
  → Queries Supabase views (dashboard_action_needed, dashboard_recent)
  → Formats into two sections: Action Needed (pinned top) + Recent Activity
  → Deduplicates so action items don't appear twice
  → Clears and repopulates the Dashboard Google Sheet tab
  
Database (Supabase)

recruiters — who's on the team
work_packets — client hiring efforts
positions — roles within work packets
observations — the core table storing every logged note with tags, urgency, visibility, and references to the above

Key Design Decisions
1. Google Sheets as the input surface (not a custom UI).
Rationale: Recruiters already live in Sheets and fast production. Zero adoption friction. Dropdowns enforce structure without requiring a new tool.
A Lovable UI is planned for later but the database and workflows are ready for it.

3. UUIDs over auto-increment IDs.
Rationale: Distributed writes (multiple recruiters, potentially offline sheet rows) mean ID collisions are a real risk. UUIDs eliminate that.

4. wp_id and position_code are nullable.
Rationale: Market signals (e.g., "Shopify layoffs") often aren't tied to a specific work packet. Forcing a WP would discourage logging those observations.

5. visibility field: internal_only vs client_shareable.
Rationale: Some observations are sensitive (candidate feedback about a hiring manager's style, internal capacity notes).
This flag prevents accidents when sharing data with clients.

7. Tags as pipe-separated strings in Sheets, arrays in Supabase.
Rationale: Pipe separation is easy to type. n8n splits into arrays on insert. This gives us freeform tagging with queryability.

8. Dashboard as a Google Sheet tab, not a custom UI.
Rationale: Fastest path to value. Recruiters check one place. The "clear and repopulate every 2 minutes" approach keeps it simple — no row-level sync logic.

Handoff Notes for Another Developer

To pick this up, you need access to:
Google Cloud project with Sheets API and Drive API enabled
Supabase project
n8n instance with both Google and Supabase credentials connected
Two Google Sheets:
  Recruitment — ingestion
  Dashboard — output

To extend:
  - Adding fields to observations: Add column to Supabase, add column to Google Sheet Observations tab,
    update both n8n workflows (validation code + Supabase insert mapping + dashboard format code).
  - Adding a new dropdown value (e.g., new signal_source): Add it to the Validation Values tab in the sheet,
    add it to the validation arrays in the ingestion Code node, no database changes needed.
  - Switching to a Lovable UI: The Supabase schema stays the same. Build the frontend against Supabase directly.
    The n8n ingestion workflow can be retired once the UI handles validation and insert. Keep the dashboard workflow as-is or replace it with a Supabase query in the UI.
  - Adding Slack/Teams alerts: Add an n8n node after Supabase insert on the ingestion workflow.
    Filter for urgency = action_needed. Use the Slack/Teams node to notify a channel.

To maintain:
  - Google Sheets Trigger can miss rows if the sheet is edited rapidly. The trigger polls for new rows — if someone inserts multiple rows between polls,
    all are captured in the next batch. The deduplication logic in the dashboard handles this. The ingestion workflow processes whatever the trigger sends.
  - The dashboard "clear and repopulate" approach means the sheet blinks empty briefly every 2 minutes. This is acceptable for a small team. If it becomes annoying,
    switch to a differential update approach.
  - Supabase views (dashboard_action_needed, dashboard_recent) have a LIMIT 50 in the combined view and LIMIT 20 in recent.
    Adjust these if the observation volume grows.

Assumptions

Recruiters will use the dropdowns. Data validation is set up in the sheet, but Google Sheets can't enforce it perfectly.
If someone types a freeform value where a dropdown is expected, the validation code in n8n will catch it and write an error back to the status column.
They'll see "error: Invalid signal_source: blah" and fix it.

One observation per row. The system doesn't support bulk/batch logging yet. If a recruiter has 5 observations after a sourcing session, they enter 5 rows.
The batch capture mode was designed but not built.

Recruiter names are static. The UUID mapping for Ana, David, and Sarah is hardcoded in the n8n validation code.
Adding a new recruiter requires: adding them to the recruiters table in Supabase, adding them to the Reference tab in the sheet (for the dropdown),
and adding their name → UUID mapping to the n8n code node. A future improvement would be to read the Reference tab dynamically instead of hardcoding.

created_at comes from the sheet, not the database. If a recruiter logs an observation they had last Tuesday, they can backdate it by entering the timestamp.
If left blank, n8n sets it to now(). This is intentional — observations aren't always logged in real time.

Dashboard refreshes every 2 minutes. Acceptable latency for a recruiting dashboard. If real-time is needed, switch to Supabase Realtime subscriptions in the Lovable UI.

The system trusts the Google Sheet as the source of truth for observations until they reach Supabase.
If someone deletes a row from the sheet after it's been inserted, Supabase still has it. No deletion sync exists.
That's by design — we don't want accidental sheet deletions to destroy data.
