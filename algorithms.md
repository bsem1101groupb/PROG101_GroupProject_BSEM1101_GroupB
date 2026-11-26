# FixSalone: Algorithms (plain language)

Each algorithm is written as numbered steps in everyday language. The same logic is written in pseudocode in `pseudocode.md` and drawn as flowcharts in `flowcharts/`.

Starting values (can be changed without changing the algorithms): 3 code attempts, code valid 5 minutes, 5 reports per day, duplicate radius 100 m, duplicate window 7 days, acknowledge in 5 days, resolve in 30 days, 3 escalation levels.

---

## 1. Register or Login (Module 1)

**Goal:** make sure the user is a real person with a working phone number.

1. Ask for the phone number. If it is not valid, stop and ask again.
2. Send a 6-digit code by SMS. Set attempts to 0.
3. While attempts are less than 3 and the code is not yet correct:
   1. Ask for the code.
   2. If more than 5 minutes have passed, say the code expired and stop.
   3. If the code is correct, the user is verified. Otherwise add 1 to attempts and say "Wrong code".
4. If not verified, lock the number for 15 minutes and stop.
5. If the number is new, create a Citizen account.
6. Show the menu for the user's role.

## 2. Submit a Report (Module 2)

**Goal:** capture a complete report with a photo and a location.

1. If the user already sent 5 reports today, stop.
2. Ask for a photo. If there is none, stop.
3. Get the GPS location. If GPS is not available, ask the user to drop a pin or describe a landmark. If neither is given, stop.
4. Ask for the category, description, severity and whether there is danger to life.
5. If the description is shorter than 10 characters, ask for more detail and stop.
6. If there is danger to life, show: "Also call the emergency services directly."
7. Find the district, create a tracking code and save the report on the phone.
8. If there is a connection, send the report now (algorithm 3). Otherwise say it will be sent later.
9. Show the tracking code.

## 3. Send Queued Reports (Module 2, offline support)

**Goal:** never lose a report, and never send it twice.

1. If there is no connection, stop.
2. Take the unsent reports, oldest first.
3. For each report:
   1. If the server already has this report ID, mark it as sent.
   2. Otherwise send it. If the server confirms, mark it as sent and SMS the tracking code to the citizen.
   3. If sending fails, stop and try again later.
4. Say "All reports sent".

## 4. Triage a Report (Module 3)

**Goal:** make sure the report is clear, private, not a repeat, and ranked.

1. If the photo shows faces or number plates, blur or remove them.
2. If the photo is unclear, fake or off topic, reject the report, SMS the reason to the citizen and stop.
3. Look for a duplicate (algorithm 5).
4. If there is a duplicate, link the new report to the first one, add 1 to its report count, add the photo, tell the citizen, recalculate the priority of the first report and stop.
5. Otherwise calculate the priority (algorithm 6), mark the report verified and send it to be routed (algorithm 7).

## 5. Find a Duplicate (Module 3)

**Goal:** treat many reports of the same problem as one case.

1. Look at open reports that are not themselves duplicates.
2. Keep only reports with the same category, created within the last 7 days.
3. Measure the distance to each of them.
4. If one is within 100 metres, it is the duplicate. Return it.
5. If none are, there is no duplicate.

## 6. Calculate Priority (Module 3)

**Goal:** decide which problems need attention first.

1. Severity points: LOW = 10, MEDIUM = 20, HIGH = 30.
2. Count points: 2 for each report on the case, up to 10 reports.
3. Danger points: 20 if there is danger to life, otherwise 0.
4. Add them to get the score.
5. Score 40 or more is URGENT. Score 25 to 39 is HIGH. Anything lower is NORMAL.

## 7. Route and Forward (Module 4)

**Goal:** get the case to the body responsible.

1. Look up the agency in the routing table by category and district.
2. If none is found, look up by category for all districts.
3. If still none, the moderator chooses an agency. If none is chosen, keep the report in the review queue and stop.
4. Build the case package: tracking code, priority, category, description, photos, location, map link, report count. **Do not include the reporter's phone number.**
5. Send it to the agency's focal person by email, SMS or the agency dashboard.
6. If sending fails, add the report to a retry list and stop.
7. Mark the report FORWARDED, record the time, set escalation level to 0 and SMS the citizen.

## 8. Update Status (Module 5)

**Goal:** keep a true record of progress.

1. Check that the new status is allowed after the current one (FORWARDED, ACKNOWLEDGED, IN_PROGRESS, RESOLVED). If not, refuse.
2. If the new status is RESOLVED, require a proof photo and record the time. Without a photo, refuse.
3. Save the status and write it to the status log.
4. SMS the citizen and everyone linked to the case.

## 9. Check Deadlines and Escalate (Module 5, runs daily)

**Goal:** make sure agencies act.

1. For each open case (FORWARDED, ACKNOWLEDGED or IN_PROGRESS), count the days since it was forwarded.
2. If it is still FORWARDED after more than 5 days, or in progress after more than 30 days, it is overdue.
3. If overdue and the escalation level is below 3: add 1 to the level, send a reminder to the contact for that level (1 = supervisor, 2 = agency head, 3 = oversight contact), restart the clock, write to the log and tell the citizen.
4. If overdue and the level is already 3, flag the case as overdue on the public dashboard.

## 10. Confirm Resolution (Module 5)

1. Show the proof photo to the citizen and ask "Is it fixed?"
2. If yes, mark it confirmed and ask for an optional rating from 1 to 5.
3. If no, set the status back to IN_PROGRESS, ask for a comment and notify the agency and moderator.

## 11. Dashboard (Module 6)

1. Set the total and resolved counts to 0.
2. For each case in the chosen dates (not counting duplicates), add to the counts by category, district, agency and status. If resolved, add its time to resolve to the total hours.
3. Resolution rate = resolved divided by total, times 100 (0 if there are no cases).
4. Average time to resolve = total hours divided by resolved (0 if none).
5. Show the counts, the rates and the areas with the most open cases. Offer a CSV export.
