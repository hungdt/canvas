# Instruction for grading-related tasks

Here are the tasks that are used as helpers for the grading process.

# Tasks

## Task 1: Create directories and download data

 - Filename: `download_data.py`
 - Input arguments: `Course ID` and `Assignment ID`
   + `Course ID`: Canvas course ID, required
   + `Assignment ID`: Canvas assignment ID, required

 - Use the configuration information in `config/` to get the Canvas API
 - Read the course and the assignment
 - Create a directory using the name of the course in `data/`, the path of this directory is stored in `course_dir`
 - Create a directory using the name of the assignment in `course_dir`, the path of this directory is stored in `assignment_dir`
 - Create a directory named "submissions" inside `assignment_dir`, the path of this directory is stored in `submissions_dir`

 - Download all submissions of the assignment and place them in `submissions_dir`:
   + All files are in the same `submission_dir`
   + Filename format: <Name>_<SIS User ID>_<Original filename>
   + No space allowed in the filename, space is replaced by '_'

 - Collect the assignment description only (HTML). If there is any image or figure used in the HTML file, get its URL link and put into the text together with the caption
 - Convert this assignment description to Markdown format "description.md" and place it in `assignment_dir`, maintaining the URL links for the images and converting mathematical formulas into Latex forms, text subscript or superscript must also be converted to Latex forms.
 - Create a JSON file `metadata.json` to store:
   + The maximal point of the assignment
   + Course name
   + Course ID
   + Assignment name
   + Assignment ID
   + Assignment Creation Date, Assignment Due Date, Assignment Until Date

## Task 2: Analyze the assignment

 - Filename: `analyze_assignment.py`
 - Input arguments: 
   + `assignment_path`: required, store the path to the assignment HTML file
   + `--points`: optional, the maximal point of the problem
   + `--output`: optional, the name of the JSON file for the output of the analysis, default value is "assignment.json"

 - If `--points` is missing, the maximal point is obtained from `metadata.json`. If there is no `metadata.json`, stop running the code.
 - Break the assignment content into smaller parts:
   + If there are `N` matches for regex patterns `^Câu +(\d+) +\((\d+) điểm\)`, there are `N` problems. If no such pattern found, there is only 1 problem.
   + Store data for the problems in a dictionary `assigment_dict`
   + Collect the point for each problem: problem number is the first group of the pattern, the point is from the second group. If there is only 1 problem, the point for the problem is the whole point assigned to that problem
   + Collect the `preamble` (if there is) from the beginning to the line before `Câu 1` and store in `assignment_dict` with the key `preamble`
   + Collect the text for each problem: 
     + the text for a problem `k` is defined from `^Câu +{k} +\(\d+ điểm\)` to the line before `^Câu +{k} +\(\d+ điểm\)`. If there is only 1 problem, the text is the whole text
     + the parts of the problem are separate by the regex `^[a-z]) +\(\d+ điểm\)`
     + store the text and point for each part in `assignment_dict`: if the second group for the point does not exist, the point is the average point for each part
     + store the preamble of the problem


 - Store the whole point into the dictionary `assignment_dict`
 - Store the dictionary `assignment_dict` as a JSON file, its name is from the input argument. If the input `--output` has only the file name, then the output directory is the same as the `assigment_path`

