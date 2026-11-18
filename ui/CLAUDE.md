# Text User Interface (TUI) for `canvas`

A terminal interface, built with [Textual](https://textual.textualize.io/), to run
every task and workflow of this project without typing command lines: pick a task,
fill in a form, watch its output live, answer its questions, and see the result.

The TUI is a **front end only**. It does not reimplement any task: it builds the
same command line a user would type and runs it as a subprocess.

## Rules

- **Do not modify the code in `tasks/`, `grading/`, `utils/`, `config/` or `bin/`**
  to make the TUI work. If a task cannot be driven from the TUI without a change,
  stop and tell the user what change is needed and why.
- **Do not run any task, workflow or the TUI itself** (`.agents/rules/script-prohibition.md`).
  Most tasks read from and write to Canvas (grades, comments, rubrics, assignments)
  and Google Sheets; the grading workflows also call the Anthropic API, which costs money.
  Running the TUI's own tests is allowed only when the user asks.
- Never show, log or save `CANVAS_API_KEY` (`config/canvas_config.py`) or
  `ANTHROPIC_API_KEY`. The Anthropic key is typed in a password field and only
  passed to the subprocess's environment; left empty, the workflow reads it from
  `config/agent_config.py` itself (the TUI never reads that file).
- Python only, in the virtual environment `canvas` (`canvas/bin/python`). Do not
  install packages yourself: ask the user to install `textual` (and `textual-dev`
  for development).

## Layout

```
ui/
├── CLAUDE.md          this file
├── __init__.py
├── __main__.py        entry point: python -m ui
├── app.py             CanvasApp (the Textual App), key bindings, screen switching
├── catalog.py         description of every task / workflow (see "Task catalogue")
├── runner.py          runs a command as a subprocess and streams its output
├── db.py              SQLite database: schema, migrations, queries (see "Database")
├── importers.py       reads course_info.yaml and graded assignments into the database
├── course_info.py     writes {course_dir}/course_info.yaml from a course of the database
├── last_values.py     last values of each form (~/.config/canvas-ui/last_values.json)
├── screens/
│   ├── home.py        choose (or create) a course, then its tasks and workflows, grouped
│   ├── form.py        form generated from the catalogue entry of a task
│   ├── run.py         live output, prompts, cancel, exit status
│   ├── history.py     past runs of this session (command, status, log)
│   ├── courses.py     Course setting: edit the chosen course, refresh, mark complete, delete
│   ├── new_course.py  new course (Canvas courses and penalty), course summary (read from Canvas)
│   └── grades.py      graded assignments in the database: browse, export
├── widgets/           reusable widgets (path picker, ID input, confirm dialog, …)
├── app.tcss           Textual CSS
└── tests/             tests with Textual's Pilot (App.run_test)
```

Start the TUI from the project directory: `canvas/bin/python -m ui`.

## Design

See [design.md](docs/design.md)

## Task catalogue

See [task_catalogue.md](docs/task_catalogue.md)

## Interactive prompts

See [prompts.md](docs/prompts.md)

## Database

See [database.md](docs/database.md)

## Textual guidelines

See [textual.md](docs/textual.md)

## Tests

Write tests in `ui/tests/` with `pytest` and Textual's `App.run_test()` / `Pilot`:
form generation and validation from the catalogue, the command line built for each
task, prompt detection, the database (schema creation, migrations, imports and
re-imports, generating `course_info.yaml` into a temporary directory and reading it back with `read_config`, on a temporary database file or `:memory:`, with small sample
`grades.xlsx` / `metadata.json` / `course_info.yaml` in `ui/tests/fixtures/` made
of fake students), and the run screen driven by a fake command (e.g. a small
script in `ui/tests/fixtures/` that prints, asks `[y/N]` and exits with a given
status), never by a real task. Ask the user before running the tests.
