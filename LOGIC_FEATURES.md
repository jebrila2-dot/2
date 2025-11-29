# Digital Implant Planning - Logic & Business Features

## 🔄 Core Business Logic

### **Phase-Based Treatment Workflow**
The system enforces a structured 5-phase treatment process:

1. **Planning Phase** (Entry point)
   - Contains 3 sub-activities: Prosthetic Work Up → CBCT Scan → 3D Planning
   - Cannot skip to next phase until planning is complete
   - Generates "Planning Complete" outstanding action when finished

2. **Labwork Phase** (Sequential)
   - Required after Planning
   - Single completion milestone
   - Generates "Labwork Complete" outstanding action
   - Can trigger automatic phase completion via outstanding action

3. **Pre-Implant Surgery Phase** (Conditional)
   - Optional - only activated if patient needs bone graft or sinus lift
   - Contains conditional sub-activities:
     - Bone Graft OR Sinus Lift (procedure type selected)
     - Healing Period (required after procedure)
     - Updated 3D Planning (after healing)
   - If not needed, skips directly to Implant Surgery
   - Generates procedure-specific outstanding actions

4. **Implant Surgery Phase** (Sequential)
   - Follows Pre-Implant (or Planning if pre-implant skipped)
   - Contains: Treatment → Healing Period → Prosthesis Fit
   - Each completion generates corresponding action

5. **Review & Completion Phase** (Final)
   - Final patient review and discharge milestone
   - Marks treatment journey as complete
   - Generates "Patient Discharged" action

**Key Logic**: Patient can only progress to next phase when previous phase is complete. Pre-implant surgery is conditionally activated based on clinical need.

---

### **Multiple Journey Support Per Patient**
- Single patient can have multiple independent treatment journeys
- Each journey has its own complete phase workflow
- Each journey is tracked separately with distinct outstanding actions
- Allows clinic to handle:
  - Multiple implant locations on same patient
  - Retreatment cases
  - Revisions or follow-up procedures
- Journeys don't interfere with each other—independent timelines and phases

---

### **Outstanding Actions Management**

#### **Automatic Generation Logic**
Outstanding actions are automatically created when:
- **Phase completion** triggers action (e.g., "Labwork Phase Complete" → "Labwork Complete" action)
- **Sub-activity completion** creates specific action (e.g., "Bone Graft Done" → "Bone Graft Complete" action)
- **Manual creation** via UI for ad-hoc tasks

#### **Action State Machine**
```
Created → Active → [Completed or Snoozed or Deleted]
         ↓
    Snoozed → Returns to Active after snooze period
```

#### **Action Lifecycle Logic**
- **Creation**: Automatically generated or manually created with action type, patient, and journey reference
- **Display**: Shown in Outstanding Actions page, dashboard, and patient views
- **Completion**: Mark as complete (deleted from system, not archived)
- **Snooze**: Temporarily hide action for X days, then reappears
- **Deletion**: Can be deleted directly without completion
- **Reference**: Each action links back to patient and journey for context

#### **Action-Phase Integration**
- Completing a "Labwork Phase Complete" action automatically triggers labwork phase completion
- Provides bidirectional link: Actions ↔ Phase Progress

---

### **Meeting Logic**

#### **Multi-Patient Meetings**
- Single meeting can include multiple patients
- Each patient within a meeting is independent:
  - Has meeting-specific notes (different from patient notes)
  - Can be marked as "actioned" individually
  - Can be removed from meeting independently
- Meeting persists even if patients are added/removed

#### **Meeting-Patient Relationship**
```
Meeting 1 ─── Patient A (notes: "...", actioned: true)
         └── Patient B (notes: "...", actioned: false)
         └── Patient C (notes: "...", actioned: true)
```

#### **Meeting Data Persistence**
- Patient's meeting-specific notes stored in `meetingPatients.notes`
- Patient's "actioned" status stored in `meetingPatients.actioned`
- Patient's general notes (separate) stored in `patients.notes`
- Meeting notes are preserved even after meeting date passes

#### **Meeting Filtering Logic**
- **Future Meetings**: Meeting date ≥ today's date
- **Previous Meetings**: Meeting date < today's date
- Automatic categorization for easy access

---

### **Auto-Save Notes Logic**

#### **Debounced Saving**
- User types patient notes → 1-second debounce timer starts
- If user continues typing, timer resets
- After 1 second of inactivity → automatically saves to database
- **Benefit**: Reduces database writes, prevents data loss without explicit save button

#### **Timestamp Tracking**
- Each note save updates `patients.notesUpdatedAt` timestamp
- Tracks when notes were last modified
- Useful for audit trail and recent activity

