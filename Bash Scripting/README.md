# TryHackMe: Bash Scripting - Complete Study Guide
### Task 1: Introduction
Core Concepts & Fundamentals
- Definition: Bash (Bourne Again SHell) is a Unix shell and command-line scripting language executed within terminal environments across Linux distributions and macOS.
- Purpose: Shell scripts chain sequence commands inside an executable file to automate repetitive system administration workflows (e.g., automated backups, system monitoring).
- Key Topics Covered:
  - Syntax structure and formatting
  - Local and global variables
  - Command-line parameters
  - Array manipulation
  - Conditional logic and control structures

### Task 2: First Simple Bash Scripts
#### File Shebang & Script Execution
Every Bash script requires a shebang line as the very first line of code. This header tells the operating system's process loader which interpreter to run when executing the file directly:
```bash
#!/bin/bash
```
Basic Execution Flow
1. Writing Script Commands: Combine standard Linux utility commands alongside standard shell echo statements inside ```.sh``` source files.
```bash
#!/bin/bash

echo "Hello World!"
whoami
id
```
2. Granting Execution Permissions: By default, new files do not have executable file bits set. Update file modes using ```chmod```:
```bash
chmod +x script.sh
```
3. Executing File: Run the file relative to the current directory path:
```bash
./script.sh
```
#### Task Questions & Answers
- Question 1: What piece of code can we insert at the start of a line to comment out our code?
  - Answer: ```#```
- Question 2: What will the following script output to the screen: ```echo "BishBashBosh"```?
  - Answer: ```BishBashBosh```

### Task 3: Variables
#### Variable Declaration & Reference Syntax
Variables allow dynamic storing, updating, and reusing of arbitrary text strings or numeric data across scripts.
- Declaration Rules: Variable assignment requires no spaces around the equals sign (```=```).
```bash
# Correct Assignment
name="Jammy"

# Incorrect Assignment (Triggers Command Not Found Errors)
name = "Jammy"
```
- Variable dereferencing: Prefix the variable identifier with a dollar sign (```$```) to reference or interpolate its value:
```bash
name="Jammy"
echo $name
```
- Output: ```Jammy```

#### Script Debugging Modes
Bash includes built-in execution flags to step through scripts and pinpoint syntax or logic errors:
- Whole-Script Execution Debugging: Run the script directly through the ```bash``` binary with the ```-x``` flag.
```bash
bash -x ./file.sh
```
  - Output Behavior: Lines preceded by ```+``` indicate the command being executed, followed immediately by its runtime output.
- Targeted Section Debugging: Wrap specific blocks of code inside the script to limit debug output:
```bash
echo "Start of script"
set -x  # Enables debugging
# Code in this block is traced during execution
set +x  # Disables debugging
```
#### Using Multiple Variables
Multiple variables can be interpolated into a single string or command output:
```bash
name="Jammy"
age=21
echo "$name is $age years old"
```
Task Questions & Answers
- Question 1: What would this code return? (```name="Jammy", age=21, echo "$name is $age years old"```)
  - Answer: ```Jammy is 21 years old```
- Question 2: How would you print out the city to the screen? (```city="Paris"```)
  - Answer: ```echo $city```
- Question 3: How would you print out the country to the screen? (```country="France"```)
  - Answer: ```echo $country```

### Task 4: Parameters
#### Positional Parameters
Positional parameters pass dynamic arguments into a script at execution time from the command line interface.
- Syntax & Indexing:
  - $0: Represents the filename of the current script.
  - $1 to $9: Represents positional arguments passed in order.
  - $#: Holds the total count of positional arguments passed to the script.
#### Example Execution:
```bash
# Script content (example.sh):
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "Total arguments: $#"
```
```bash
./example.sh Alex Tony
```
- Output:$0 $\rightarrow$ ./example.sh
- $1 $\rightarrow$ Alex
- $2 $\rightarrow$ Tony
- $# $\rightarrow$ 2

