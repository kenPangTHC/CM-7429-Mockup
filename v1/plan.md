# CARES Scheduling App — Development Plan

## Context

**Codebase:** `C:\Users\USER\Documents\GitHub\CARES-HOME`
**Stack:** ASP.NET 4.8 Web Forms + VB.NET + SQL Server (Dapper + EF6) + jQuery
**UI Mockups:** `C:\Users\USER\Documents\MARKDOWN_notes\CM-7429\`
**Standards:** Follow `.github/copilot-instructions.md` + `.github/STANDARDS.md`

---

## Problem Statement

Build a full therapy scheduling module into CARES that covers:
- Monthly therapist shift setup (Tier 1/2/3 template hierarchy → commit → slot generation)
- Appointment booking against open slots
- Recurring appointment series
- Stage 1 auto-fill (match recurring series to open slots after commit)
- Stage 2 manual fill (coordinator assigns unfilled appointments)
- Therapist leave + conflict detection + resolution (reassign)
- Completion marking per session + billing linkage to PatientAICServices

---

## Key Design Decisions

### 1. Polymorphic Reference Pattern — "Recipient" (Therapist / Machine)

Appointments are booked *to* a resource (therapist or machine). Rather than hard FK columns per table type, use:

| Column | Type | Example Values |
|---|---|---|
| `RecipientType` | `nvarchar(50) NOT NULL` | `'Users'`, `'Equipment'` |
| `RecipientID` | `int NOT NULL` | UserId, EquipmentID |

**No enforced FK constraint.** Application layer resolves the referenced table by `RecipientType`.

### 2. Polymorphic Reference Pattern — "Participant" (Patient side)

Same pattern for the patient/subject side of the appointment:

| Column | Type | Example Values |
|---|---|---|
| `ParticipantType` | `nvarchar(50) NOT NULL` | `'Patient'` |
| `ParticipantID` | `int NOT NULL` | PatientID |

Extensible for future types (e.g., group sessions, guest attendees).

### 3. Recurrence Columns — Reuse Existing Pattern

Reuse the same columns already in `PatientMedicationConsume` and `Task`:

| Column | Type | Notes |
|---|---|---|
| `Recurrence_Frequency` | `int NULL` | 1=Never, 2=Daily, 3=Weekly, 4=Monthly, 5=SpecificDates |
| `Recurrence_Days` | `int NULL` | Bitmask: Sun=1, Mon=2, Tue=4, Wed=8, Thu=16, Fri=32, Sat=64 |
| `Recurrence_Interval` | `int NULL` | Nth occurrence for monthly (1=first…5=last) |
| `Recurrence_IntervalFlag` | `int NULL` | 0=day-of-month, >0=Nth day-of-week pattern |
| `recurrenceDailyHourly` | `int NULL` | Daily/hourly sub-type |
| `Every_Days` | `int NULL` | Repeat every N days (Daily with interval) |

### 4. PatientAICServices Reference

Not a recurring design target (already exists). Referenced via nullable FK in:
- `TherapyAppointment.PatientAICServicesID_FK` (links session to AIC episode for billing)
- `TherapyAppointmentRecurrenceSeries.PatientAICServicesID_FK` (default AIC service for series)

### 5. Multi-Tenancy

All tables include `FacilityID_FK INT NOT NULL`.

### 6. Audit

Every new table has a mirror `Audit_TherapyXxx` table per `.github/copilot-instructions.md`.

---

## Database Schema

### Tier Hierarchy for Slot Generation

```
Tier 1 — TherapyShiftOverride       (Date-specific range override, highest priority)
  ↓ fallback
Tier 2 — TherapyShiftDefaultTemplate (Therapist default weekly schedule)
  ↓ fallback
