## Textual guidelines

- Target the Textual version installed in `canvas`; check it
  (`canvas/bin/python -m pip show textual`) before using recent APIs.
- One `Screen` class per screen, pushed with `push_screen`; modal questions are
  `ModalScreen`s that return their answer through `push_screen(..., callback)`
  or `push_screen_wait` inside a worker.
- Long-running work happens in `@work` workers (`exclusive=True` for the task
  runner); update widgets from the worker with `call_from_thread` when the worker is
  a thread, or directly when it is async.
- Style in `app.tcss`, not inline. Support Textual's light and dark themes.
- Key bindings in `BINDINGS`, with descriptions, so that they are shown in the
  `Footer`: `q` quit (asks if a task is running), `escape` back, `ctrl+r` run,
  `h` history.
- The interface text is in English; Vietnamese text from the tasks (sheet names,
  comments, assignment names) is shown unchanged.