#### Interactive User Input (read)
To prompt users for input interactively during runtime instead of requiring command-line flags, use the ```read``` builtin:
```bash
#!/bin/bash
echo "Enter your name"
read test
echo "Your name is $test"
```
#### Task 
- Questions & AnswersQuestion 1: How can we get the number of arguments supplied to a script?
  - Answer: $#
- Question 2: How can we get the filename of our current script (aka our first argument)?
  - Answer: $0
- Question 3: How can we get the 4th argument supplied to the script?
  - Answer: $4
- Question 4: If a script asks us for input how can we direct our input into a variable called 'test' using "read"?
  - Answer: read test
- Question 5: What will the output of echo $1 $3 if the script was ran with ./script.sh hello hola aloha?
  - Answer: hello aloha

### Task 5: Arrays
#### Array Declaration & Indexing
Arrays store indexed collections of elements inside a single variable identifier. Bash arrays use zero-based indexing ($0, 1, 2, \dots$).
- Declaration Syntax: Wrap space-delimited elements inside parentheses:
```bash
transport=('car' 'train' 'bike' 'bus')
```
| Item   | Index Position  |
|:--     |:--              |
| car    | 0               |
| train  | 1               |
| bike   | 2               |
| bus    | 3               |

#### Accessing & Modifying Array Elements
- Print All Elements: Use the ```@``` symbol within bracket notation:
```bash
echo "${transport[@]}"
# Output: car train bike bus
```
- Print Specific Element: Pass the index position inside brackets:
```bash
echo "${transport[1]}"
# Output: train
```
- Delete an Element: Use the ```unset``` built-in command:
```bash
unset transport[1]
```
- Overwrite or Assign Element: Assign a new value directly to a target index:
```bash
transport[1]='trainride'
```
### Task 6: Conditionals
#### Basic If/Else Syntax
Conditionals allow scripts to execute specific code blocks based on whether evaluated expressions return true or false.
```bash
if [ condition ]
then
    # Code executed if condition is true
else
    # Code executed if condition is false
fi
```
Note: Spaces are mandatory immediately inside both sides of the bracket ```[ ]``` operators (e.g., ```if [ $count -eq 10 ]```). Every ```if``` block must conclude with a matching ```fi```.
#### Relational Integer Operators
| Operator   | Description               | Example                                                                                       |
|:--         |:--                        |:--                                                                                            |
| -eq        | Equal to                  | [ $num -eq 10 ]                                                                               |
| -ne        | Not equal to              | [ $num -ne 10 ]                                                                               |          
| -gt        | Greater than              | [  $num -gt 5 ]                                                                               | 
| -lt        | Less than                 | [ $num -lt 20 ]                                                                               | 
| -ge        | Greater than or equal to  | [ $num -ge 10 ]                                                                               | 
| -le        | Less than or equal to     | [ $num -le 10 ]Comparison operators evaluate numeric conditions within conditional brackets:  |

#### File Test Operators
File condition flags evaluate attributes of target file system paths:
|Flag        | Description                                         | Example                  |
|:--         |:--                                                  |:--                       |
| -f         | Checks if path exists and is a regular file         | "[ -f ""$filename"" ]"   |
| -d         | Checks if path exists and is a directory            | "[ -d ""$directory"" ]"  |
| -r         | Checks if file exists and has read permissions      | "[ -r ""$filename"" ]"   |
| -w         | Checks if file exists and has write permissions     | "[ -w ""$filename"" ]"   |
| -x         | Checks if file exists and has execute permissions   | "[ -x ""$filename"" ]"   |

#### Multi-Condition Script Example
Combining logical operators (```&&``` for AND, ```||``` for OR) allows evaluating complex file states:
```bash
#!/bin/bash
filename=$1

if [ -f "$filename" ] && [ -w "$filename" ]
then
    echo "hello" > $filename
else
    touch "$filename"
    echo "hello" > $filename
fi
```
#### Task Questions & Answers
- Question 1: What is the flag to check if we have read access to a file?
  - Answer: ```-r```
- Question 2: What is the flag to check to see if it's a directory?
  - Answer: ```-d```
