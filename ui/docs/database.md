## Database

The TUI keeps a local database of the courses and of the assignments that have
been graded, so that course information does not have to be typed again and past
grades and comments can be looked up without opening Canvas or the spreadsheets.

### Storage

- **SQLite**, through Python's standard `sqlite3` module (no ORM, no extra package).
- One file: `data/canvas.db`. `data/` is in `.gitignore`: the database holds
  student names, SIS User IDs, emails and grades, so it must never be committed, uploaded
  or sent to any service. It never stores API keys.
- The database is a **local record, not the source of truth**: Canvas holds the
  official grades, and the grading files in `data/<course>/<assignment>/` are what
  the tasks read. The TUI never pushes grades to Canvas from the database; that
  is always done by running `push_grades.py` or a workflow.
- All access goes through `ui/db.py`. Open connections with
  `PRAGMA foreign_keys = ON`; use parameterized queries only (`?` placeholders, never
  string formatting); write each import in a single transaction, so that a failed
  import leaves the database unchanged.
- Version the schema with `PRAGMA user_version`: `db.py` creates the tables when
  the version is 0 and applies numbered migrations in order. Never change the
  schema without a migration.
- Timestamps are ISO 8601 text in UTC (`2026-11-14T09:30:00Z`).

### Schema

```sql
-- A course, with the information of {course_dir}/course_info.yaml
-- (see "Collect grades and attendance report" in tasks/AGENTS.md)
CREATE TABLE courses (
    id            INTEGER PRIMARY KEY,   -- local ID
    name          TEXT NOT NULL,
    course_dir    TEXT UNIQUE,           -- directory with course_info.yaml, if any
    penalty       REAL,                  -- points deducted per absence (update_attendance)
    destination_group TEXT,              -- for gather_grades: a group of the main Canvas course,
                                         -- given (and remembered) when gather_grades is run
    completed_at  TEXT,                  -- when the course was marked complete; NULL while
                                         -- it is open. Once set, never cleared (see
                                         -- "Completed courses")
    created_at    TEXT NOT NULL,
    updated_at    TEXT NOT NULL
);

-- The Canvas courses of a course (one or more), each with its Google Spreadsheet
-- (oral grades, attendance) and Drive folder. Exactly one of them is the main
-- Canvas course: gather_grades.py pushes "Điểm quá trình" of all the students of
-- the course into the destination_group of the main course.
CREATE TABLE canvas_courses (
    course_id         INTEGER NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    canvas_course_id  INTEGER NOT NULL,
    is_main           INTEGER NOT NULL DEFAULT 0 CHECK (is_main IN (0, 1)),
    name              TEXT,              -- name in Canvas
    spreadsheet_id    TEXT,              -- spreadsheet_ids[i], i = index in course_ids
    directory_id      TEXT,              -- directory_ids[i]
    attendance_assignment_id INTEGER,    -- "Điểm điều kiện thi KTHP"
    PRIMARY KEY (course_id, canvas_course_id)
);

-- At most one main Canvas course per course (at least one is checked on saving)
CREATE UNIQUE INDEX canvas_courses_one_main ON canvas_courses (course_id) WHERE is_main = 1;

-- The assignment groups of the Canvas courses of a course, with their weights
-- (read from Canvas when the course is created, the 0-weight ones left out; Edit may
-- still set a weight to 0); gather_grades gathers those marked gathered with a
-- weight > 0, except destination_group
CREATE TABLE assignment_group_weights (
    course_id         INTEGER NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    name              TEXT NOT NULL,
    weight            REAL NOT NULL,     -- % (group_weight in Canvas), may be 0
    canvas_course_id  INTEGER,           -- the Canvas course of the group, if known (migration 3)
    gathered          INTEGER NOT NULL DEFAULT 1 CHECK (gathered IN (0, 1)),
                                         -- 0: kept here, not gathered by gather_grades (migration 4)
    PRIMARY KEY (course_id, name)
);

CREATE TABLE students (
    sis_user_id  TEXT PRIMARY KEY,
    name         TEXT NOT NULL,
    email        TEXT                    -- from Canvas; NULL until fetched
);

-- A graded assignment (Canvas or PUOJ) and where its grading files are
CREATE TABLE assignments (
    id                    INTEGER PRIMARY KEY,
    canvas_course_id      INTEGER NOT NULL,
    canvas_assignment_id  INTEGER,       -- NULL for a PUOJ assignment not created yet
    name                  TEXT NOT NULL,
    kind                  TEXT NOT NULL CHECK (kind IN ('canvas', 'puoj')),
    points                REAL,          -- maximal points
    assignment_path       TEXT NOT NULL UNIQUE,  -- data/<course>/<assignment>
    grading_scheme        TEXT,          -- content of grading_scheme.txt, if any
    graded_at             TEXT,          -- when the grades were collected
    pushed_at             TEXT,          -- when push_grades.py succeeded, if known
    imported_at           TEXT NOT NULL
);

-- One criterion per problem or part: "3a (2)" → problem 3, part a, 2 points
CREATE TABLE criteria (
    id             INTEGER PRIMARY KEY,
    assignment_id  INTEGER NOT NULL REFERENCES assignments(id) ON DELETE CASCADE,
    problem        INTEGER NOT NULL,
    part           TEXT NOT NULL DEFAULT '',   -- '' when the problem has no parts
    max_points     REAL NOT NULL,
    UNIQUE (assignment_id, problem, part)
);

-- Grade and comment of a student for one criterion
CREATE TABLE grades (
    criterion_id    INTEGER NOT NULL REFERENCES criteria(id) ON DELETE CASCADE,
    sis_user_id     TEXT NOT NULL REFERENCES students(sis_user_id),
    score           REAL,                -- "3a (2)": original grade
    modified_score  REAL,                -- "3a mod (2)": after the grading scheme (pushed)
    comment         TEXT,                -- "Comment 3a"
    PRIMARY KEY (criterion_id, sis_user_id)
);

-- Per student: submitted files and totals of an assignment
CREATE TABLE submissions (
    assignment_id         INTEGER NOT NULL REFERENCES assignments(id) ON DELETE CASCADE,
    sis_user_id           TEXT NOT NULL REFERENCES students(sis_user_id),
    files                 TEXT,          -- JSON list of the "Submission k" files
    total_grade           REAL,          -- "Total grade"
    total_modified_grade  REAL,          -- "Total modified grade"
    PRIMARY KEY (assignment_id, sis_user_id)
);

-- A completed course is frozen: the database itself refuses any change to it
CREATE TRIGGER courses_completed_frozen BEFORE UPDATE ON courses
WHEN OLD.completed_at IS NOT NULL
BEGIN SELECT RAISE(ABORT, 'course is completed'); END;

CREATE TRIGGER courses_completed_no_delete BEFORE DELETE ON courses
WHEN OLD.completed_at IS NOT NULL
BEGIN SELECT RAISE(ABORT, 'course is completed'); END;

-- Same three triggers (BEFORE INSERT using NEW.course_id, BEFORE UPDATE using
-- OLD.course_id or NEW.course_id, BEFORE DELETE using OLD.course_id) on
-- canvas_courses and on assignment_group_weights, e.g.:
CREATE TRIGGER canvas_courses_completed_insert BEFORE INSERT ON canvas_courses
WHEN (SELECT completed_at FROM courses WHERE id = NEW.course_id) IS NOT NULL
BEGIN SELECT RAISE(ABORT, 'course is completed'); END;

-- and on assignments, for an assignment of a Canvas course of a completed course
-- (BEFORE INSERT using NEW.canvas_course_id, BEFORE UPDATE using OLD or NEW,
-- BEFORE DELETE using OLD.canvas_course_id), e.g.:
CREATE TRIGGER assignments_completed_insert BEFORE INSERT ON assignments
WHEN EXISTS (SELECT 1 FROM canvas_courses cc JOIN courses c ON c.id = cc.course_id
             WHERE cc.canvas_course_id = NEW.canvas_course_id
               AND c.completed_at IS NOT NULL)
BEGIN SELECT RAISE(ABORT, 'course is completed'); END;
```

