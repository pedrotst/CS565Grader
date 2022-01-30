# CS565Grader

This is a tool for grading Coq homework for CS565, Purdue's graduate programming
language course. To use this tool, we also need to develop grading script (in
Coq, mostly Ltac) for each homework.

Currently it only supports BrightSpace (hard-coded), the learning platform
Purdue uses.

Unfortunately, the code has very few comments (my bad) at the moment.

## Workflow

This section outlines the workflow of using this tool for CS565. I assume you
have installed `stack` and know how to use it to build and run Haskell programs.
I will not talk about how to write grading scripts in this section, and I will
explain that in details in section (Grading scripts).
- Distribute homework: a homework typically consists of a homework file with
  exercises, a testing file for sanity check, a `Makefile` and `_CoqProject`
  file for building, and auxiliar files that the homework imports. Although the
  testing file is somewhat similar to the testing file used for grading, make
  sure to not reveal any answers in this testing file. See [an example
  homework](/example/for_student).

- Export grade book: go to BrightSpace, select the `Grades` tab, and click the
  `Export` button. Make sure we select the following options:
  + Key field: Org Defined ID
  + Grade values: points grade
  + User details: last name and first name
  
  Then select the homework we want to grade. Do not select more than one.
  Finally click `Export to CSV` to download a CSV file. We will call this
  `gradebook` file.
  
- Download submitted homework: go to BrightSpace, select `Course Tools` and then
  `Assignments`. Click the homework we want to grade. In the submission page, go
  to the buttom of this page and select `200 per pages`. Blame BrightSpace for
  not having a `download all` or `select all` button, and pray that you don't
  have more than 200 submissions. Now click the `select all rows` checkbox and
  then click `Download` button to download a zip file of all submissions.
  
- Grader file structure: See [an example grader directory](/example/for_grader).
  This directory, called `top` directory, consists of 3 directories: `input`,
  `output` and `build`. `input` directory contains students' homework, gradebook
  and our grading scripts. `output` directory will host the updated gradebook
  and feedback files for students after we finish grading. `build` directory is
  the working directory where we build and test homework.

- Create the `input` directory: copy the gradebook file to
  `input/gradebook.csv`. Unzip the submission zip file to a subdirectory called
  `hw`. This `hw` directory should have:
  + `index.html` for whatever reason
  + directory per submission: each directory should contain only one file, that
    is the homework file, usually called `hw<number>.v`. The directory name
    should have the format `<some id> - <last name> <first name> - <timestamp>`.
    There might be multiple submissions from the same student, and the grader
    will pick the latest one using the timestamp. However, BrightSpace
    constantly changes their timestamp format (and the format of this directory
    name) for no reason (they changed 3 times in one semester!). In that case,
    the grader will throw an error (hopefully!), and you will have to update the
    corresponding parser (`loadStudentDirs` in `Lib.hs`).

  We also need to create a directory `aux`, consisting of grading scripts. I
  will talk about this in details later, but now you just need to know the
  grading scripts will produce a log file.
  
- Prepare the build directory: run `stack run -- prepare <top directory>
  <homework file>`. For example, `stack run -- prepare for_grader hw.v` if you
  try out this using the [example grader homework](/example/for_grader). Check
  the output of this command and see if there is anything weird, e.g., you may
  check if it indeed copies the latest submission if a student has multiple
  submissions. At the end of the output, it should say "All done!!".
  
  Also check the generated `build` directory (e.g., `for_grader/build`). It
  should consist of directories indexed by PUID, each of which contains the
  student's homework and the grading scripts from `aux`. You should try not to
  map these PUID to the students' names, to minimize bias.

- Build and grade homework: run `stack run -- grade <top directory>`. For
  example, `stack run -- grade for_grader`. It is possible that the compilation
  fails for some submissions (or even loops forever if they submitted bad proof
  scripts!). Go to that student's build directory and manually fix the issues.
  If it's their faults, edit the file `local.v` and modify `Comment` and
  `Penalty` to reflect that. More on that in section (Grading scripts).
  
- Manually grade some exercises: ideally the `grade` command finishes all the
  grading, but in most cases we need to manually grade some exercises. It is
  either because they are open questions, or because we need to give partial
  credits. The grader allows us to write some automation (called sound verifier
  and sound falsifier) to decide if this student gets the answers correct (or
  incorrect) incompletely. Although a good automation can greatly reduce the
  number of exercises that require manual grading, we still need to manually
  grade some answers if the automation fails.
  
  After running the `grade` command, it should output a list of exercises and
  student IDs that requires manual grading (in the form of `<exercise name>:
  <student IDs>`). Manually process those exercises and update the scores in the
  corresponding `local.v`. You can also update the automation in testing scripts
  to cover more cases. I will talk more about it later, but the key is to
  balance efforts spending on automation and efforts spending on manual grading.
  Strengthening the automation might take more work than manual grading if we
  overdo it. But it can save us a lot of efforts when it's done right.
  
  Re-run `grade` command until we finish all manual grading and it should output
  "All done!!". Usually I like to run `stack run -- grade <top directory> -f`
  one last time just to make sure nothing weird happens (`-f` will run `make
  clean` first).
  
- Generate `output` directory: run `stack run -- publish <top directory>`. The
  generated `output` directory should contain an updated `gradebook.csv` and a
  `feedback` directory which consists of `feedback.txt` for each student. Check
  some of these files just to make sure nothing weird happened. Note that some
  students might get more points than maximum points due to bonus questions.
  