## Task 3: Prepare the Excel file to store the grades

 - Filename: `prepare_spreadsheet.py`
 - Input arguments:
   + `assignment_json`: required, path to the assignment JSON file obtained from analyzing the assignment content. If it is a directory, append the default filename "assignment.json"
   + `--output`: optional, filepath of the output spreadsheet (Excel XLSX format), default value is "grades.xlsx" at the same folder as the JSON file. The code must validate the file extension to ensure XLSX format.

 - Read data from `assignment_json` into the dictionary `assignment_dict`
 - Check if `metadata.json` exists in the same directory as `assignment_json`, if it does not, stop running, else read `metadata.json` into the dictionary `assignment_info`
 - Check the directory "submissions", if it does not exist, stop running
 - Check the XLSX file: The code fails if the file exists, otherwise creates the file
 - Create a worksheet with the name of the assignment
 - In the first row of the worksheet, write:
   + A1: course name, B1: course ID
   + C1: assignment name, D1: assignment ID
   + E1: assignment creation date, F1: assignment due date, G1: assignment until date
 - In the second row of the worksheet, prepare the columns for:
   + A: student name
   + B: SIS User ID
   + C: Total grade
   + D: Total modified grade
   + If Problem 1 has non empty parts: D: Submission 1, E: 1a ({points for 1a}), F: 1a mod ({points for 1a}), G: Comment 1a, H: 1b ({points for 1b}), I: 1b mod ({points for 1b}), K: Comment 1b, ... 
   + Else if Problem has empty parts: D: Submission 1, E: 1 ({points for Problem 1}), F: 1 mod ({points for Problem 1}), G: Comment 1
   + Similar for Problem 2, Problem 3 ...

 - Inside the "submission" directory, create directory "problem_{n}/raw" for `n` is the problem number for all the problems

 - For each file in in "submissions" directory
   + Assign the file name to `submission_filename`
   + From the file name, obtain the student name, SIS User ID, and the `original_filename` using the regex pattern "(.*?)_(\d+)_(.*)\..*" (group 1: student name, group 2: SIS User ID, group 3: `original_filename`)
   + Write the student name, SIS User ID if the corresponding cells are empty
   + Examine the `original_filename` to detect which problem it is:
     + If there is only 1 problem: `problem_number` = 1
     + If there are more than 1 problems: 
       + based on the regex pattern "cau.*(\d)", "bai.*(\d)", "câu.*(\d)", "bài.*(\d)", the first digit in this pattern (group 1 of these regex) determines the problem number; then store the problem number in `problem_number`
       + if `problem_number` is not yet determined: get the first digit from `original_filename` and assign it to `problem_number`
   + If it is still unable to determine `problem_number`: log then continue
   + If the column "Submission {`problem_number`}" is empty or "":
     + write the file name to the corresponding cell 
     + move the this file to the directory "problem_{`problem_number`}/raw"

 - Resize all the columns in use with optimal width
 - Save the XLSX file and exit with success

## Task 4: Prepare the rubrics for grading the assignment
 - Filename: `prepare_rubric.py`
 - Input arguments: `assignment_path`: required, the path to the directory containing `assignment.json` and `metadata.json`
 - For each problem `k` in `problems` of `assignment.json`
   + Copy the templates `grading/rubric.yaml.tmpl` into the directory "{`assignment_path`}/submissions/problem_{`k`}" with the new name `rubric.yaml`, if this file exists, overwrite it.
   + Replace all possible placeholders denoted by {{}} with all possibile parameters from `assignment.json` and `metadata.json`
   + For the criteria section of `rubric.yaml`: use the first criterion as the template
     + If `problems.parts` is empty: create a criterion with criteria="problem_{`k`}" in `rubric.yaml` with criterion_name="Problem {`k`}", criterion_weight=1.0, problem_description=problem.preamble
     + Else: For each part `p` in `problems['parts']`: create a criterion with criteria="problem_{`k`}{`p`}" in `rubric.yaml` with criterion_name="Problem {`k`} - Part {`p`}", criterion_weight=part.points / problem.point, problem_description=problem.preamble + '\n' + part.text
   + Save `rubric.yaml`

