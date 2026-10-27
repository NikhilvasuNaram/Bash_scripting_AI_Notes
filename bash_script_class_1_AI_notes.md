# Bash Scripting for DevOps: Class 1 (Easy Version)

**Goal of today:** write a small script, make it runnable, use variables, and test it.
**Time:** about 1.5 hours. Read a part, then type the examples yourself before moving on.

**Today's map (8 small parts):**

1. What is a script?
2. Your first script, line by line
3. Making it run (permissions)
4. Variables
5. Quotes (the most important habit)
6. Comments
7. Testing your script
8. Common mistakes

At the end: practice tasks and a cheat sheet.

---

## Part 1: What is a script?

You already type commands in the terminal. A **script** is just those same commands saved in a file, so you can run them again with one command.

Without a script, you type this every time:

```bash
cd /var/log
grep "ERROR" app.log | wc -l
```

With a script, you type one thing:

```bash
./check_errors.sh
```

**Think of it like a recipe.** You write the steps once. The shell (Bash) follows them from the **top line to the bottom line**, one by one.

**Why DevOps people use scripts:** deployments, backups, log cleanup, health checks, and connecting tools like `aws`, `kubectl`, `docker` and `terraform`.

---

## Part 2: Your first script, line by line

Create the file:

```bash
touch hello.sh
```

Open it in any editor (`nano hello.sh` or `vi hello.sh`) and type:

```bash
#!/bin/bash
# My first script
echo "Hello, World!"
echo "Today is $(date)"
echo "I am running on $(hostname)"
```

What each line does:

| Line | Meaning |
|---|---|
| `#!/bin/bash` | "Run this file using Bash." (explained in Part 2.1 below) |
| `# My first script` | A comment. Bash ignores it. It is a note for humans. |
| `echo "Hello, World!"` | Prints text on the screen. |
| `$(date)` | Runs the `date` command and puts its result into the text. |
| `$(hostname)` | Same idea, for the server name. |

Now make it runnable and run it (details in Part 3):

```bash
chmod +x hello.sh
./hello.sh
```

You will see something like this (your date and hostname will differ):

```
Hello, World!
Today is Tue Oct 27 10:30:45 UTC 2026
I am running on server1
```

### Part 2.1: The first line, `#!/bin/bash`

This line is called the **shebang**. It tells Linux **which program should read this file**.

- `#!/bin/bash` means "use Bash".
- You can write other programs too, for example `#!/usr/bin/python3` for a Python script.

**Very important:** it must start with `#!` (hash **and** exclamation mark). If you forget the `!`, it is just a comment and does nothing.

There are two common ways to write it:

| Shebang | In simple words |
|---|---|
| `#!/bin/bash` | "Bash is at this exact location." Works on almost all Linux servers. **Use this by default.** |
| `#!/usr/bin/env bash` | "Search for bash wherever it is installed." Use this if the script must also run on Mac or unusual systems. |

One more to know about: `#!/bin/sh` is **not the same as Bash**. It is a simpler shell with fewer features. If your script uses Bash features, always write `bash`, not `sh`.

---

## Part 3: Making the script run

### Step 1: Give it permission

A new file is not allowed to run as a program. You must add the **execute** permission:

```bash
chmod +x hello.sh
ls -l hello.sh
```

The output should start with something like `-rwxr-xr-x`. The `x` letters mean "can execute".

If you forget this step, you get:

```
bash: ./hello.sh: Permission denied
```

### Step 2: Run it

| How | Command | Needs `chmod +x`? |
|---|---|---|
| Direct | `./hello.sh` | Yes |
| Using bash | `bash hello.sh` | No |

Why `./` ? Linux only looks for programs in certain folders (the `PATH`). Your current folder is not one of them. So `./` means "the file in **this** folder".

If you type `hello.sh` without `./`, you get `command not found`.

### Step 3 (important idea): Running vs sourcing

There is a third way to run a script: `source hello.sh` (or `. hello.sh`). It behaves differently, and this matters a lot in DevOps.

**Think of it like rooms.**

- `./script.sh` opens a **new room**, does the work inside, then closes the room. Anything it changed (variables, current folder) stays in that room and is gone when it ends. Your own room is not changed.
- `source script.sh` does the work **in your own room**. So the changes stay.

