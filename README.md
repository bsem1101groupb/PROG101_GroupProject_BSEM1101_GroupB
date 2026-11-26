# PROG101_GroupProject_BSEM1101_Group

# FixSalone

See it. Snap it. Send it. Fix it.
An open source community problem reporting system that connects citizens to the government bodies responsible for fixing local problems in Sierra Leone.

PROG101: Principles of Pragramming Logic and Design, Limkokwing University of Creative Technology, Sierra Leone
Semester 1 Group Project, Group: Group B Class: BSEM1101 | Examiner: Elijah Fullah

This is a software design project. It contains analysis, algorithms, pseudocode, flowcharts and documentation. It does not contain working code, as the project brief requires.

## The Problem

People in Sierra Leone see problems in their communities every day: burst pipes, blocked drains, uncollected waste, potholes. They stay unfixed because the responsible body does not get clear information about what is wrong and where, citizens do not know whom to contact, and nobody can follow up or see delays.

## The Idea

1. A citizen **takes a photo** of the problem. The phone adds the **location**, and the citizen adds the category and a short description.
2. The report is sent to us (the FixSalone team). It is saved on the phone first, so it also works offline.
3. We verify it: blur faces and number plates, reject unclear photos, merge repeat reports, and set a priority.
4. We forward it to the government body responsible, using a routing table by category and district.
5. The agency fixes it and uploads a proof photo. The citizen gets SMS updates and confirms the fix.
6. If the agency is late, the case is escalated and finally flagged on a public dashboard.

The reporter's phone number is never shared with the agency, only a tracking code.

## Modules
Module  Responsibility

 M1 Access and Roles:  Account creation & login; Citizen, Moderator, Agency Officer, Admin 
 M2 Report Submission:  Photo, location, details, tracking code, offline queue 
 M3 Verification and Triage: Quality check, privacy blur, duplicates, priority score
 M4 Routing and Forwarding:  Find the responsible agency and send the case
 M5 Tracking and Feedback:  Status updates, SMS, deadline check, escalation
 M6 Dashboard:  Results by category, district and agency 

## Key Rules (configurable)

- **Duplicate:** same category, within 100 m, within 7 days.
- **Priority score:** severity (10, 20 or 30) + 2 per report (max 10) + 20 if danger to life. URGENT at 40 or more, HIGH at 25 or more.
- **Deadlines:** agency acknowledges within 5 days and resolves within 30 days; up to 3 escalation levels.

## Repository Contents


README.md           This file
pseudocode.md       Pseudocode for all modules
flowcharts/         7 flowcharts (PNG) and Graphviz sources
docs/               Final report, presentation, meeting minutes

## Open Source, DPG and SDGs

- **Licence:** MIT License
- **Digital Public Goods:** designed to meet the nine indicators of the DPG Standard.
- **SDGs:** mainly SDG 11 (Sustainable Cities and Communities) and SDG 16 (Strong Institutions), supporting SDG 6 and SDG 17.

## Scope

FixSalone collects, checks, forwards and tracks reports. It does not fix problems itself, is **not an emergency service**, and does not handle complaints about individuals.


## Team

   Name                           Student ID  
James John Jeremiah Marke         905006303
Mariama Barrie.                   905006320
Oladie Cynthia Coker              905006027
Chernor Osman Iyne                905006286

This repository is the group's own work and is not shared with other groups.
