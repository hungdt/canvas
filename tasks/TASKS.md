# AI Agent Instruction

Here are the usual tasks, each task is separate in a file. 
Helper functions can be read in other directories.
Configuration information can also be read in other directories

# Tasks

## Task 1: Collect grades from the in-class Q&A
 - Filename: `collect_oral_grades.py`
 - 3 input arguments:  
   + Spreadsheet ID: required
   + --sheet-name: optional, the sheet name, stored as `sheet_name`, default value: "Vấn đáp"
   + --course-id: optional, Canvas Course ID
   + --assignment-id: optional, Canvas Assignment ID
 - Read sheet `sheet_name` of the input spreadsheet and store from the 2nd row and below in a Pandas dataframe
 - If the Course ID is not given, it is cell A1 of the sheet
 - If the Assignment ID is not given, it is cell C1 of the sheet
 - This dataframe has the columns: `Name`, `ID` (Canvas User ID), `SIS User ID` (Canvas SIS ID), and `N` columns (values are `session_name`) corresponding to the `N` sessions, `Mean` column for the average grade and `Count` for the number of times the scores are given
 - Associate the dates (`session_date`) in the first row of the sheet with the session names (`session_name`) in the `N` session columns
 - Clean the data in the dataframe:
   + if a row that has empty ID or empty SIS User ID, drop it
   + For column `Mean`: set empty for any NaN value
 - Use `SIS User ID` as the index of this dataframe
 - Connect to the assignment with the input Canvas Assignment ID of the course with Canvas Course ID in Canvas website
 - Adjust the maximum points for the assigment to be 3
 - For each student:
   + Update the grade using the one from `Mean`
   + For each session column `n` (`n` from 1 to `N`): 
        + if the value of this column for the student is empty or NA, continue
        + prepare the text "{`session_name`} ({`session_date`}): x điểm" where `x` is the value of this column; 
        + if there is no Assignment Comment with the content exactly the same as this text, add a comment using this text as content

## Task 2: Prepare the Attendance Report and store it in a Google Spreadsheet
 - Filename: `prepare_attendance.py`
 - 1 input arguments: `Google Drive Folder ID`
 - Open the Google Drive folder `Google Drive Folder ID` and name it `root` folder

 - In that folder, read the Google Spreadsheet with the name starting with "Attendance"
 - Use the skill `get-attendance-from-spreadsheet` and the input `Spreadsheet ID`, `Canvas API` and None for `Course ID` to obtain `attendance_df`, Canvas course, _, and `date_to_column`
 
 - `processed_columns` is used to managed the columns that have been processed, in the following loops, if the matched column is in the `process_columns`, simply continue to the next loop

 - For each quiz `Quiz k` (where `k` starts from 1) in the Quizzes
   + Read and store the creation date of the quiz as `quiz_creation_date`
   + Match the creation date to the date of column `matched_col` in `attendance_df`
   + If there is no matching: continue to the next loop
   + Get the list of all students and their `SIS User ID` taking the quiz as `students_taking_quiz`
   + Fill in value "Y" in column `matched_col` at the rows corresponding to the students and their SIS User ID in `students_taking_quiz`
   + Fill in zero in the quiz for students who do not take the quiz

 - For each `BTTLk` (where `k` starts from 1) in Assignment Group `BTTL`
   + If `BTTLk` is unpublished: continue
   + Read and store the creation date and "Available until" date of `BTTLk`
   + If creation date and "Available until" date are not the same: continue
   + Match this date to the date of column `matched_col` in `attendance_df`
   + If it does not match: continue
   + Get the list of all students and their `SIS User ID` making submission as `students_submitting`
   + Fill in value "Y" in column `matched_col` at the rows corresponding to the students and their SIS User ID in `students_submitting`

 - If there is a folder named "PUOJ" inside the `root` folder: 
   + Read all the CSV file inside that folder, each as a Pandas dataframe
   + Use a dictionary named `puoj` to store these dataframes: the key is the filename (without .csv extension, the value is the dataframe)

 - For each `key` and `value` in `puoj`:
   + Extract the date and time in the `key`
   + Match the obtained date to the date of column `matched_col` in `attendance_df`
   + If there is no matching: continue to the next loop
   + Fill in value "Y" in column `matched_col` at the rows corresponding to the students where their SIS User ID matches `Username` column of the `value`

 - Update the columns associated with the dates in the sheet "Attendance" with the values from `attendance_df` 

## Task 3: Update the attendance of a class in Canvas
 - Filename: `update_attendance.py`
 - Input arguments:
   + `Spreadsheet IDs`: required, at least one ID, but can be many, stored in a list `spreadsheet_ids`. The main spreadsheet is the first one in the list
   + --penalty: required float, the deduction point for each absent session, stored as `penalty`
   + --course-id: optional, the Canvas course ID, stored as `course_id`
   + --assignment-id: optional, the Attendance assignment ID in canvas, stored as `assignment_id`

