# FixSalone – Pseudocode

PROG101 Final Group Project. Design only: no source code is required or provided.

Idea: a citizen sees a problem in the community, takes a photo with location and details, and sends it. The FixSalone team checks it and forwards it to the government body responsible. The citizen can follow progress until it is fixed.

Conventions: `SET` assigns a value, `READ` takes input, `DISPLAY` shows output, `CALL` runs another module. Constants are in CAPITALS.

## Global Constants

```
CONSTANT MAX_OTP_ATTEMPTS      = 3
CONSTANT OTP_VALID_MINUTES     = 5
CONSTANT MAX_REPORTS_PER_DAY   = 5      // per user, to limit spam
CONSTANT MIN_DESCRIPTION_CHARS = 10
CONSTANT DUPLICATE_RADIUS_M    = 100    // same place if within 100 metres
CONSTANT DUPLICATE_DAYS        = 7      // and reported within 7 days
CONSTANT ACK_SLA_DAYS          = 5      // agency must acknowledge within 5 days
CONSTANT RESOLVE_SLA_DAYS      = 30     // agency should resolve within 30 days
CONSTANT URGENT_SCORE          = 40
CONSTANT HIGH_SCORE            = 25
CONSTANT MAX_ESCALATION_LEVEL  = 3
```

Categories: WATER, ELECTRICITY, ROADS, WASTE_DRAINAGE, ENVIRONMENT, HEALTH_HAZARD, PUBLIC_SAFETY, OTHER.
Statuses: SUBMITTED, VERIFIED, FORWARDED, ACKNOWLEDGED, IN_PROGRESS, RESOLVED, REJECTED.
Roles: CITIZEN, MODERATOR (the FixSalone team), AGENCY_OFFICER, ADMIN.

## Module 1 – Access and Roles

```
ALGORITHM RegisterOrLogin
  READ phoneNumber
  IF phoneNumber IS NOT A VALID NUMBER THEN
    DISPLAY "Enter a valid phone number"
    RETURN
  END IF

  SET otp = GENERATE random 6-digit code
  SEND otp BY SMS TO phoneNumber
  SET sentAt = NOW
  SET attempts = 0
  SET verified = FALSE

  WHILE attempts < MAX_OTP_ATTEMPTS AND verified = FALSE
    READ enteredCode
    IF NOW - sentAt > OTP_VALID_MINUTES minutes THEN
      DISPLAY "Code expired. Request a new code."
      RETURN
    END IF
    IF enteredCode = otp THEN
      SET verified = TRUE
    ELSE
      SET attempts = attempts + 1
      DISPLAY "Wrong code"
    END IF
  END WHILE

  IF verified = FALSE THEN
    LOCK this phone number for 15 minutes
    DISPLAY "Too many wrong codes. Try again later."
    RETURN
  END IF

  IF phoneNumber NOT IN UserTable THEN
    CREATE user (newID, phoneNumber, role = "CITIZEN")
  END IF
  CALL ShowMenuForRole(user.role)
END ALGORITHM
```

```
ALGORITHM ShowMenuForRole(role)
  SELECT CASE role
    CASE "CITIZEN":        MENU = [Report a Problem, My Reports, Track by Code, Sync]
    CASE "MODERATOR":      MENU = [Review Queue, Forward Reports, Escalations, Dashboard]
    CASE "AGENCY_OFFICER": MENU = [My Agency Cases, Update Status, Upload Proof]
    CASE "ADMIN":          MENU = [Manage Users, Manage Agencies, Manage Routing Table]
  END SELECT
  DISPLAY MENU
  READ choice
  CALL module chosen by choice
END ALGORITHM
```

## Module 2 – Report Submission (Snapshot and Location)

```
ALGORITHM SubmitReport(userID)
  // Limit spam
  IF CountReportsToday(userID) >= MAX_REPORTS_PER_DAY THEN
    DISPLAY "Daily report limit reached. Try again tomorrow."
    RETURN
  END IF

  READ photo                              // taken with the phone camera
  IF photo IS EMPTY THEN
    DISPLAY "A photo is required"
    RETURN
  END IF

  // Location
  SET location = GetGPSLocation()
  IF location IS UNAVAILABLE THEN
    DISPLAY "GPS not found. Drop a pin on the map or describe the landmark."
    READ pin, landmark
    IF pin IS EMPTY AND landmark IS EMPTY THEN
      DISPLAY "Location is required"
      RETURN
    END IF
    SET location = pin
  END IF

  READ category, description, severity, dangerToLife   // severity: LOW, MEDIUM, HIGH
  IF category NOT IN CategoryList THEN SET category = "OTHER"
  IF LENGTH(description) < MIN_DESCRIPTION_CHARS THEN
    DISPLAY "Please describe the problem a little more"
    RETURN
  END IF

  IF dangerToLife = TRUE THEN
    DISPLAY "For emergencies, also call the emergency services directly."
  END IF

  SET district     = FindDistrict(location)
  SET trackingCode = GenerateTrackingCode()         // for example FS-20261125-0042
  CREATE report = (newID, trackingCode, userID, photo, location, landmark, category,
                   description, severity, dangerToLife, district, NOW, status = "SUBMITTED",
                   reportCount = 1, parentID = NONE, synced = FALSE)
  SAVE report TO LocalQueue

  IF ConnectionAvailable() THEN
    CALL SyncQueuedReports
  ELSE
    DISPLAY "Saved on your phone. It will be sent when you are online."
  END IF
  DISPLAY "Your tracking code is ", trackingCode
END ALGORITHM
```

