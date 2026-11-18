## Interactive prompts

The TUI must be able to answer these questions (see "Running a task"):

| Where | Question |
|---|---|
| `tasks/gather_grades.py` | `Write the Final grades to the assignment … in Canvas? [y/N]` |
| `grading/tasks/puoj/create_assignment.py` (without `--yes`) | `Proceed and push the grades to this assignment? [y/N]` |
| `bin/run_canvas.sh` | course ID and assignment ID when not given; `ANTHROPIC_API_KEY` (hidden) when neither set nor in `config/agent_config.py`; `Rerun this step with a new grading scheme? [y/N]` then `Path of the new grading scheme file:`; `Push these grades to Canvas? [y/N]` |
| `bin/run_puoj.sh` | input directory and course ID when not given; `ANTHROPIC_API_KEY` (hidden) when neither set nor in `config/agent_config.py`; `Press Enter to continue...` (after checking `description.md`); the same grading-scheme question as `run_canvas.sh`; `Continue and write the data to Canvas (create the assignment, push the grades)? [y/N]` (without `--yes`); the push question of `run_canvas.sh` |
| `bin/report_attendance.py` | none (its tasks do not ask either) |

Show long outputs before a question (the grade table of `gather_grades.py`, the
dry run of the workflows) in full, so that the user can read them before answering.

