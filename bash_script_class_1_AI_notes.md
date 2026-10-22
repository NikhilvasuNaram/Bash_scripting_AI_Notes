# Bash Shell Scripting for DevOps: Class 1 (Day 1)

**Topic:** Script anatomy, running scripts, permissions, comments, variables basics, testing, and common mistakes
**Time:** about 1.5 hours (30 min reading, 60 min hands-on)
**Sources:** *Linux Shell Scripting for DevOps*, Ch0; *Introduction to Bash Scripting* (Intro, Structure, Hello World, Variables, Comments)

> **Level note:** You already run commands all day. Day 1 is about the *discipline* of turning commands into a script that behaves the same on your laptop, a jump host, a Docker container and a CI runner. Read the "Gotcha" boxes carefully; they are where experienced engineers still get bitten.

---

## 1. What a shell script really is

A shell script is a plain text file of commands that an interpreter (here, **Bash**) reads and executes **top to bottom, one line at a time**. There is no compile step, and a syntax error halfway down only surfaces when execution reaches it (so a script can run its first 20 lines and then fail).

```bash
# Without a script (typed by hand each time)
cd /var/log
grep "ERROR" app.log | wc -l

# With a script: same steps, repeatable
./check_errors.sh
```

### Why it matters in DevOps

| Area | Typical use |
|---|---|
| Automation | Deployments, backups, log cleanup, service restarts |
| Configuration | Server bootstrap, user and permission setup |
| CI/CD | Build, test and deploy steps in Jenkins / GitHub Actions / GitLab CI, health checks after deploy |
| Monitoring | Disk, CPU and log checks, alerting hooks |
| Glue | Wiring together `aws`, `kubectl`, `docker`, `terraform`, `curl`, `jq` |

Rule of thumb: Bash is excellent glue for **orchestrating other tools**. When a script needs complex data structures, heavy parsing or lots of logic, that is the signal to move to Python.

---

## 2. Your first script

```bash
touch hello.sh
```

```bash
#!/bin/bash
# My first shell script
echo "Hello, World!"
echo "Today is $(date)"
echo "I am running on $(hostname)"
```

```bash
chmod +x hello.sh
./hello.sh
```

Four steps to remember: **create, write, make executable, run.**

---

## 3. The shebang (`#!`)

The first line tells the kernel *which interpreter to use* when the file is executed directly.

```bash
#!/bin/bash            # Bash at a fixed path
#!/usr/bin/env bash    # Finds bash via PATH (more portable)
#!/bin/sh              # POSIX sh (no Bash-only features!)
#!/usr/bin/python3     # Works for any interpreter, not only shells
```

### Which one to use?

| Shebang | Use when |
|---|---|
| `#!/bin/bash` | Servers and containers you control. Predictable. |
| `#!/usr/bin/env bash` | Script must run on macOS, Homebrew bash, NixOS, or anywhere bash is not at `/bin/bash`. |
| `#!/bin/sh` | You want strict POSIX portability (Alpine/BusyBox images). **Do not use Bash features** (arrays, `[[ ]]`, etc.) here. |

> **Gotcha: `sh` is not always bash.** On Debian/Ubuntu, `/bin/sh` is **dash**; on Alpine it is **BusyBox ash**. A script with `#!/bin/sh` that uses `[[ ]]` or arrays will fail in those environments. This is a classic "works on my machine, fails in the CI container" bug.

> **Gotcha: macOS ships Bash 3.2** (old, no associative arrays). If your script must run on Mac laptops, either avoid Bash 4+ features or require a newer bash explicitly.

If the shebang is missing, the script still works when you call `bash script.sh`, but direct execution (`./script.sh`) falls back to whatever shell launched it, which makes behavior unpredictable.

---

## 4. Permissions and ways to run a script

### Execute permission

```bash
chmod +x script.sh      # add execute for everyone (according to umask rules)
chmod 755 script.sh     # rwxr-xr-x  (owner all; others read + execute)
chmod 700 script.sh     # rwx------  (owner only; good for scripts with secrets or sensitive logic)
ls -l script.sh         # verify
```

### Ways to execute a script, and why they differ

| Method | Command | Needs `+x`? | Runs in |
|---|---|---|---|
| Direct | `./script.sh` | Yes | **New child process** |
| Explicit interpreter | `bash script.sh` | No | New child process |
| From `PATH` | `script.sh` (after `sudo mv script.sh /usr/local/bin/`) | Yes | New child process |
| **Sourced** | `source script.sh` or `. script.sh` | No | **Your current shell** |

**Why this matters:** a child process cannot change its parent's environment.

```bash
# set_env.sh
export APP_ENV=staging
cd /opt/app
```

```bash
./set_env.sh
echo "$APP_ENV"      # (empty) - child process died, changes gone
pwd                  # unchanged

source set_env.sh
echo "$APP_ENV"      # staging
pwd                  # /opt/app
```

