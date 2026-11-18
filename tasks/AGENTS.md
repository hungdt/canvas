# AI Agent Instruction

This is for the usual tasks, mostly about checking for attendance and collect grades without AI decisions.

# Tasks

The tasks are described in the file "TASKS.md"

# Workflows:

## 1. Collect grades and attendance report

A bash script is created to collect the in-class Q&A grades and attendance report in order to push into Canvas.

Workflow script name: `$project_dir/bin/report_attendance.py`

Input: path for the file "course_info.yaml" (required)

1. Read the YAML file "course_info.yaml" for the information of the course and store in `course`

2. Search all Canvas courses in `course["course_ids"]` for an assignment named "Vấn đáp", store that assignment ID in `oral_assignment_id` and the corresponding course ID as `oral_course_id`

2. For each `n`, `course_id` in `enumerate(course["course_ids"])`:
 - If the spreadsheet with ID `course["spreadsheet_ids"][n]` has a sheet named "Vấn đáp": run "collect_oral_grades.py" with input arguments:
   + spreadsheet ID `course["spreadsheet_ids"][n]`
   + --course-id `oral_course_id`
   + --assignment-id `oral_assignment_id`
 - Run "prepare_attendance.py" with input `course["directory_ids"][n]`

3. Set a dictionary `attendance_ids`:
 - For each `n`, `course_id` in `enumerate(course["course_ids"])`:
   + Search for assignment named "Điểm điều kiện thi KTHP"
   + Get the `assignment_id` for this assignment if the assignment exists else -1
   + `attendance_ids[course_id] = [course["spreadsheet_ids"[n], course_id, assignment_id]`
   + Look for a positive `positive_assignment_id` from key `positive_course_id` in `attendance_ids`, assign any value with negative `assignment_id` in `attendance_ids` with the value  `[spreadsheet_id (unchanged), positive_course_id, positive_assignment_id]`

4. Run "update_attendance.py":
- For each `key`, `value` in `attendance_ids`: run "update_attendance"
   + Input arguments: 
     + spreadsheet_id = value[0]
     + penalty = course["penalty"]
     + course-id = value[1]
     + assignment-id = value[2]
   + Run "update_attendance.py" with these input arguments

