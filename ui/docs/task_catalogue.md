## Task catalogue

The specifications of the tasks are in `tasks/TASKS.md`, `tasks/AGENTS.md`,
`grading/AGENTS.md`, `grading/tasks/AGENTS.md` and `grading/tasks/puoj/AGENTS.md`.
Read the module's `argparse` definition before writing its catalogue entry: the
arguments below may have changed since this file was written.

### Group "Attendance and grades" (`tasks/`)

| Task | Command | Parameters | Writes to |
|---|---|---|---|
| Collect oral (Q&A) grades | `python -m tasks.collect_oral_grades` | `spreadsheet_id` (required); `--sheet-name` (default "Vấn đáp"); `--course-id`; `--assignment-id` | Canvas |
| Prepare the attendance report | `python -m tasks.prepare_attendance` | `google_drive_folder_id` (required) | Google Sheets, Canvas (gives 0 in quizzes) |
| Update the attendance in Canvas | `python -m tasks.update_attendance` | `spreadsheet_ids` (one or more, the first is the main one); `--penalty` (required float); `--course-id`; `--assignment-id` | Canvas |
| Gather grades for finalizing | `python -m tasks.gather_grades` | `yaml_filename` (required, file) | Canvas, after a question |

### Group "Grading, step by step" (`grading/tasks/`)

| Task | Command | Parameters | Writes to |
|---|---|---|---|
| Download submissions | `python -m grading.tasks.download_data` | `course_id`, `assignment_id` (both required) | `data/` |
| Analyze the assignment | `python -m grading.tasks.analyze_assignment` | `assignment_path` (required, directory); `--points`; `--output` | `data/` |
| Prepare the spreadsheet | `python -m grading.tasks.prepare_spreadsheet` | `assignment_json` (required, file); `--output` | `data/` |
| Prepare the rubrics | `python -m grading.tasks.prepare_rubric` | `assignment_path` (required, directory) | `data/` |
| Collect the grades | `python -m grading.tasks.collect_grades` | `result_dir` (required, directory); `--output`; `--grading-scheme` (file) | `data/` |
| Push the grades | `python -m grading.tasks.push_grades` | `spreadsheet` (required, file); `--assignment-name`; `--dry-run`; `--upload-submissions` | Canvas (grades; with `--upload-submissions`, a comment to each student with their graded files) |

### Group "PUOJ, step by step" (`grading/tasks/puoj/`)

| Task | Command | Parameters | Writes to |
|---|---|---|---|
| Prepare the PUOJ data | `python -m grading.tasks.puoj.prepare_data` | `input_dir` (required, directory); `course_id` (required); `--points`; `--check-history` | `data/` (and, with `--check-history`, `input_dir`) |
| Create the Canvas assignment | `python -m grading.tasks.puoj.create_assignment` | `assignment_path` (required, directory); `--yes`; `--upload-problems` | Canvas (the assignment) |

### Group "Workflows" (`bin/`)

| Workflow | Command | Parameters |
|---|---|---|
| Grade a Canvas assignment | `bin/run_canvas.sh` | `course_id`, `assignment_id`; `--grading-scheme` (file); `--yes`; `--restart` |
| Grade a PUOJ assignment | `bin/run_puoj.sh` | `input_dir` (directory), `course_id`; `--grading-scheme` (file); `--points`; `--check-history`; `--upload-problems`; `--no-upload-submissions` (by default the students' files are attached to a comment); `--yes`; `--restart` |
| Collect grades and attendance report | `bin/report_attendance.py` | `course_info.yaml` (required, file) |

`bin/report_attendance.py` (workflow 1 of `tasks/AGENTS.md`) is run as
`canvas/bin/python bin/report_attendance.py <course_info.yaml>`; it asks no
question, and the path of the chosen course's `course_info.yaml` is filled in.

The grading workflows call homework-grader, which needs `ANTHROPIC_API_KEY`. The
workflows take it from the environment variable, else from
`config/agent_config.py` (`ANTHROPIC_API_KEY`), else ask for it (hidden). The
form has an optional password field for it, prefilled (hidden) from the
environment when it is set; a key typed there is passed through the subprocess's
environment. Left empty, the workflow reads `config/agent_config.py` itself: the
TUI never reads that file. Show that the workflows resume from a checkpoint in
`data/`, and that `--restart` deletes it.

Helper modules without a command line (`html_to_markdown.py`, `markdown_to_html.py`,
`pdf_to_markdown.py`, `utils/`) are not in the catalogue.