Try this to see it. Create `setenv.sh`:

```bash
#!/bin/bash
export REGION=ap-south-1
cd /tmp
```

Run it normally:

```bash
./setenv.sh
echo "Region is: [$REGION]"
pwd
```

Result: `Region is: []` (empty) and you are still in the folder you started in.

Now source it:

```bash
source setenv.sh
echo "Region is: [$REGION]"
pwd
```

Result: `Region is: [ap-south-1]` and you are now in `/tmp`.

This is exactly why you type `source venv/bin/activate` (and not `./activate`) when you start a Python virtual environment. It must change **your** shell.

---

## Part 4: Variables

A **variable** is a name that holds a value, like a labelled box.

```bash
name="DevOps"
count=42
echo "Hello, $name"
echo "Count is $count"
```

Output:

```
Hello, DevOps
Count is 42
```

### The rules

1. **Create:** `name="value"` with **no spaces** around `=`.
   - Correct: `name="DevOps"`
   - Wrong: `name = "DevOps"` (Bash thinks `name` is a command)
2. **Use:** put a `$` in front: `$name`.
3. Use curly braces when letters come right after the name: `${name}_report`.
   - `$name_report` would look for a variable called `name_report` (which does not exist).
4. Names can contain letters, numbers and `_`. They cannot start with a number.
5. A common style: `lowercase_names` for your own variables, `UPPERCASE` for constants and environment variables.

### Saving a command's result into a variable

Put the command inside `$( )`:

```bash
current_user=$(whoami)
today=$(date +%F)
file_count=$(ls | wc -l)

echo "User: $current_user"
echo "Date: $today"
echo "Files here: $file_count"
```

### Three small extras

```bash
readonly APP_NAME="billing-api"   # cannot be changed later
unset count                       # deletes a variable
echo "$0"                         # the name of the script itself
echo "$?"                         # result of the last command (0 = success)
```

---

## Part 5: Quotes (the most important habit)

Quotes decide how Bash reads your text. There are two kinds:

```bash
name="John Doe"

echo "Hello $name"     # Hello John Doe
echo 'Hello $name'     # Hello $name
```

| Quote | What happens |
|---|---|
| Double quotes `" "` | Variables **are** replaced by their values. |
| Single quotes `' '` | Everything is printed **exactly as typed**. |

### Why you must quote variables

Look at what happens with a file name that has a space:

```bash
file="my report.txt"

touch $file        # creates 2 files: "my" and "report.txt"   (wrong!)
touch "$file"      # creates 1 file: "my report.txt"          (correct)
```

Without quotes, Bash cuts the text at every space and treats the pieces as separate things.

**Golden rule: write `"$variable"` with double quotes almost every time.**

This is serious in DevOps. Imagine a script that deletes or copies files and the name contains a space. The wrong files could be touched.

---

## Part 6: Comments

A comment is a note for humans. Bash skips it.

```bash
# This is a full-line comment
echo "Hello"    # This is a comment after a command
```

Good comments explain **why** you did something, not what the command does (the command already shows that).

```bash
# Wait 5 seconds because the service needs time to start   <- good (why)
sleep 5

# sleep for 5 seconds                                      <- not useful (what)
sleep 5
```

Put a short header at the top of each script:

```bash
#!/bin/bash
# Script:  system_info.sh
# Purpose: Show basic server information
# Author:  Your Name
# Usage:   ./system_info.sh
```

For many lines of comment, the simple method is to start each line with `#`.

---

## Part 7: Testing your script

Before using a script on a real server, test it. Three simple tools:

```bash
bash -n script.sh     # Checks the spelling/grammar only. Does NOT run it.
bash -x script.sh     # Runs it and shows every command as it executes (great for debugging).
echo $?               # After running: 0 means success, anything else means failure.
```

Example of `bash -n` catching an error. This file has a missing closing quote:

```bash
#!/bin/bash
echo "hello
```

`bash -n` says: `unexpected EOF while looking for matching '"'`. Nothing was executed.

Also test with awkward input: an empty value, a name with spaces, a file that does not exist.