This is exactly how `source venv/bin/activate` and `source ~/.bashrc` work.

### Common errors and what they mean

| Error | Cause | Fix |
|---|---|---|
| `Permission denied` | No execute bit | `chmod +x script.sh` |
| `command not found` (script is in current dir) | Current dir is not in `PATH` | `./script.sh` |
| `bad interpreter: No such file or directory` / `bash\r: No such file...` | **Windows (CRLF) line endings** | `dos2unix script.sh` or `sed -i 's/\r$//' script.sh` |
| `Permission denied` even with `+x` | Filesystem mounted `noexec` (common on `/tmp` or hardened hosts) | Run with `bash script.sh` or move the script |

> **Gotcha: CRLF.** If you edit scripts on Windows and copy them to Linux, the invisible `\r` becomes part of the shebang (`/bin/bash\r`), and the kernel cannot find that interpreter. Very common in teams mixing Windows and Linux.

---

## 5. Anatomy of a well-structured script

```bash
#!/bin/bash
# Script:  system_info.sh
# Purpose: Display basic system information
# Author:  Your Name
# Usage:   ./system_info.sh

set -e                      # exit on error (we will study this properly in Day 12)

# --- Variables ---
HOSTNAME=$(hostname)
DATE=$(date)

# --- Main logic ---
echo "=== System Information ==="
echo "Hostname:   $HOSTNAME"
echo "Date:       $DATE"
echo "Uptime:     $(uptime -p)"
echo "Disk usage: $(df -h / | awk 'NR==2 {print $5}')"
```

Five building blocks to always think about:

1. **Shebang**: which interpreter
2. **Header comments**: what it does, how to use it
3. **Settings**: shell options such as `set -e`
4. **Variables**: values defined once, reused everywhere
5. **Main logic**: the actual work

> Do not worry about `set -e` yet. For today, just know it makes the script stop at the first failing command. We will cover it, and its traps, on Day 12.

---

## 6. Comments

```bash
# Single-line comment
echo "Hello"      # Comment after code

# Multi-line comment using the no-op colon command
: '
This is a multi-line comment.
Everything inside the single quotes is ignored.
'
```

> **Gotcha: the `: '...'` trick breaks if the comment contains a single quote** (`don't` ends the string early). The safer approach is a quoted heredoc:
>
> ```bash
> : <<'COMMENT'
> This block can contain don't, "quotes", $variables, anything.
> COMMENT
> ```
>
> Many teams simply prefix each line with `#`, which is the most readable and the least surprising.

**Comment well:** explain **why**, not what. Add usage examples at the top of the script, and use `TODO:` markers sparingly.

---

## 7. Variables basics

### Assigning and using

```bash
name="DevOps"          # NO spaces around =
count=42
echo "Hello, $name"
echo "Count: $count"
echo "File: ${name}_report.txt"   # braces needed when text follows the name
```

### Rules

- **No spaces around `=`**: `name = "x"` is parsed as running a command called `name`.
- Names: letters, digits, underscore; cannot start with a digit. Case-sensitive.
- Convention: `lowercase_with_underscores` for script-local variables, `UPPERCASE` for environment variables and constants.
- Everything is a **string** by default. Numbers become numbers only in arithmetic context (Day 4).
- Use `${var}` when the variable is directly followed by letters, digits or underscores (`${name}_report` vs `$name_report`, which looks for a variable called `name_report`).

### Command substitution

```bash
current_user=$(whoami)
file_count=$(ls | wc -l)
today=$(date +%F)
```

Use `$(...)`, not backticks `` `...` ``. It nests cleanly and is easier to read.

### Quoting: the single most important habit

```bash
name="John Doe"

echo "Hello $name"     # Hello John Doe        (double quotes: variables expand)
echo 'Hello $name'     # Hello $name           (single quotes: everything literal)
echo Hello \$name      # Hello $name           (backslash escapes one character)
```

**Always quote variable expansions:** `"$var"`.

```bash
file="my report.txt"

touch $file           # creates TWO files: "my" and "report.txt"
touch "$file"         # creates ONE file: "my report.txt"
```

In DevOps scripts that delete, copy or deploy, an unquoted variable is a real risk. It can split on spaces, expand wildcards, or, if empty, disappear completely:

```bash
dir=""
rm -rf $dir/*         # becomes: rm -rf /*     <- disaster
rm -rf "$dir"/*       # becomes: rm -rf /*   (quoting alone does not save you!)
```

Quoting helps with spaces but **an empty variable is its own problem**. Later days will teach `set -u` and `${var:?message}` to guard against it. For now: *never* run destructive commands on a variable you have not checked.

### A few extras worth knowing now