Criteria, grades and submissions have no triggers of their own: they only change
through an import of their assignment, which also updates its `assignments` row
(`imported_at`) in the same transaction, so the trigger on `assignments` rolls the
whole import back.

### Filling the database (`ui/importers.py`)

The tasks are not changed to write into the database: the TUI reads the files they
produce.

- **Courses:** created in two steps from the home screen ("New course" in
  design.md): the user enters the Canvas courses and the penalty, and the course
  name and the assignment groups with their weights are read from Canvas
  (read-only, `get_course` and `get_assignment_groups`, `group_weight`); edited in
  **Course setting**; or imported from an existing
  `{course_dir}/course_info.yaml`, where the first entry of `course_ids` becomes
  the main Canvas course. Importing is only for a course that is not created yet:
  it is refused when the `course_dir`, or one of the Canvas courses, is already in
  the database (edit the course in **Course setting** instead). Canvas course names are fetched only when the user asks ("Refresh
  from Canvas"), read-only.
- **Students' emails:** fetched from Canvas, read-only, only when the user asks
  ("Refresh from Canvas" in Course setting): for each Canvas course of the
  course, the students with
  `course.get_users(enrollment_type=["student"], include=["email"])`, matched to
  `students` by `sis_user_id`. A student not yet in `students` is added with their
  Canvas name and email; for an existing student, `email` is updated and the
  name is kept. A student without an SIS User ID or without an email in Canvas is
  skipped. All the updates of one refresh are done in one transaction; report how
  many emails were added or changed. Emails are never read from the grading files
  and an import of an assignment never changes them.
- **Graded assignments:** imported from an assignment directory: `metadata.json`
  (course, assignment ID and name, points), `grades.xlsx` (the grades) and
  `grading_scheme.txt` if present. Read the spreadsheet header with the patterns
  of `grading/tasks/collect_grades.py` (`SCORE_HEADER`, `MOD_HEADER`,
  `COMMENT_HEADER`, `SUBMISSION_HEADER`, `TOTAL_HEADER`, `TOTAL_MOD_HEADER`) and
  `HEADER_ROW` of `prepare_spreadsheet.py`, by importing them, not copying them,
  so that the import follows any change of the spreadsheet layout.
- Importing an assignment again **replaces** its criteria, grades and submissions
  (in one transaction), after showing how many grades will change and asking.
- After a run of `collect_grades.py`, `bin/run_canvas.sh` or `bin/run_puoj.sh`
  ends successfully, offer "Save these grades to the database". After a successful
  `push_grades.py` (not a dry run), set `pushed_at`.
- An "Import" action also takes any assignment directory under `data/`, so that
  assignments graded before the TUI existed can be added.

### Generating `course_info.yaml` (`ui/course_info.py`)

The Canvas courses and the penalty are entered by the user, the course name and the
assignment groups with their weights are read from Canvas, and `destination_group`
is given when `gather_grades.py` is run. Together they are used to generate
`{course_dir}/course_info.yaml`, the file read by `bin/report_attendance.py` and
`tasks/gather_grades.py`.

- **New course** (home screen, `ui/screens/new_course.py`), two steps:
  1. the Canvas courses, one row each (Canvas course ID, `spreadsheet_id`,
     `directory_id`, a **Main** radio button; as many rows as needed), and
     `penalty` (a number > 0);
  2. the **course summary** screen, read from the Canvas courses (read-only;
     **Refresh** reads them again): the course name is the name of the main Canvas course; the assignment groups
     with a weight > 0, with their weights and their Canvas course (the 0-weight
     groups are left out, and listed as such), each with a checkbox, all ticked
     by default: every group is saved, the unticked ones marked as not gathered
     (`gathered = 0`); only the ticked ones are written into `assignment_groups` of
     `course_info.yaml` (the choice is kept across **Refresh**, and can be changed
     later with **Edit**). A group name found in two Canvas
     courses is kept once and reported, since `gather_grades.py` stops on it. The
     course directory is `{parent}/{course name}`, the parent being `data/` by
     default or a directory chosen by the user; it is created on saving, with its
     `course_info.yaml` (an existing file is kept).
- **Opening an existing course** (`enter` on the home screen): its Canvas courses
  are read first (read-only). When the assignment groups with a weight > 0 (names
  and weights) are the same as those of the database, gathered or not, the task
  list opens directly; otherwise the course summary screen is shown with the
  differences (new, removed, weight changed), the database's choice of gathered
  groups (a group new on Canvas ticked) and the course directory kept as it is.
  **Save** updates the course (same ID, `destination_group` kept) and rewrites
  `course_info.yaml` when it differs, then opens the task list; **Back** returns
  to the list of courses with nothing changed. A completed course is not checked;
  when Canvas cannot be read, the course is used as it is, with a warning.
- **Edit** (**Course setting**, `c`): the edit form of the chosen course, shown
  directly: all the fields, `destination_group` optional, the groups as an editable
  table *group name → weight (%)*, with a **Gathered** checkbox on each row; the
  total shown is that of the gathered groups. **Save** stores the course and
  rewrites `{course_dir}/course_info.yaml` when it differs (the old one kept as
  `.bak`, then read back with `read_config`).
- **Validation** before saving: `course_ids` non-empty integers without
  duplicates; exactly one main Canvas course; a spreadsheet ID and a Drive folder ID
  for each Canvas course; `penalty` > 0; group names non-empty and unique, weights
  numbers ≥ 0, at least one group to gather; `destination_group`, when given,
  leaves at least one group to gather (that it is a group of the main Canvas course
  is checked when `gather_grades.py` is run).
  Warn (do not block) when the weights of the gathered groups do not add up to 100.
- **Saving** stores the course in the database (`courses`, `canvas_courses`,
  `assignment_group_weights`) in one transaction.
- **destination_group** is chosen in the form of `gather_grades.py` among all the
  groups of the main Canvas course, 0-weight ones included, read from Canvas
  (read-only) when the form opens; until then, or if Canvas cannot be read, among
  the groups of the database and the last destination (preselected). When the YAML file of
  the form is the course's own `course_info.yaml`, the TUI saves the choice in the
  database and writes the file with it (the old one kept as `.bak`, then read back
  with `read_config`) after the confirmation and before running; any other YAML
  file is used as it is.
- **"Generate course_info.yaml"** builds the file from the database with
  `yaml.safe_dump(..., allow_unicode=True, sort_keys=False)`. `gather_grades.py`
  takes the first entry of `course_ids` as the main course, so `course_ids` lists
  the main Canvas course first, then the others in any order (by
  `canvas_course_id`, so that the generated file does not change from one run to
  the next); `spreadsheet_ids` and `directory_ids` follow the same order as
  `course_ids`. `destination_group` is written once it is known;
  `assignment_groups` holds the gathered groups with a weight > 0 except
  `destination_group`. Keys in this order:

  ```yaml
  course_ids: [12345, 12346]
  spreadsheet_ids: [1AbC..., 1DeF...]
  directory_ids: [1GhI..., 1JkL...]
  penalty: 0.5
  destination_group: Điểm quá trình
  assignment_groups:
    Bài tập: 40
    Vấn đáp: 30
    Kiểm tra giữa kỳ: 30
  ```

  It shows the YAML in a preview first. If the file already exists, it shows the
  differences and asks before overwriting; the old file is kept as
  `course_info.yaml.bak`. Weights are written as integers when they are whole
  numbers.
- After writing, read the file back with `tasks.gather_grades.read_config`
  (imported, not run) and check that it gives the same `course_ids`,
  `destination_group` and `assignment_groups`; report any difference as an error.
- The YAML file is generated from the database, so edit the course in the TUI, not
  the file; a file edited by hand is not imported again (the course is already
  created): make the same change with **Edit**, then generate the file.

### Using the database in the TUI

- **Course setting** (`c`, "Course setting" in the bottom bar): the edit form of
  the chosen course, shown directly (no list of courses), with the buttons
  **Save**, **Refresh from Canvas**, **Mark as complete**, **Delete** and **Back**;
  under the form, the Canvas course names and attendance assignments read by
  Refresh. With no course chosen yet, `c` says to choose one first. Courses are
  chosen, created (**New course**) and imported (`course_info.yaml` of a course not
  created yet) on the home screen. Choosing a course fills
  in the course ID, spreadsheet IDs, Drive folder IDs and penalty in the forms of
  the attendance tasks and of `bin/report_attendance.py`, and the YAML file of
  `gather_grades.py`.
- **Grades screen** (`g`): the graded assignments of the chosen course only (no
  other course can be chosen there; with no course chosen, `g` says to choose one
  first); importing an assignment of another course's Canvas course is refused,
  and "Import new from data/" skips them. For an assignment, the table of
  students with their total, modified total and each criterion; select a student
  to read the comments in full (with the student's email, if known). Filter by
  student name, SIS User ID or email, and see one student's grades across all
  assignments of a course.
- **Export** an assignment, or a student's grades, to CSV in `data/exports/`,
  with an `Email` column after the name.
- Deleting a course or an assignment asks for confirmation; it only deletes rows in
  the database, never files in `data/` or anything in Canvas. A completed course
  and its assignments cannot be deleted.

### Completed courses

When a course is over (final grades pushed), the user marks it complete, and from
then on the TUI allows no update of it and no push to its Canvas courses.

- **Marking:** "Mark as complete" in Course setting sets `completed_at`. It
  asks for confirmation and says that it cannot be undone; there is no "reopen".
- **In the database:** no change to the course, its Canvas courses, its
  assignment group weights or its assignments (and so their criteria, grades and
  submissions), and no deletion. `db.py` checks `completed_at` and refuses with a
  clear message; the triggers above are the safety net if the code forgets.
  Students' rows are shared between courses and are not frozen.
- **In the TUI**, for a completed course:
  - Course setting shows it as completed and read-only: the fields, Save,
    Refresh from Canvas, Mark as complete and Delete are disabled; only Back is
    left (its `course_info.yaml` cannot be imported again, as for any created
    course); it can still be chosen on the home screen;
  - the grades screen shows its assignments read-only: no import, re-import or
    delete; export is allowed;
  - "Save these grades to the database" is not offered, and `pushed_at` is not
    set, for an assignment of its Canvas courses;
  - **no push:** a task that writes to Canvas (`writes_canvas` in the catalogue)
    does not run, not even as a dry run, when the Canvas course it writes to
    belongs to a completed course. The form finds that Canvas course from the
    course ID fields, the `course_ids` of the YAML file (`gather_grades.py`), or the
    `metadata.json` of the assignment directory (`push_grades.py`, the grading
    workflows), and shows an error on the field instead of running. Tasks that only
    read from Canvas (e.g. `download_data`) still run.
- These checks are in the TUI only: the tasks themselves are not changed, so a
  task run from the shell outside the TUI is not blocked.