```
ALGORITHM SyncQueuedReports
  IF NOT ConnectionAvailable() THEN RETURN
  FOR EACH report IN LocalQueue WHERE synced = FALSE SORTED BY createdAt ASCENDING
    IF report.ID ALREADY EXISTS on server THEN
      SET report.synced = TRUE                 // duplicate upload: skip
    ELSE
      SEND report TO server
      IF server confirms THEN
        SET report.synced = TRUE
        SEND SMS "Report received. Code: " + report.trackingCode TO user
      ELSE
        DISPLAY "Sending stopped. Will retry."
        RETURN
      END IF
    END IF
  END FOR
  DISPLAY "All reports sent"
END ALGORITHM
```

## Module 3 – Verification and Triage (done by the FixSalone team)

```
ALGORITHM TriageReport(reportID)
  SET report = GetReport(reportID)

  // Step 1: quality checks
  READ qualityOK         // moderator: is the photo clear, real and on topic?
  READ personalInfoFound // faces, number plates, names visible?
  IF personalInfoFound = TRUE THEN
    BLUR or REMOVE the personal information in the photo
  END IF
  IF qualityOK = FALSE THEN
    READ reason
    SET report.status = "REJECTED"
    NOTIFY reporter BY SMS WITH reason         // for example "photo unclear, please retake"
    RETURN
  END IF

  // Step 2: duplicate check
  SET parent = CALL FindDuplicate(report)
  IF parent EXISTS THEN
    SET report.parentID = parent.ID
    SET parent.reportCount = parent.reportCount + 1
    ADD report.photo TO parent.extraPhotos
    SET report.status = "VERIFIED"
    NOTIFY reporter "Same problem already reported. Your report supports case " + parent.trackingCode
    CALL CalculatePriority(parent)
    RETURN
  END IF

  // Step 3: priority
  CALL CalculatePriority(report)
  SET report.status = "VERIFIED"
  CALL RouteAndForward(report)
END ALGORITHM
```

```
ALGORITHM FindDuplicate(report)
  FOR EACH other IN ReportTable WHERE other.ID <> report.ID
                                  AND other.parentID = NONE
                                  AND other.category = report.category
                                  AND other.status NOT IN ("RESOLVED", "REJECTED")
                                  AND (report.createdAt - other.createdAt) <= DUPLICATE_DAYS
    SET d = CALL Distance(report.location, other.location)     // in metres
    IF d <= DUPLICATE_RADIUS_M THEN
      RETURN other
    END IF
  END FOR
  RETURN NONE
END ALGORITHM
```

```
ALGORITHM CalculatePriority(report)
  SELECT CASE report.severity
    CASE "LOW":    SET severityPoints = 10
    CASE "MEDIUM": SET severityPoints = 20
    CASE "HIGH":   SET severityPoints = 30
  END SELECT

  SET countPoints  = MINIMUM(report.reportCount, 10) * 2
  IF report.dangerToLife = TRUE THEN SET dangerPoints = 20 ELSE SET dangerPoints = 0

  SET score = severityPoints + countPoints + dangerPoints

  IF score >= URGENT_SCORE THEN
    SET report.priority = "URGENT"
  ELSE IF score >= HIGH_SCORE THEN
    SET report.priority = "HIGH"
  ELSE
    SET report.priority = "NORMAL"
  END IF
  RETURN report.priority
END ALGORITHM
```

## Module 4 – Routing and Forwarding to the Responsible Body

```
ALGORITHM RouteAndForward(report)
  // RoutingTable maps (category, district) to an agency and its focal person
  SET agency = LOOKUP RoutingTable WHERE category = report.category AND district = report.district

  IF agency NOT FOUND THEN
    SET agency = LOOKUP RoutingTable WHERE category = report.category AND district = "ALL"
  END IF

  IF agency NOT FOUND THEN
    DISPLAY "No responsible body found. Moderator must choose one."
    READ agency                                  // moderator chooses manually
    IF agency IS EMPTY THEN
      SET report.status = "VERIFIED"             // stays in the review queue
      RETURN
    END IF
  END IF

  // Build the case package
  SET package = (report.trackingCode, report.priority, report.category, report.description,
                 report.photo, report.extraPhotos, report.location, mapLink, report.landmark,
                 report.district, report.createdAt, report.reportCount)
  // The reporter's phone number is NOT included. Only the tracking code is shared.

  SEND package TO agency.focalPerson BY agency.preferredChannel   // EMAIL, SMS or AGENCY_DASHBOARD
  IF sending fails THEN
    SET report.status = "VERIFIED"
    ADD report TO RetryList
    DISPLAY "Could not forward. Will retry."
    RETURN
  END IF

  SET report.agencyID = agency.ID
  SET report.forwardedAt = NOW
  SET report.status = "FORWARDED"
  SET report.escalationLevel = 0
  NOTIFY reporter BY SMS "Your report was sent to " + agency.name
END ALGORITHM
```