#### **Meeting Notes vs Patient Notes**
- Patient notes: Global, stored in `patients.notes` (across all meetings)
- Meeting notes: Local to specific meeting, stored in `meetingPatients.notes`
- Both auto-save independently with debounce

---

### **Date Validation Logic**

#### **Flexible Input Processing**
Input formats accepted:
- `DD-MM-YYYY` (strict)
- `D-M-YYYY` (single digits auto-normalize)
- `DD/MM/YYYY` (slash separator)
- `D/M/YYYY` (slash with single digits)

Examples that normalize:
- `5-1-2024` → `05-01-2024`
- `2/12/2024` → `02-12-2024`

#### **Validation Rules**
Rejects impossible dates:
- Month outside 1-12 range
- Day outside 1-31 range
- Invalid days for specific months:
  - February: 1-29 (leap year aware)
  - April, June, September, November: 1-30
  - Others: 1-31
- Year outside 1900-2100 range
- Non-numeric values

#### **Auto-Correction Prevention**
JavaScript's `Date` constructor auto-corrects invalid dates:
- Input: `30-02-2024` → JS converts to `01-03-2024`
- **App detects this and rejects**: User sees "Invalid Date" error
- Ensures data integrity—no silently corrupted dates

#### **Display Format**
All dates displayed as `DD-MM-YYYY` throughout app regardless of input format

---

### **Photo Management Logic**

#### **Automatic Compression**
- User uploads photo → Client-side compression occurs
- Reduces file size without quality loss
- Benefits:
  - Faster uploads
  - Reduced storage
  - Preserves patient privacy (no high-res originals)

#### **Gallery Organization**
- Photos linked to specific patient
- Multiple photos per patient supported
- Photos persist in patient profile across sessions
- Deleted photos removed from database and storage

---

### **Export & Reporting Logic**

#### **Report Generation**
Data aggregated from:
- All patients (count, phase distribution)
- All journeys (active vs completed)
- All meetings (total, patient participation)
- Outstanding actions (by type, age)

#### **Export Formats**

**PDF Export:**
- Professional report layout
- Includes charts, statistics, and patient details
- Suitable for printing and distribution
- Client-side generation

**Excel Export:**
- Tabular format for data analysis
- Multiple sheets for different data types
- Sortable and filterable columns
- Suitable for import into other systems

---

### **Patient Status Tracking Logic**

#### **Current Phase Assignment**
Each patient has a "current phase" based on their most recent journey:
- System determines which phase is active (Planning, Labwork, Pre-Implant, Surgery, Review)
- Phase changes as journey progresses
- Used for "Patients by Phase" dashboard view

#### **Phase Completion Tracking**
- Each phase has completion flag
- Sub-activities within phase have individual completion flags
- Can't move to next phase until current phase marked complete
- Enforces workflow progression

---

### **Data Consistency Logic**

#### **Referential Integrity**
- Patient ← Journey ← Outstanding Actions (if journey deleted, actions deleted)
- Patient ← Meeting Attendance (if patient deleted, meeting attendance removed)
- Patient ← Photos (if patient deleted, photos removed)

#### **Atomic Operations**
Multi-step operations use database transactions:
- Completing phase + creating outstanding action = atomic
- Prevents partial updates if database fails
- Ensures consistency

---

## 🎯 Advanced Logic Patterns

### **Conditional Workflow**
Pre-implant surgery is conditionally activated based on:
- Clinical decision (patient needs bone graft/sinus lift)
- Not automatically triggered
- If skipped, phase workflow continues normally

### **State Preservation**
- Meeting notes persist even after meeting date passes (historical record)
- Outstanding actions remain visible until completed
- Patient notes auto-save to prevent loss

### **Cascading Actions**
- Complete phase → Auto-create outstanding action
- Complete outstanding action (labwork type) → Auto-advance phase
- Add patient to meeting → Initialize meeting notes and actioned state

---

## 📋 System Rules & Constraints

| Rule | Logic |
|------|-------|
| **One Current Phase per Journey** | Only one phase can be "active" at a time |
| **Sequential Phases** | Phases must be completed in order (with Pre-Implant conditional) |
| **Independent Journeys** | Multiple journeys don't affect each other |
| **Action-Phase Bidirectional** | Actions can trigger phase completion and vice versa |
| **Note Persistence** | Notes saved even if meeting/patient deleted later |
| **Date Immutability** | Past dates in meetings/journeys don't change based on current date |
| **Multi-Patient Meetings** | Meeting persists even if all patients removed |
| **Auto-Save Safety** | Unsaved changes indicated visually (no data loss) |

---

**Summary**: The app enforces a rigorous phase-based workflow while supporting flexible multi-journey tracking, intelligent outstanding action management, and preserves all historical data throughout the treatment lifecycle.
