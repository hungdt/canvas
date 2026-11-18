# Conduct grading with raw input

Given the problem in PDF and all the submissions, conduct grading, then creating assignment in Canvas and push all grades to the newly created assignment.

# Details

Input arguments:
 - Directory for problem description ("raw.pdf") and submissions, each of which is a directory with name "SIS User ID" and the submission inside. The directory "$History" should be neglected.
 - Grading scheme: optional, the grading scheme file
 - Course ID for the corresponding Canvas course

Carry out similar workflow as @grading/AGENTS.md

Workflow script name: `$project_dir/bin/run_puoj.sh`

1. Prepare the directory and data:
 - Create directory using course name inside `data/`
 - Create subdirectory with the same name as the input directory, which is also the assignment name
 - Create the directory "submissions" inside this subdirectory
 - Check file "problem_mapping.txt" in the input directory:
   + If it does not exist: create default Pandas dataframe `problem_mapping`: 
        {num: [1], points: [10], prefix: ["*"], PDFfile: ["raw.pdf"]}
   + Else read the file into `problem_mapping` dataframe (the first line which is a comment line contains the column names of the dataframe)
 - Create the file "description.md" (Markdown format) in this subdirectory
 - For each problem `k` (each line) in `problem_mapping`
   + If "PDFfile" column exists in `problem_mapping`: 
     + Append "Câu `num[k]` (`points[k]` điểm) into "description.md"
     + Append the content of the PDF file `PDFfile[k]` into "description.md"
   + Else: We assume that the whole PDF file is in "raw.pdf" and convert "raw.pdf" into 'description.md"
   + Note: ensure maintaining the URL links for the images and converting mathematical formulas into Latex forms, text subscript or superscript must also be converted to Latex forms.

 - For each directory with name as SIS User ID in the input directory:
   + Check the input arguments of the Python script for the argument '--check-history'
   + If there is '--check-history' and if there is subdirectory "$History":
     + For each file `fn` in subdirectory "$History":
       + Only consider `fn` with the structure "{prefix}_{file_id}.{extension}": read `prefix`, `file_id` and `extension`
       + move the file "{prefix}.{extension}" from the parent directory to "$History"
       + For each `prefix`: find the file `fn` with the largest `file_id`, copy this file to the parent directory, keep the filename intact
   + Ignore the subdirectory "$History"
   + Copy all files to "submissions" directory, prepending the name with student's name, the SIS User ID obtained from the name of the directory and "cau{`num[k]`}" where `prefix[k]` matches the original file name (`num` and `prefix` are columns of `problem_mapping`)

 - Pause and ask the user to check and modify "description.md" before continue.

2. Analyze and separate different problems and parts in the problem set from "description.md".

3. Prepare for the spreadsheet for the grades

4. Prepare the rubrics for grading each problem, which are required for `homework-grader`

5. Run `homework-grader` for grading: To do that, for each problem:
 - Change to the directory for that problem
 - Print the full command and run `preprocess.py` to prepare the "ir" directory
 - Print the full command and run `batch_score.py` using the given rubric for grading

6. Collect the resulted grades and save it into the spreadsheet

7. Ask user if they want to continue writing data to Canvas. If not, stop here.

8. Create the assignment in Canvas with the assignment name obtained from the name of the subdirectory of the course in `data/`. 
 - If the assignment exists, ask user if they want to proceed; 
 - Else: If the input argument "--upload-problems" sets: upload the content of "description.md" as the content of the assignment.

9. Push the grades in the spreadsheet into Canvas. Students who do not have submissions have zero grade.

Reuse the code in `grading/tasks` as much as possible

Do not modify the code in `grading/tasks`

Raise exception if there are 10 problems or above.