Tier 3 — TherapyShiftGlobalConfig   (Facility-wide defaults, lowest priority)
```

---

### Table: `TherapyShiftGlobalConfig`

Per-facility global defaults. One active row per facility.

```sql
CREATE TABLE [dbo].[TherapyShiftGlobalConfig] (
    [TherapyShiftGlobalConfigID]  INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]               INT NOT NULL,
    [DefaultSessionDurationMins]  INT NOT NULL DEFAULT 30,
    -- Per-day AM/PM time windows (NULL = day off)
    [Mon_AM_Start]  TIME NULL,  [Mon_AM_End]  TIME NULL,
    [Mon_PM_Start]  TIME NULL,  [Mon_PM_End]  TIME NULL,
    -- ... Tue through Sun same pattern
    [IsDeleted]     BIT NOT NULL DEFAULT 0,
    [CreatedDate]   DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]  INT NOT NULL,
    [ModifiedDate]  DATETIME NULL,
    [ModifiedBy_FK] INT NULL
)
```

---

### Table: `TherapyShiftDefaultTemplate`

Tier 2 — per-therapist default weekly schedule with optional effective date range.

```sql
CREATE TABLE [dbo].[TherapyShiftDefaultTemplate] (
    [TherapyShiftDefaultTemplateID] INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]                 INT NOT NULL,
    [RecipientType]                 NVARCHAR(50) NOT NULL,
    [RecipientID]                   INT NOT NULL,
    [EffectiveStartDate]            DATE NULL,
    [EffectiveEndDate]              DATE NULL,
    -- Per-day AM/PM windows (Mon through Sun)
    [Mon_AM_Start]  TIME NULL,  [Mon_AM_End]  TIME NULL,
    [Mon_PM_Start]  TIME NULL,  [Mon_PM_End]  TIME NULL,
    -- ... Tue through Sun same pattern
    [SessionDurationMins]           INT NULL,   -- NULL = use global default
    [IsDeleted]     BIT NOT NULL DEFAULT 0,
    [CreatedDate]   DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]  INT NOT NULL,
    [ModifiedDate]  DATETIME NULL,
    [ModifiedBy_FK] INT NULL
)
```

---

### Tables: `TherapyShiftOverride` + `TherapyShiftOverrideDetail`

Tier 1 — named date-range override with per-day detail rows.

```sql
CREATE TABLE [dbo].[TherapyShiftOverride] (
    [TherapyShiftOverrideID] INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]          INT NOT NULL,
    [RecipientType]          NVARCHAR(50) NOT NULL,
    [RecipientID]            INT NOT NULL,
    [OverrideName]           NVARCHAR(100) NOT NULL,
    [StartDate]              DATE NOT NULL,
    [EndDate]                DATE NOT NULL,
    [IsDeleted]              BIT NOT NULL DEFAULT 0,
    [CreatedDate]            DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]           INT NOT NULL,
    [ModifiedDate]           DATETIME NULL,
    [ModifiedBy_FK]          INT NULL
)