Open sheet "Attendance" of spreadsheet `spreadsheet_ids[0]` as `main_attendance_sheet`
 - If `course_id` is not defined, `course_id` is assigned by cell A1 of `main_attendance_sheet` and open the course
 - If `assignment_id` is not defined, `assignment_id` is assigned by cell C1 of `main_attendance_sheet` and open the assignment as `attendance_assignment`
 - Let `num_days_off` a dictionary with keys are SIS User IDs obtained from `main_attendance_sheet`, the values are integer initialized as 0
 - Check if the students enrolled in the class of `course_id` is the contains those from `num_days_off.keys()` by comparing the SIS User ID
   + Stop running if the check fails

 For each `spreadsheet_id` in spreadsheet_ids:
  - Use the skill `get-attendance-from-spreadsheet` and the input `spreadsheet_id`, `Canvas API`, `course_id` to obtain `attendance_df`, Canvas course, and `date_to_column`
   + Stop running if any of the returning values is empty
  
 - Get the date for today as `today`
 - For each student (or row) in `attendance_df`:
   + If there is no score for this student in `attendance_assignment`, initialize it with value 10.
   + Read the row corresponding to that student
   + For each column corresponding to `date_to_column`
     + If the date in `date_to_column` is after `today`: continue
     + If the cell is empty, "" or "Y": continue
     + Else if the cell is "N":
       + Prepare the text: `comment_str` = "{column name} ({corrresponding date}) vắng mặt"
       + If `comment_str` do not exist in the comments of `attendance_assigment`: 
         + write a new comment with `comment_str` and subtract the score by `penalty`
         + if the score is negative, set it to 0
     + Else: issue a warning that the value in the cell is not valid
     + Count the number of cells with "N" value and add up in `num_days_off` corresponding to the student SIS User ID
  
For each SIS User ID (key) and `n_days_off` (value) in `num_days_off`: If 10 - `penalty`*`n_days_off` != the score of that SIS User ID of `attendance_assignment`, issue a warning of the mismatch.


## Task 4: Gather grades of all assignment groups for finalizing

Collect grades from all Canvas courses related to a main course, reducing them to a single grade for each student and putting into a newly-created assignment in the main course.

 - Filename: `gather_grades.py`
 - Input arguments: `yaml_filename`: required, an YAML file contains information of all assignment groups for gathering grades

Structure of the YAML file `yaml_filename`:
 - `course_ids`: list of all Canvas course IDs, the first item is the main course
 - `destination_group`: the name of the destination assignment group that the summary located
 - `assignment_groups`: information of the assignment groups to be gatherd, a dictionary
   + key: the name of the assignment group
   + value: the weight of the assignment group in percentage
 - Other fields are not used in this task

Read the YAML file and store the information of the YAML as variables `course_ids`, `destination_group`, `assignment_groups`

The main course ID: `course_id` = `course_ids[0]`
Total weight: `total_weight` = 0
Dataframe `grade_df`: 
 - Using SIS User ID of all students in the main course as the index
 - Columns are: "Name", then all keys of `assignment_groups`, then "Final"

Check if `destination_group` does not exist in the main course: stop running

Check the validity of the assignment groups and gather information from them: For each `name` and `weight` as key and value of `assignment_groups`
 - Check if the assignment group `name` exists in any of the courses using `course_ids`. If any fails, stop running.
 - Check if the assignment group `name` occurs more than one: stop running
 - Add the `weight` to `total_weight`
 - Put all scores of that assignment group to the corresponding column in `grade_df` using 0-10 scale. If a student does not have the score of that assignment group, mark the cell NA

Calculate the final grades for column "Final" of `grade_df`: sum(`grade_df[name]` * `weight`) / `total_weight` with `name` and `weight` for each assignment group

Print the `grade_df`

Ask user to proceeding writing `grade_df['Final']` to Canvas. If yes:
 - Create an assignment `assignment` named "Điểm quá trình" inside the assignment group `destination_group` with maximal points 10.
 - Create a rubric for `assignment`
   + Rubric name: the same name as `assignment` name
   + Setting the rubric in Canvas: choose options "Use this rubric for assignment grading"
   + Criteria: 
     + names and maximal points are keys and values/`total_weight` (keys and values of `assignment_groups`)
     + criterion description: "Trọng số {weight}%, trong điểm quá trình chiếm {weight}/{total_weight}"
 - For each row in `grade_df`:
   + If value for "Final" column is NA: skip
   + For each criterion of the rubric: Update the grade of the criterion with `grade_df`[criterion name] / 10 * criterion points
   + Check if the grade of the assignment is the same as the value in the "Final" column: if not, warn and stop running