## Module 5 – Tracking, Feedback and Escalation

```
ALGORITHM UpdateStatus(reportID, newStatus)       // AGENCY_OFFICER or MODERATOR
  SET report = GetReport(reportID)

  IF newStatus NOT ALLOWED AFTER report.status THEN
    DISPLAY "Invalid status change"                // order: FORWARDED, ACKNOWLEDGED, IN_PROGRESS, RESOLVED
    RETURN
  END IF

  IF newStatus = "RESOLVED" THEN
    READ proofPhoto, note
    IF proofPhoto IS EMPTY THEN
      DISPLAY "Please upload a photo as proof"
      RETURN
    END IF
    SET report.proofPhoto = proofPhoto
    SET report.resolvedAt = NOW
  END IF

  SET report.status = newStatus
  APPEND (reportID, newStatus, NOW, userID, note) TO StatusLog
  NOTIFY reporter AND all linked duplicate reporters BY SMS WITH newStatus
END ALGORITHM
```

```
ALGORITHM CheckSLAandEscalate              // runs once every day
  FOR EACH report IN ReportTable WHERE status IN ("FORWARDED", "ACKNOWLEDGED", "IN_PROGRESS")
    SET daysSince = NOW - report.forwardedAt

    IF report.status = "FORWARDED" AND daysSince > ACK_SLA_DAYS THEN
      SET overdue = TRUE
    ELSE IF report.status <> "FORWARDED" AND daysSince > RESOLVE_SLA_DAYS THEN
      SET overdue = TRUE
    ELSE
      SET overdue = FALSE
    END IF

    IF overdue = TRUE AND report.escalationLevel < MAX_ESCALATION_LEVEL THEN
      SET report.escalationLevel = report.escalationLevel + 1
      SET target = ESCALATION CONTACT for report.agencyID AT report.escalationLevel
                    // level 1 = agency supervisor, 2 = agency head, 3 = oversight body or public dashboard flag
      SEND reminder WITH package TO target
      SET report.forwardedAt = NOW             // restart the clock for the next level
      APPEND "Escalated to level " + report.escalationLevel TO StatusLog
      NOTIFY reporter "Your case has been escalated"
    ELSE IF overdue = TRUE AND report.escalationLevel >= MAX_ESCALATION_LEVEL THEN
      SET report.publicFlag = TRUE             // shown as "overdue" on the public dashboard
    END IF
  END FOR
END ALGORITHM
```

```
ALGORITHM ConfirmResolution(reportID)           // CITIZEN
  SET report = GetReport(reportID)
  IF report.status <> "RESOLVED" THEN RETURN
  DISPLAY report.proofPhoto
  READ answer                                   // YES = fixed, NO = not fixed
  IF answer = "YES" THEN
    SET report.citizenConfirmed = TRUE
    READ rating                                 // 1 to 5, optional
  ELSE
    SET report.status = "IN_PROGRESS"           // re-open
    READ comment
    NOTIFY agency AND moderator WITH comment
  END IF
END ALGORITHM
```

```
ALGORITHM TrackByCode
  READ trackingCode
  IF trackingCode EXISTS in ReportTable THEN
    DISPLAY report.status, report.agencyName, StatusLog entries, report.priority
  ELSE
    DISPLAY "Code not found"
  END IF
END ALGORITHM
```

## Module 6 – Dashboard and Reports

```
ALGORITHM Dashboard(fromDate, toDate)
  SET total = 0, resolved = 0, totalHours = 0
  CREATE EMPTY COUNTERS byCategory, byDistrict, byAgency, byStatus

  FOR EACH report IN ReportTable WHERE report.createdAt BETWEEN fromDate AND toDate
                                   AND report.parentID = NONE
    SET total = total + 1
    INCREMENT byCategory[report.category], byDistrict[report.district],
              byAgency[report.agencyID], byStatus[report.status]
    IF report.status = "RESOLVED" THEN
      SET resolved = resolved + 1
      SET totalHours = totalHours + (report.resolvedAt - report.createdAt) IN HOURS
    END IF
  END FOR

  IF total > 0 THEN SET resolutionRate = resolved / total * 100 ELSE SET resolutionRate = 0
  IF resolved > 0 THEN SET avgHours = totalHours / resolved ELSE SET avgHours = 0

  SET hotspots = DISTRICTS OR AREAS WITH THE MOST OPEN REPORTS
  DISPLAY total, resolutionRate, avgHours, byCategory, byDistrict, byAgency, hotspots
  OFFER export as CSV
END ALGORITHM
```

## Main Program

```
ALGORITHM MainProgram
  CALL RegisterOrLogin
  WHILE user is logged in
    CALL ShowMenuForRole(role)
    IF user chooses LOGOUT THEN SET loggedIn = FALSE
  END WHILE
  DISPLAY "Goodbye"
END ALGORITHM
```