**Bonus tool (not in your books):** `shellcheck script.sh` reads your script and points out mistakes, such as missing quotes. Install it with `apt install shellcheck`. Many DevOps teams run it in their pipelines.

---

## Part 8: Common mistakes

| # | Mistake | Wrong | Correct |
|---|---|---|---|
| 1 | No shebang, or missing `!` | `#/bin/bash` | `#!/bin/bash` |
| 2 | Spaces around `=` | `name = "x"` | `name="x"` |
| 3 | No quotes on a variable | `echo $name` | `echo "$name"` |
| 4 | Forgot to make it executable | `Permission denied` | `chmod +x script.sh` |
| 5 | Forgot `./` | `script.sh` gives `command not found` | `./script.sh` |

And a few error messages you may meet:

| Error | Likely reason | Fix |
|---|---|---|
| `Permission denied` | No execute permission | `chmod +x script.sh` |
| `command not found` | Forgot `./` | `./script.sh` |
| `bad interpreter ... ^M` | File was made on Windows (extra hidden `\r` characters) | `dos2unix script.sh` |
| `Permission denied` even with `x` | The folder is mounted as `noexec` | `bash script.sh` or move the file |

---

## Full example: a system info script

Type this one yourself. Every line uses something you learned today.

```bash
#!/bin/bash
# Script:  system_info.sh
# Purpose: Show basic server information
# Usage:   ./system_info.sh

# Save values once, use them below
host=$(hostname)
today=$(date)
user=$(whoami)

echo "=============================="
echo "   System Information Report"
echo "=============================="
echo "Host:    $host"
echo "Date:    $today"
echo "User:    $user"
echo "Home:    $HOME"
echo "Kernel:  $(uname -r)"
echo "Uptime:  $(uptime -p)"
echo ""
echo "Disk usage of /:"
df -h /
echo "=============================="
```

Then:

```bash
chmod +x system_info.sh
bash -n system_info.sh      # syntax check
./system_info.sh            # run it
echo $?                     # should print 0
```

---

## Practice tasks (do these before the assignment)

1. Type `hello.sh` by hand. Make it executable and run it with `./hello.sh` and with `bash hello.sh`.
2. Remove the execute permission with `chmod -x hello.sh`. Run `./hello.sh` and read the error. Then run `bash hello.sh`.
3. Make `setenv.sh` from Part 3. Run it with `./` and with `source`. Compare the results.
4. Type `system_info.sh`. Add two lines of your own (try `free -h` for memory or `who | wc -l` for logged-in users).
5. Break `system_info.sh` on purpose (delete a closing quote). Run `bash -n` and read the message.
6. Run `file="my report.txt"`, then `touch $file`, then `ls`. Delete those files. Now run `touch "$file"` and `ls` again.

---

## Cheat sheet

```bash
#!/bin/bash                  # first line (hash + exclamation mark)
# text                       # comment
name="value"                 # create variable (no spaces around =)
echo "$name"                 # use variable (with quotes)
result=$(command)            # save command output
chmod +x script.sh           # allow running
./script.sh                  # run (new room)
bash script.sh               # run without chmod
source script.sh             # run in your own room
bash -n script.sh            # syntax check
bash -x script.sh            # show each step
echo $?                      # 0 = success
```

## Five things to remember

1. A script is saved commands, run top to bottom.
2. The first line is `#!/bin/bash` (do not forget the `!`).
3. `chmod +x`, then `./script.sh`.
4. Variables: `name="value"` (no spaces), use as `"$name"` (with quotes).
5. `./script.sh` runs in a new room. `source script.sh` runs in your room.

---

## Extra (optional, read after you are comfortable)

- **Empty variable danger:** `rm -rf $dir/*` becomes `rm -rf /*` if `dir` is empty. Never run delete commands on a variable you have not checked. We will learn safe patterns in later days.
- **Mac:** macOS ships an old Bash (3.2) that lacks some newer features.
- **`set -e`:** you may see this near the top of scripts. It stops the script at the first error. We will study it properly on Day 12.

---

When you have read this and practiced, say **"day-1 assignment"** again if you want to continue, or just send your answers for Q3 to Q10 from the assignment I already gave you.