- Publish feedback and grades: go to `output/feedback`, and run `zip -r
  feedback.zip . -x ".*"` (on Mac, you may run `zip -r feedback.zip . -x ".*" -x
  "__MACOSX"`). This command is part of `Info-ZIP` that should be available in
  most UNIX-like systems by default (e.g., MacOS and Linux). DO NOT use other
  tools, whether or not they come with your system. I know it's easy to create a
  zip file from GUI, but it will most likely not work! Blame BrightSpace for
  that.
  
  Go to BrightSpace, and select `Grades` tab. Click the drop-down menu of the
  grade category (the first line of the table header) and select `Edit`. Find
  the option `Allow category grade to exceed category weight` in the `Edit` page
  and consider enabling it if you want the bonus points of this homework/exam to
  compensate their overall grade (I enabled it when I taught the course, but
  please discuss it with the instructor!). Go back to the `Grades` page. Click
  the drop-down menu of the particular homework/exam (the second line of the
  table header) and select `Edit`. Find the option `Can Exceed` and consider
  enabling it (to compensate the grade in this category with bonus points).
  
  Go back to the `Grades` page, click `Import` button, and import the grades
  using `gradebook.csv` in `output` directory.
  
  Go to `Course Tools` tab and click `Assignments`. After selecting the
  corresponding homework, click `Add Feedback Files` on top, and upload
  `feedback.zip`. You should check some students and see if `feedback.txt`
  appears as attachments. Finally, click the `select all rows` checkbox and then
  click `Publish Feedback`.
  
  Be careful that the homework category and homework name should not be the
  same! You can check that by going to `Grades` tab: the first line of the
  header is category and the second line is homework names. Importing grades or
  uploading feedback files could trigger yet another bug that prevents you from
  doing so if they use the same name. Again, blame BrightSpace.

- Inform the students: we have finished the grading! Announce it on whatever
  platform we decide to use (e.g., Piazza).

## Commands

This section documents the commands this grader provides. We invoke the commands
by `stack run -- <command> <arguments>`.

- `prepare <top directory> <homework file>`: create `build` directory from
  `input` directory. E.g., `prepare for_grader hw.v`. One important note is that
  this command does not overwrite the homework file and `local.v` (actually all
  files end with `local`) if they are already present in `build` directory, so
  we can safely rerun this command when we make updates to the grading scripts.
- `grade <top directory> [-f] [student IDs]`: build and grade homework. It
  optionally takes a list of student IDs and only builds those. If `-f` is
  specified, run `make clean` first.
- `publish <top directory>`: create `output` directory with feedbacks and
  updated gradebook.
- `parse <log file>`: parse a log file, grade it and print out the result. The
  log file is generated by running the grading script (e.g., `make -s > log`).
  This command is mostly for debugging.
- `one <homework directory>`: grade a single directory outside of `top
  directory`. This homework directory should follow the same structure as the
  directory `<top directory>/build/<student ID>`, with scripts from `aux` and
  the student homework file. This bypasses the gradebook and other BrightSpace
  related processing. This command is used when you have to deal with a late
  submission (e.g., the student was sick) or other situations where you get the
  homework file in a different way (e.g., from email). After running this
  command, you can send the student the generated `feedback` file and the score
  is also in that file.

## Grading scripts

While the grader provides a framework for grading CS565 homeworks, it does not
know how to do the actual grading. We need to develop grading scripts for each
homework. Grading scripts reside in `<top directory>/input/aux` and will be
copied to the `build` directory. See [an
example](/example/for_grader/input/aux), and their inline comments to understand
how to write them.

This repository also provides [templates](/aux) of the grading scripts.

The grading scripts (`aux` diretory) consist of the following files:
- `Makefile` and `_CoqProject`: you can use `Makefile` as it is from
  [templates](/aux). But you need to edit `_CoqProject` to modify or add file
  names.
- `graderlib.v`: a small library for writing the test script. You should use the
  one from [template](/aux).
- auxiliary files that the homework imports (but they should not submit).
- `local.v`: this file contains comments for students, penalty, manually graded
  scores and other code that is specific to a student. See the inline comments
  in the [example](/example/for_grader/input/aux/local.v) for usage. This file
  will not be overwritten even when you run `prepare`, so we don't have worry
  about these information, like manually graded scores, getting lost.
- `test.v`: the testing script in Coq (mostly Ltac). This is the most important
  file that we need to take time developing. The output will be redirected to a
  log file and used by the grader. It contains automation checking students'
  answers, sanity checks and options like what axioms they are allowed to use.
  See the inline comments in the [example](/example/for_grader/input/aux/test.v)
  for more instructions.

## Developing test scripts

Here I share my workflow of writing test scripts. You may have your own
workflow.

- First I use the solution file (with all correct answers, provided by the
  instructor) to develop the first version of the test script. In this
  iteration, I make sure the automation is good enough that it requires no
  manual grading for this perfect homework.
- Then I improve the test script against a blank homework file. It should
  automatically give the blank homework zero (sometimes it's not possible
  because we may need to give partial credits).
- When we start grading real students' homework, there will be a lot of
  exercises requiring manual grading. I check one such answer and ask myself
  this question: is this answer very exotic? How many other students might have
  answers similar to this? If it is not likely, I give it a manual grade and
  check the next submission. But if it seems like a common answer, I update
  `test.v` (interactively) so that it can automatically grade this answer too.
  Then copy `test.v` to `input/aux`, and rerun `prepare` and `grade`. It also
  feels great when you add a snippet of automation and knock down half of the
  exercises that you need to manually grade!
- Repeat this process of manual grading and script enhancing, until all
  exercises are graded.

## TODO

- The design of the `dependencies` option is flawed. See the inline comments in
  the [example](/example/for_grader/input/aux/test.v). I know how to fix it, but
  haven't got the chance to do it yet.