CREATE TABLE [dbo].[TherapyShiftOverrideDetail] (
    [TherapyShiftOverrideDetailID] INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [TherapyShiftOverrideID_FK]    INT NOT NULL,
    [DayOfWeek]                    TINYINT NOT NULL,   -- 1=Mon…7=Sun
    [Period]                       CHAR(2) NOT NULL,   -- 'AM' or 'PM'
    [StartTime]                    TIME NOT NULL,
    [EndTime]                      TIME NOT NULL,
    [SessionDurationMins]          INT NULL,
    [IsActive]                     BIT NOT NULL DEFAULT 1,
    [IsDeleted]                    BIT NOT NULL DEFAULT 0,
    [CreatedDate]                  DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]                 INT NOT NULL,
    [ModifiedDate]                 DATETIME NULL,
    [ModifiedBy_FK]                INT NULL
)
```

---

### Table: `TherapyShiftMonthlyCommit`

Records the commit event for a facility + month.

```sql
CREATE TABLE [dbo].[TherapyShiftMonthlyCommit] (
    [TherapyShiftMonthlyCommitID] INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]               INT NOT NULL,
    [CommitYear]                  SMALLINT NOT NULL,
    [CommitMonth]                 TINYINT NOT NULL,    -- 1-12
    [StatusID]                    INT NOT NULL,        -- Code: Draft/Committed/Conflict
    [CommittedDate]               DATETIME NULL,
    [CommittedBy_FK]              INT NULL,
    [IsDeleted]                   BIT NOT NULL DEFAULT 0,
    [CreatedDate]                 DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]                INT NOT NULL,
    [ModifiedDate]                DATETIME NULL,
    [ModifiedBy_FK]               INT NULL
)
```

---

### Table: `TherapyShiftSlot`

Generated time slots after a monthly commit.

```sql
CREATE TABLE [dbo].[TherapyShiftSlot] (
    [TherapyShiftSlotID]             INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]                  INT NOT NULL,
    [TherapyShiftMonthlyCommitID_FK] INT NOT NULL,
    [RecipientType]                  NVARCHAR(50) NOT NULL,
    [RecipientID]                    INT NOT NULL,
    [SlotDate]                       DATE NOT NULL,
    [Period]                         CHAR(2) NOT NULL,
    [StartTime]                      TIME NOT NULL,
    [EndTime]                        TIME NOT NULL,
    [SessionDurationMins]            INT NOT NULL,
    [TierApplied]                    TINYINT NOT NULL,   -- 1, 2, or 3
    [SlotStatusID]                   INT NOT NULL,        -- Code: Open/Booked/Blocked
    [IsDeleted]                      BIT NOT NULL DEFAULT 0,
    [CreatedDate]                    DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]                   INT NOT NULL,
    [ModifiedDate]                   DATETIME NULL,
    [ModifiedBy_FK]                  INT NULL
)
```

---

### Table: `TherapyAppointmentRecurrenceSeries`

Recurring series definition. Recurrence columns mirror `PatientMedicationConsume`.

```sql
CREATE TABLE [dbo].[TherapyAppointmentRecurrenceSeries] (
    [RecurrenceSeriesID]          INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]               INT NOT NULL,
    [ParticipantType]             NVARCHAR(50) NOT NULL,
    [ParticipantID]               INT NOT NULL,
    [RecipientType]               NVARCHAR(50) NOT NULL,
    [RecipientID]                 INT NOT NULL,
    [PatientAICServicesID_FK]     INT NULL,
    [SeriesStartDate]             DATE NOT NULL,
    [SeriesEndDate]               DATE NULL,
    -- Recurrence columns (same as PatientMedicationConsume)
    [Recurrence_Frequency]        INT NULL,
    [Recurrence_Days]             INT NULL,
    [Recurrence_Interval]         INT NULL,
    [Recurrence_IntervalFlag]     INT NULL,
    [recurrenceDailyHourly]       INT NULL,
    [Every_Days]                  INT NULL,
    [Remarks]                     NVARCHAR(500) NULL,
    [IsDeleted]                   BIT NOT NULL DEFAULT 0,
    [CreatedDate]                 DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]                INT NOT NULL,
    [ModifiedDate]                DATETIME NULL,
    [ModifiedBy_FK]               INT NULL
)
```

---

### Table: `TherapyAppointment`

Individual appointment session.

```sql
CREATE TABLE [dbo].[TherapyAppointment] (
    [TherapyAppointmentID]        INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]               INT NOT NULL,
    [TherapyShiftSlotID_FK]       INT NULL,
    [ParticipantType]             NVARCHAR(50) NOT NULL,
    [ParticipantID]               INT NOT NULL,
    [RecipientType]               NVARCHAR(50) NOT NULL,
    [RecipientID]                 INT NOT NULL,
    [PatientAICServicesID_FK]     INT NULL,
    [RecurrenceSeriesID_FK]       INT NULL,
    [IsRecurring]                 BIT NOT NULL DEFAULT 0,
    [AppointmentDate]             DATE NOT NULL,
    [StartTime]                   TIME NOT NULL,
    [EndTime]                     TIME NOT NULL,
    [Remarks]                     NVARCHAR(500) NULL,
    [StatusID]                    INT NOT NULL,   -- Scheduled/Completed/Cancelled/NoShow
    [CompletionNotes]             NVARCHAR(1000) NULL,
    [CompletedDate]               DATETIME NULL,
    [CompletedBy_FK]              INT NULL,
    [IsBilled]                    BIT NOT NULL DEFAULT 0,
    [BilledDate]                  DATETIME NULL,
    [IsDeleted]                   BIT NOT NULL DEFAULT 0,
    [CreatedDate]                 DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]                INT NOT NULL,
    [ModifiedDate]                DATETIME NULL,
    [ModifiedBy_FK]               INT NULL
)
```

---

### Table: `TherapyLeaveApplication`

```sql
CREATE TABLE [dbo].[TherapyLeaveApplication] (
    [TherapyLeaveApplicationID]   INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]               INT NOT NULL,
    [RecipientType]               NVARCHAR(50) NOT NULL,
    [RecipientID]                 INT NOT NULL,
    [LeaveTypeID]                 INT NOT NULL,   -- Code: Annual/Medical/Emergency
    [StartDate]                   DATE NOT NULL,
    [EndDate]                     DATE NOT NULL,
    [Remarks]                     NVARCHAR(500) NULL,
    [StatusID]                    INT NOT NULL,   -- Pending/Approved/Rejected
    [ApprovedBy_FK]               INT NULL,
    [ApprovedDate]                DATETIME NULL,
    [IsDeleted]                   BIT NOT NULL DEFAULT 0,
    [CreatedDate]                 DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]                INT NOT NULL,
    [ModifiedDate]                DATETIME NULL,
    [ModifiedBy_FK]               INT NULL
)
```

---

### Table: `TherapyConflict`

Auto-detected: appointments during approved leave.

```sql
CREATE TABLE [dbo].[TherapyConflict] (
    [TherapyConflictID]            INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    [FacilityID_FK]                INT NOT NULL,
    [TherapyAppointmentID_FK]      INT NOT NULL,
    [TherapyLeaveApplicationID_FK] INT NOT NULL,
    [DetectedDate]                 DATETIME NOT NULL DEFAULT GETDATE(),
    [StatusID]                     INT NOT NULL,   -- Unresolved/Resolved
    [ResolutionTypeID]             INT NULL,        -- Reassigned/Rescheduled/Cancelled
    [ResolvedDate]                 DATETIME NULL,
    [ResolvedBy_FK]                INT NULL,
    [IsDeleted]                    BIT NOT NULL DEFAULT 0,
    [CreatedDate]                  DATETIME NOT NULL DEFAULT GETDATE(),
    [CreatedBy_FK]                 INT NOT NULL,
    [ModifiedDate]                 DATETIME NULL,
    [ModifiedBy_FK]                INT NULL
)
```

---

### Audit Tables

Each table has a mirror `Audit_TherapyXxx` with all columns + `AuditID`, `AuditAction`, `AuditDate`, `AuditBy_FK`.

---

### Migration File Naming

```
20260401_0900_01_TherapyShiftGlobalConfig.sql
20260401_0900_02_TherapyShiftDefaultTemplate.sql
20260401_0900_03_TherapyShiftOverride_and_Detail.sql
20260401_0900_04_TherapyShiftMonthlyCommit.sql
20260401_0900_05_TherapyShiftSlot.sql
20260401_0900_06_TherapyAppointmentRecurrenceSeries.sql
20260401_0900_07_TherapyAppointment.sql
20260401_0900_08_TherapyLeaveApplication.sql
20260401_0900_09_TherapyConflict.sql
20260401_0900_10_Audit_TherapyTables.sql
20260401_0900_11_Code_TherapySchedulingLookups.sql
```

---

## DAL Layer Pattern

```vb
' Data class mirrors DB row
Public Class TherapyAppointmentData
    Public Property TherapyAppointmentID As Integer
    Public Property FacilityID_FK As Integer
    Public Property ParticipantType As String
    Public Property ParticipantID As Integer
    Public Property RecipientType As String
    Public Property RecipientID As Integer
    Public Property PatientAICServicesID_FK As Integer?
    ' ... all columns
