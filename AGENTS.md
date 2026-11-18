# AI Agent Instruction

`canvas` is a set of tools that I use to manage the grades, attendance reports, etc. in Canvas

## General discussion

In `canvas`, I will various tools, each for a task on Canvas, a learning management system. I will gradually add tasks into this instruction whenever I have a new idea or a new demand of work.

## Structure of the package

There are three main directories

 - `config`: information for accessing Canvas, information for accessing Google Drive, etc. and the saved configuration
 - `utils`: a set of helper files for shared classes and functions
 - `tasks`: files, each corresponds to a specific task interacting with Canvas

## Preparation

 - Python will be employed for this work.
 - Create a virtual environment named `canvas` for running the code.
 - Install packages: `canvasapi`, `google-api-python-client` and packages to read data from Google Spreadsheet

## Code

 - Read tasks/AGENTS.md to generate codes for each task

## Grading
 
 - Workflow for grading assignments is described in `grading/AGENTS.md`. Do not run it without user's permission.