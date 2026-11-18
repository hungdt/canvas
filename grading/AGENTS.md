# Grading Instruction

## Tasks

The tasks are listed in the directory `tasks`.

## Grading workflow

Our goal is that by giving the course ID and assignment ID in Canvas, all the submissions are graded and the results are posted on Canvas.

Optional input is the grading scheme, which is used when collecting the grades (Step 8).

Workflow script name: `$project_dir/bin/run_canvas.sh`

In details, the workflow is as follows

1. Collect course ID as `course_id` and assignment ID as `assignment_id` from user

2. Run `download_data.py` with `course_id` and `assignment_id` as inputs to download the submission to local directory

3. Extract the path of the local directory for the assignment and store in `assignment_path`

4. Run `analyze_assignment.py` with input `assignment_path` to analyze and separate different problems and parts in the problem set

5. Run `prepare_spreadsheet.py` with input `assignment_path` to prepare for the spreadsheet for the grades

6. Run `prepare_rubric.py` with input `assignment_path` to prepare the rubrics for grading each problem, which are required for `homework-grader`

7. Run `homework-grader` for grading: To do that, for each problem:
 - Change to the directory for that problem
 - Print the full command and run `preprocess.py` to prepare the "ir" directory
 - Print the full command and run `batch_score.py` using the given rubric for grading

8. Collect the resulted grades and save it into the spreadsheet by running `collect_grades.py`. Ask if the user wants to rerun Step 8 with a new grading scheme; if Yes, prompt the user to provide the path for the new grading scheme file and rerun Step 8.

9. Run `push_grades.py` to push the grades in the spreadsheet into Canvas

If any step of this workflow fails, stop the workflow immediately.

Highlight each step with colors so that one can follow.

Raise exception if there are 10 problems or above.