End Class

' DAL — all Public Shared, accepts userId As Integer, facilityId As Integer?
Public Class TherapyAppointmentDal
    Public Shared Function GetById(id As Integer) As TherapyAppointmentData
    Public Shared Function InsertUpdate(input As TherapyAppointmentData, userId As Integer, facilityId As Integer?) As Integer
    Public Shared Function MarkComplete(id As Integer, notes As String, userId As Integer, facilityId As Integer?) As Integer
    Public Shared Function SoftDelete(id As Integer, userId As Integer, facilityId As Integer?) As Integer
End Class
```

### Slot Generation Tier Resolution

```
GenerateSlotsForMonth(commitId):
  For each day in month:
    For each RecipientID in facility:
      1. Tier 1: TherapyShiftOverride active on date? → use detail rows
      2. Tier 2: TherapyShiftDefaultTemplate active for date? → use template
      3. Tier 3: TherapyShiftGlobalConfig → use global defaults
      Split working window into SessionDurationMins slots → insert TherapyShiftSlot rows
```

### Stage 1 AutoFill Logic

```
For each RecurrenceSeries in facility within month date range:
  Expand to occurrence dates using Recurrence_Frequency / Recurrence_Days / Every_Days
  For each occurrence date:
    Find open TherapyShiftSlot for same RecipientType/RecipientID on that date
    If found: create TherapyAppointment, mark slot Booked
    Else: log as failed → Stage 2 queue