## Task 5: Collect grades
 - Filename: `collect_grades.py`
 - Input arguments:
   + `--output`: optional, filepath of the output spreadsheet (Excel XLSX format), default value is "grades.xlsx" at the same folder as the JSON file. The code must validate the file extension to ensure XLSX format.
   + `result_dir`: required, the path for the data directory from homework-grader
   + `--grading-scheme`: optional, a text file for grading scheme in use. The text file contains a 2D array with 3 columns (ignore the comments): 1st and 2nd columns define the range of the input score in percentage, the 3rd column is the output score in percentage. If the score is in a range defined by 1st and 2nd columns, its final score is defined by the 3rd column. If an input score is not in any range, the output score is unchanged.

 - If "grading_scheme.txt" exists in the same directory as spreadsheet, use it for grading scheme.
 - Open the spreadsheet 
 - For each "problem_`k`" in "{`result_dir`}/submissions"
   + problem number = k
   + `problem_dir` = "{`result_dir`}/submissions/problem_`k`"
   + Open YAML file "{`problem_dir`}/rubric.yaml", read the `assignment_name` from "rubric.course.assignment" of this YAML file. If the YAML file is not found or unable to read: continue.
   + Check if the spreadsheet has the sheet named `assignment_name` and assign the sheet to `sheet`. Otherwise, continue.
   + If "{`problem_dir`}/scores" does not exist: continue
   + For each JSON file in directory "{`problem_dir`}/scores"
    + Open this JSON file as a dictionary `scores`
    + Open the file with the same filename in directory "{`problem_dir`}/ir", read the item "source_files" to `source_files`
    + Get `original_filename` = basename of `source_files[0]` and determine the cell of `sheet` corresponding to `original_filename` 
    + For each `criterion` in `scores["dimension_scores"]`
      + Obtain from `criterion_name` the problem number and the part: e.g. "Problem 3 - Part a" means problem number is 3 and part is "a". There are cases where there is no part.
      + The original score is `score = criterion["score"] / 5`. If the grading scheme exists, `mod_score` is the converted score based on the grading scheme, else `mod_score` equals `score`
      + Based on the problem number and the part: 
         + Locate the column where column name without "mod" for writing `score * max_grade` and column where column name with "mod" for writing `mod_score * max_grade`.  Here `max_grade` is determined as the value inside "()" in the column name.
         + Locate the column for writing the comment `"Minh chứng: " +  criterion["evidence"] + "\nNhận xét: " + criterion["reasoning"] + "\nCải tiến: " + criterion["improvement"]`
 - Get the sum of all original grades in the sheet and put into the column "Total grade"
 - Get the sum of all modified grades in the sheet and put into the column "Total modified grade"
 - If the grading scheme is defined, copy the grading scheme file to the same directory as the spreadsheet with name "grading_scheme.txt"
 - Save the spreadsheet
 - For each problem:
   + Get the modified grade of the problem (sum of modified grades of all parts)
   + Calculate descriptive statistics: Mean, median and standard deviation
   + Plot the histogram of the grades directly on the terminal using plotext
 - Do the same thing for the total modified grades (calculate descriptive statistics and plot histogram)
 
## Task 6: Push the grades to Canvas
 - Filename: `push_grades.py`
 - Input arguments: 
   + `spreadsheet`: required, the XLSX file containing grade data
   + `--assignment-name`: optional, the name of the assignment, the default value is the name of the first sheet in `spreadsheet`
   + `--upload-submissions`: optional, default value: not defined, a flag to determine whether to upload students' submissions to the assignment

 - Open the spreadsheet, choose the sheet with name `assignment_name` and collect the data:
   + The first row: A1: course_name, B1: course_id, C1: assignment_name_reference, D1: assignment_id
   + From the second row and below: put in a Pandas dataframe `grades`
 - Compare `assignment_name_reference` with the input `assignment_name`: if they are different, stop running
 - Open Canvas course isomg `course_id`
 - Open the assignment in Canvas using `assignment_id`
 - If there is no rubric in the assignment, create a new rubric
   + Title of the rubric is the same as the `assignment_name`
   + Setting the rubric in Canvas: choose options "I'll write free-form comments when assessing students" and "Use this rubric for assignment grading"
   + The criteria are created based on the problem number or problem number with part of column names of `grades`
   + The maximal points for each criterion is obtained from the number inside "()" next to the criterion name in the column names of `grades`
 - If the rubric already exists: check if it has the same structure as described for creating a new one; if not, stop running

 - Upload the submissions:
   + Check input argument `--upload-submissions`, if not defined, continue to updating the grades
   + For each "SIS User ID" in the spreadsheet: 
     + create a comment to the student in this Canvas assignment
     + comment's content: "Các file đã đuợc dùng để chấm điểm đuợc gắn trong comment này."
     + attach all files in "*/submissions/problem*/raw" which contain the current "SIS User ID" to this comment. Note that there can be more than one "submissions" directory the assignment directory. (The assignment directory contains the XLSX file)
 
 - Update the grades: For each row of `grades`
   + Use the modified grades
   + If the grade is not available in the row: continue
   + Update the grade and comment for each criterion to the rubric for students with Canvas's SIS User ID matches the "SIS User ID" in `grades`

 - Students who do not have submissions have zero grade.

  