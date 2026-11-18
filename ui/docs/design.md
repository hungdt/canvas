## Design

### Task catalogue (`catalog.py`)

One declarative entry per task: a title, the group it belongs to, a short
description, the command to run, and its parameters (name, kind, required or not,
default, help text). The forms, validation and command lines are all generated from
this catalogue, so adding a task means adding one entry.

Parameter kinds: integer ID, float, text, a list of texts (e.g. several spreadsheet
IDs), a file path, a directory path, and a flag (checkbox). An optional parameter
that is left empty is **not passed**, so the task uses its own default (e.g. a
course ID read from cell A1).

Tasks are run as `canvas/bin/python -m <module> …` from the project directory;
workflows as `bin/<script> …`.

### Running a task (`runner.py`)

- Use `asyncio.create_subprocess_exec` (never `shell=True`; pass the arguments as
  a list) inside a Textual worker, so that the interface stays responsive.
- Set `PYTHONUNBUFFERED=1`. Read the output in chunks, not lines, because a
  question such as `Proceed? [y/N] ` does not end with a newline.
- The output goes to a `RichLog`, with ANSI colours converted by `Text.from_ansi`.
  The bash workflows only use colours on a TTY; the TUI shows their text as it is.
- Show the full command line above the output, like the bash workflows do.
- **Prompts:** when the output ends with a question and no newline, show an input
  line under the log (or a yes/no dialog when the question ends with `[y/N]`), and
  write the answer and a newline to the subprocess's stdin. The prompts that exist
  today are listed under "Interactive prompts".
- A **Cancel** button (and `ctrl+c`) asks for confirmation, then terminates the
  subprocess. Warn the user that a cancelled task may have written part of its
  changes to Canvas.
- When the subprocess ends, show its exit status (success / failure). Keep the log
  of every run in the history screen, and offer to save it to a file in `data/logs/`.
- Only one task runs at a time.

### Screens

- **Home** works in two steps: first choose the course, then the task.
  1. **Choose a course.** The TUI opens on the list of the courses of the
     database (see "Database"): name, main Canvas course, the other Canvas courses,
     `course_dir`, and whether it is completed. Open courses come first, most
     recently updated first; completed ones after them, marked as completed. `/`
     searches by name or Canvas course ID.
     - `enter` on a course continues the work on that course;
     - **New course** opens the course form with the following fields:
       - `penalty`: a float value (on top)
       - Button to add new Canvas course as many as the user wants (above the list of Canvas courses)
       - the Canvas courses with their spreadsheet and Drive folder IDs and the radio button for the main one 
     - The course summary screen: additional information for the course (for both existing course and new course)
       - Read all Canvas courses to collect information: main Canvas course name, all assignment groups with their corresponding weight. Excludes all assignment groups with zero weight.
       - If the list of assignment groups (gathered or not) is unchanged as compared to that in the database, ignore this screen
       - Display the course name is the name of the main Canvas course
       - Option for choosing directory of the course: Parent directory for the course (`course_dir`) is the default directory, then the course directory is created with the name the same as the course name. User has an option to choose a different parent directory for the course
       - Display all assignment groups and their corresponding weights: User can choose from this list the assignment groups that will be used for `gather_grades.py` (which is included in `course_info.yaml`)
       - Button "Refresh" to reread the data from Canvas courses and display the information again
     - The course is validated and saved in the database before the task list is shown; cancelling the form goes back to the list of courses.
     - When the database has no course yet, the list only shows **New course** and
       **Import course_info.yaml**.
  2. **Tasks and workflows of the course.** The course's name and Canvas courses
     are shown at the top; under them, the tasks grouped as in the Task Catalogue
     section, with a short description of each. `enter` opens the form, `/`
     searches, and **Change course** (`escape`) goes back to step 1 (`c` stays
     the shortcut of the **Course setting** screen, "Course setting" in the bottom bar).
     - The chosen course fills the course fields of every form (Canvas course ID,
       spreadsheet IDs, Drive folder IDs, penalty, the YAML file of
       `gather_grades.py`); the user can still change them in the form. A task run
       on several Canvas courses one at a time (e.g. `update_attendance`) asks in
       its form which Canvas course of the course to use, the main one by default.
     - For a completed course (see "Completed courses" in "Database"), the tasks
       that write to Canvas are shown disabled, with the reason; the other tasks
       stay available.
  The last chosen course is remembered with the last values of the forms, and is
  selected (not opened) in the list the next time the TUI starts.
- **Form:** one field per parameter, with a help text, required fields marked,
  integers/floats checked before running, path fields with a directory picker
  (`DirectoryTree`) starting from `data/`. Show the command line that will be run
  as the fields change. **Run** opens the run screen.
- **Run:** command, live output, prompt line, Cancel, exit status, and "Run again"
  (back to the form with the same values).
- **History:** the runs of this session; select one to see its log again.
- **Course setting** (`c`): the "Edit the course" form of the chosen course,
  shown directly, with the buttons **Save**, **Refresh from Canvas**, **Mark as
  complete**, **Delete** and **Back** (see "Using the database in the TUI" in
  database.md). Read-only for a completed course.

Remember the last values of each form (not the API key), and the last chosen
course, in `~/.config/canvas-ui/last_values.json`, so that a course ID or a
spreadsheet ID does not have to be typed again. Values filled from the chosen
course take precedence over the remembered ones.

### Tasks that change Canvas

Before running a task that writes to Canvas (marked `writes_canvas` in the
catalogue), show a confirmation with the full command. When the task has a dry-run
option (`push_grades.py --dry-run`), offer "Dry run" as the default action, with
"Run for real" next to it.

### Course specific

When a course is chosen, all tasks and workflows are specifically for this course: all information of the course is used in forms for the tasks and workflows.

User can only another course if they go back to the main screen and switch to another course.

At the bottom bar, menu item "Course setting" for setting the information of the current course only.

### Course settings

The screen "Course settings" can be accessed from the bottom menu bar.

Provide possiblity for the users to change:
 - `course_dir`
 - Canvas courses
 - List of assignment groups for gathering grades
 - Penalty

Buttons:
 - Save: for saving the data
 - Refresh from Canvas: read the data from Canvas again for refreshment
 - Mark as complete: mark the course as complete
 - Delete: delete the course from the database, ask if the user also wants to delete the `course_dir`
 - Back: go back to previous screen