```

---

## API Endpoints (`ScheduleController.vb`)

| Action | Method | Notes |
|---|---|---|
| `GetGlobalConfig` | GET | By facilityId |
| `SaveGlobalConfig` | POST | |
| `GetDefaultTemplates` | GET | By recipient |
| `SaveDefaultTemplate` | POST | |
| `GetOverrides` | GET | By recipient |
| `SaveOverride` | POST | Header + detail rows |
| `CommitMonth` | POST | Triggers slot generation |
| `GetSlots` | GET | By date/recipient/status |
| `BookAppointment` | POST | |
| `UpdateAppointment` | POST | |
| `CancelAppointment` | POST | |
| `MarkComplete` | POST | |
| `GetRecurrenceSeries` | GET | By participant |
| `SaveRecurrenceSeries` | POST | |
| `ChangeTherapist` | POST | scope: 'one' only |
| `RunStage1AutoFill` | POST | By commitId |
| `GetUnfilledAppointments` | GET | Stage 2 queue |
| `AssignSlotManual` | POST | Stage 2 |
| `GetConflicts` | GET | Unresolved only |
| `ResolveConflict` | POST | Reassign/Reschedule/Cancel |

---

## Feature Folder Structure

```
SourceCode/WebApp/AppCode/DataAccessLayer/
  TherapyShiftGlobalConfig.vb
  TherapyShiftDefaultTemplate.vb
  TherapyShiftOverride.vb
  TherapyShiftMonthlyCommit.vb
  TherapyShiftSlot.vb
  TherapyAppointmentRecurrenceSeries.vb
  TherapyAppointment.vb
  TherapyLeaveApplication.vb
  TherapyConflict.vb

SourceCode/WebApp/AppCode/Controller/
  ScheduleController.vb

SourceCode/WebApp/TherapySchedule/
  ShiftSetup.aspx(.vb)
  Schedule.aspx(.vb)
  BookAppointment.aspx(.vb)
  Appointments.aspx(.vb)
  RecurringEdit.aspx(.vb)
  TherapistChange.aspx(.vb)
  Stage1Automation.aspx(.vb)
  Stage2Manual.aspx(.vb)
  Conflicts.aspx(.vb)
```

---

## Phased Delivery

| Phase | Deliverable |
|---|---|
| 0 | Feature folder scaffold, confirm no existing scheduling table conflicts |
| 1a–1e | All DB migration scripts + audit tables + Code lookup rows |
| 2a–2e | All Data + Dal classes, slot generation, Stage1 logic |
| 3a–3c | ScheduleController endpoints |
| 4a–4f | WebForms pages wired to API (follow CM-7429 mockups) |

---

## Reference Files

| Resource | Path |
|---|---|
| UI Mockups | `C:\Users\USER\Documents\MARKDOWN_notes\CM-7429\*.html` |
| Copilot Instructions | `.github/copilot-instructions.md` |
| Standards | `.github/STANDARDS.md` |
| Recurrence Reference | `DataAccessLayer/MedicationAdministration.vb` Line 1377 |
| WeekDayBitType Enum | `DataAccessLayer/Task.vb` |
| PatientAICServices DAL | `DataAccessLayer/PatientAICServices.vb` |
| DB Migrations | `Database/` |