```bash
readonly APP_NAME="billing-api"   # cannot be changed or unset afterwards
unset count                       # remove a variable

# Special variables you will see everywhere (we go deeper on Day 2)
echo "$0"    # name of the script
echo "$$"    # PID of the current shell/script
echo "$?"    # exit status of the last command (0 = success)
```

---

## 8. Testing your scripts

Develop the habit from day one:

```bash
bash -n script.sh      # 1. Syntax check only. Does NOT execute anything.
bash -x script.sh      # 2. Trace mode: prints each command (after expansion) before running it
./script.sh            # 3. Run it
echo $?                # 4. Check exit code: 0 = success, non-zero = failure
```

Test with awkward input too: empty string, a path with spaces, a nonexistent file.

```bash
./script.sh test1.txt
./script.sh "/path/with spaces/file"
./script.sh ""
```

**Bonus (not in the books, but standard practice in DevOps teams):** use **ShellCheck**, a static analyzer that catches unquoted variables, wrong test syntax, and more.

```bash
shellcheck script.sh
```

It is available via `apt install shellcheck`, `brew install shellcheck`, and as a pre-commit hook or CI step.

---

## 9. The five classic beginner mistakes (and the fixes)

| # | Mistake | Wrong | Correct |
|---|---|---|---|
| 1 | Forgetting the shebang | `echo "hello"` | `#!/bin/bash` first line |
| 2 | Spaces around `=` | `name = "value"` | `name="value"` |
| 3 | Not quoting variables | `echo $file_name` | `echo "$file_name"` |
| 4 | Forgetting execute permission | `script.sh` -> Permission denied | `chmod +x script.sh` |
| 5 | Running by bare name from current dir | `script.sh` -> command not found | `./script.sh` |

---

## 10. Worked example: system info report

```bash
#!/bin/bash
# system_info.sh - Display system information

echo "==================================="
echo "      System Information Report"
echo "==================================="
echo ""
echo "Date and Time:"
echo "    $(date)"
echo ""
echo "System:"
echo "    Hostname: $(hostname)"
echo "    OS: $(uname -s)"
echo "    Kernel: $(uname -r)"
echo ""
echo "User:"
echo "    Current user: $(whoami)"
echo "    Home directory: $HOME"
echo "    Shell: $SHELL"
echo ""
echo "System Load:"
echo "    Uptime: $(uptime -p 2>/dev/null || uptime)"
echo ""
echo "Disk Usage:"
df -h / | awk 'NR==1 {print "       " $0} NR==2 {print "      " $0}'
echo ""
echo "Report generated by: $0"
echo "==================================="
```

What it demonstrates:

- Shebang, comments, and readable structure
- Built-in variables (`$HOME`, `$SHELL`, `$0`)
- Command substitution (`$(date)`, `$(hostname)`)
- `2>/dev/null ||` as a simple fallback when a flag is not supported (redirection gets a full day on Day 9)

---

## 11. Hands-on practice (do these before asking for the assignment)

1. Type the `hello.sh` script by hand (do not paste), make it executable, and run it three ways: `./hello.sh`, `bash hello.sh`, and from `/usr/local/bin`.
2. Remove the execute bit and observe the error. Then run it with `bash hello.sh`.
3. Create `set_env.sh` that exports a variable and changes directory. Run it with `./` and with `source`; compare the results.
4. Create `system_info.sh` from Section 10. Add at least two lines of your own (for example, memory with `free -h`, or the number of logged-in users with `who | wc -l`).
5. Run `bash -n` and `bash -x` on it. Deliberately break something (remove a closing quote) and see what `bash -n` reports.
6. Make a file called `my report.txt` using `touch $file` and `touch "$file"` with `file="my report.txt"`. Run `ls` after each and compare.
7. If you can, install `shellcheck` and run it on your scripts.

---

## 12. Quick reference

```bash
#!/bin/bash                      # shebang
# comment                        # comment
var="value"                      # assign (no spaces)
echo "$var"  /  echo "${var}"    # use (always quote)
x=$(command)                     # capture command output
readonly VAR=1 ; unset var       # constant / remove
chmod +x f.sh ; ./f.sh           # permission + run
bash f.sh                        # run without +x
source f.sh   /   . f.sh         # run in current shell
bash -n f.sh                     # syntax check
bash -x f.sh                     # trace
echo $?                          # last exit code
```

## 13. Key takeaways

- A script = shebang + comments + settings + variables + logic.
- `./script.sh` and `bash script.sh` run in a **child** process; `source` runs in your **current** shell.
- Assign with **no spaces** around `=`; use `"$var"` **with quotes** almost always.
- Test early: `bash -n`, `bash -x`, and check `$?`.
- `#!/bin/sh` is **not** Bash. Be careful in Alpine/Debian containers.

---

**When you have read and practiced, say "day-1 assignment" and I will quiz you.**
