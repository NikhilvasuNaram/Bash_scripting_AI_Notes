# Bash Scripting for DevOps: Class 2

**Topic:** Environment variables, PATH, default values, user input, command-line arguments
**Level:** Intermediate. You know what a script and a variable are; today is about making scripts **configurable and reusable**.
**Time:** about 1.5 to 2 hours
**Sources:** *Linux Shell Scripting for DevOps* Ch1 and Ch2; *Introduction to Bash Scripting* (User Input, Arguments)

**Today's map:**

1. Environment variables (shell variables vs exported variables)
2. PATH
3. Default values and safe expansion (`:-`, `:=`, `:?`)
4. Reading user input with `read`
5. Command-line arguments (`$1`, `$#`, `$@` vs `$*`, `shift`)
6. Validating input and exit codes
7. Putting it together: a small deploy script

---

## 1. Environment variables

An **environment variable** is a named value that is passed from a parent process to every child process it starts. This is how configuration reaches your scripts without being hardcoded: the same script runs in dev, staging and prod and only the environment differs.

### Reading them

```bash
echo "$HOME"        # /home/nikhil
echo "$USER"
echo "$SHELL"
env                 # all exported variables
printenv HOME       # one variable (no $ sign)
```

### Shell variable vs environment variable

```bash
MY_VAR="hello"          # shell variable: visible only in this shell
export MY_VAR="hello"   # environment variable: visible to child processes too
```

Quick proof:

```bash
NAME="local"
bash -c 'echo "child sees: [$NAME]"'     # child sees: []

export NAME="exported"
bash -c 'echo "child sees: [$NAME]"'     # child sees: [exported]
```

This connects to Day 1: a child process gets a **copy** of the exported variables. If it changes them, your shell never sees the change.

### Setting a variable for one command only

```bash
DEBUG=true ./script.sh       # DEBUG exists only for this run
AWS_REGION=ap-south-1 aws s3 ls
```

This is very common in CI/CD and when testing scripts.

### Where environment variables come from in DevOps

| Source | Example |
|---|---|
| CI/CD | GitHub Actions `env:`, Jenkins `environment {}`, GitLab CI variables |
| Containers | `docker run -e DB_HOST=db`, Kubernetes `env:` / `envFrom:` |
| Files | `.env` loaded with `source .env` or `set -a; source .env; set +a` |
| Shell profiles | `~/.bashrc`, `/etc/environment` |

**Rule:** never hardcode secrets in a script. Read them from the environment (and don't `echo` them into logs).

### Useful built-in variables

| Variable | Meaning |
|---|---|
| `$HOME`, `$USER`, `$SHELL` | home directory, current user, login shell |
| `$PWD` | current directory |
| `$HOSTNAME` | machine name |
| `$RANDOM` | random number 0 to 32767 |
| `$$` | PID of the current shell/script |
| `$?` | exit status of the last command |

---

## 2. PATH

`PATH` is a colon-separated list of directories. When you type a command, the shell checks each directory **in order** and runs the first match.

```bash
echo "$PATH"
# /usr/local/bin:/usr/bin:/bin

which ls            # /usr/bin/ls   (which file will run)
type ls             # also shows aliases, functions, builtins
```

### Adding a directory

```bash
export PATH="$PATH:/opt/tools/bin"      # add at the END (lowest priority)
export PATH="/opt/tools/bin:$PATH"      # add at the START (takes priority)
```

Always keep `$PATH` in the new value. `export PATH="/opt/tools/bin"` on its own wipes everything else, and then even `ls` stops working in that shell.

### Why `./script.sh` needs the `./`

Your current directory (`.`) is not in `PATH` (deliberately, for security). So `script.sh` is not found, but `./script.sh` is a direct path. To run a script from anywhere, either put it in a PATH directory (`/usr/local/bin`) or add its folder to `PATH`.

### Changes are temporary

`export PATH=...` lasts only for that shell session. To make it permanent, add the line to `~/.bashrc` (or `~/.bash_profile`) and open a new shell or run `source ~/.bashrc`.

### Counting PATH entries (a useful pattern)

```bash
IFS=: read -ra dirs <<< "$PATH"
echo "PATH has ${#dirs[@]} directories"
for d in "${dirs[@]}"; do echo "  $d"; done
```

`IFS=:` tells `read` to split on colons, `-a` stores the pieces in an array. (Arrays get a full day on Day 7; for now just know the pattern.)

---

## 3. Default values and safe expansion

Scripts often need "use this value if set, otherwise a fallback". Bash has built-in operators for it.

| Syntax | Meaning |
|---|---|
| `${var:-default}` | Use `default` if `var` is **unset or empty**. Does not change `var`. |
| `${var:=default}` | Same, but **also assigns** the default to `var`. |
| `${var:?message}` | If `var` is unset or empty, **print `message` and exit** (the script stops with an error). |
| `${var:+value}` | Use `value` only if `var` **is set** and not empty; otherwise nothing. |

```bash
LOG_DIR="${LOG_DIR:-/var/log/myapp}"       # default if not provided
ENVIRONMENT="${ENVIRONMENT:-development}"

echo "${DB_HOST:?DB_HOST must be set}"     # stops here if DB_HOST is missing
```

A good DevOps pattern: **optional settings get defaults, required settings use `:?`.**

```bash
#!/bin/bash
: "${API_KEY:?API_KEY is required}"        # required; the leading : just discards the value
PORT="${PORT:-8080}"                       # optional
echo "Starting on port $PORT"
```

Run it:

```bash
./app.sh                      # app.sh: line 3: API_KEY: API_KEY is required
API_KEY=abc ./app.sh          # Starting on port 8080
API_KEY=abc PORT=9000 ./app.sh
```

`:-` vs `-` (without colon): `${var-default}` only uses the default when `var` is **unset**; `${var:-default}` also when it is **set but empty**. In practice you almost always want the colon version.

This also answers the "empty variable danger" from Day 1. `rm -rf "${BACKUP_DIR:?}"/*` refuses to run if `BACKUP_DIR` is empty.

---

## 4. Reading user input with `read`

```bash
read name
echo "Hello, $name"
```

### Useful options

| Option | What it does | Example |
|---|---|---|
| `-p "prompt"` | show a prompt | `read -p "Name: " name` |
| `-s` | **silent** (hide typing), for passwords | `read -s -p "Password: " pass` |
| `-t N` | time out after N seconds | `read -t 10 -p "Continue? " ans` |
| `-n N` | read exactly N characters, no Enter needed | `read -n 1 -p "y/n? " ans` |
| `-r` | don't treat backslash as an escape (use almost always) | `read -r line` |

```bash
#!/bin/bash
read -r -p "Enter environment (dev/prod): " env
read -s -p "Enter password: " password
echo ""                                  # newline, because -s hides Enter too
echo "Deploying to $env"
```

### Default answer

```bash
read -r -p "Region [ap-south-1]: " region
region="${region:-ap-south-1}"           # empty answer falls back to the default
```

### Confirmation before something risky

```bash
read -r -p "Delete all logs? (y/N): " answer
if [[ "$answer" == "y" || "$answer" == "Y" ]]; then
    echo "Deleting..."
else
    echo "Cancelled."
fi
```

(The `if` syntax is covered fully on Days 4 and 5. Today just read the pattern.)

### Important: input vs arguments in automation

`read` waits for a human. In CI/CD or cron there is nobody to type, so the script **hangs**. For anything automated, accept values as **arguments or environment variables**, and use `read` only for interactive tools. A common design is: arguments first, and prompt only if the argument is missing.

---

## 5. Command-line arguments

```bash
./script.sh alice 25 london
```

| Variable | Value here | Meaning |
|---|---|---|
| `$0` | `./script.sh` | script name |
| `$1` | `alice` | first argument |
| `$2` | `25` | second argument |
| `$3` | `london` | third argument |
| `$#` | `3` | number of arguments |
| `$@` | `alice 25 london` | all arguments (as separate words when quoted) |
| `$*` | `alice 25 london` | all arguments (as one string when quoted) |

For the 10th argument and beyond, use braces: `${10}`. (`$10` means `$1` followed by `0`.)

### Basic example

```bash
#!/bin/bash
# greet.sh
name=$1
age=$2
echo "Hello, $name! You are $age."
```

```bash
./greet.sh Alice 25         # Hello, Alice! You are 25.
```

### Default values for arguments

```bash
name="${1:-Guest}"
age="${2:-unknown}"
```

### Arguments with spaces

```bash
./greet.sh "John Doe" 30     # $1 = John Doe,  $2 = 30
./greet.sh John Doe 30       # $1 = John, $2 = Doe, $3 = 30   (probably not what you meant)
```

And **inside** the script, always quote `"$1"` too, or a value with spaces gets split again.

---

## 6. `$@` vs `$*` (a classic interview question)

Both mean "all the arguments", and they only differ **when quoted**:

| Form | Behaviour |
|---|---|
| `"$@"` | Each argument stays a **separate word** (spaces inside an argument are preserved) |
| `"$*"` | All arguments are joined into **one single string** (using the first character of `IFS`, a space by default) |

Demonstration:

```bash
#!/bin/bash
# args_demo.sh
echo "--- using \"\$@\" ---"
for a in "$@"; do echo "[$a]"; done

echo "--- using \"\$*\" ---"
for a in "$*"; do echo "[$a]"; done
```

```bash
./args_demo.sh "John Doe" 30 london
```

Output:

```
--- using "$@" ---
[John Doe]
[30]
[london]
--- using "$*" ---
[John Doe 30 london]
```

**Rule of thumb:**

- Use **`"$@"`** when you pass arguments on to another command or loop over them. It is the correct and safe default.
- Use **`"$*"`** only when you need everything as a single string, for example for a log message.
- Unquoted `$@` and `$*` behave the same and both break on spaces. Avoid them.

Example of forwarding arguments to another command:

```bash
#!/bin/bash
# wrapper.sh: run a command with logging
echo "Running: $*" >> /tmp/wrapper.log     # one string for the log line
"$@"                                        # run it with arguments intact
```

```bash
./wrapper.sh ls -l "/path/with spaces"
```

---

## 7. `shift` (processing arguments one at a time)

`shift` removes `$1` and moves everything down: `$2` becomes `$1`, and so on. `$#` decreases by one.

```bash
#!/bin/bash
# shift_demo.sh
echo "First: $1"
shift
echo "Now first is: $1, remaining: $#"
```

```bash
./shift_demo.sh a b c
# First: a
# Now first is: b, remaining: 2
```

A common use: take the first argument as a "command" or "environment" and treat the rest as a list.

```bash
env="$1"
shift
for server in "$@"; do
    echo "Deploying to $server in $env"
done
```

```bash
./deploy.sh prod web1 web2 web3
```

Looping over arguments is also possible without `shift`: `for arg in "$@"; do ... done`.

(Parsing flags like `-v` and `--env prod` properly is done with `getopts` or a `case` loop. We will do that later, once you know `case`.)

---

## 8. Validating arguments and exit codes

A script that is used by others (or by a pipeline) should check its input and fail clearly.

```bash
#!/bin/bash
# fileinfo.sh <filename>

if [ $# -eq 0 ]; then
    echo "Usage: $0 <filename>" >&2        # error messages go to stderr
    exit 1
fi

file="$1"

if [ ! -f "$file" ]; then
    echo "Error: file '$file' not found" >&2
    exit 2
fi

echo "Name:  $(basename "$file")"
echo "Size:  $(wc -c < "$file") bytes"
echo "Lines: $(wc -l < "$file")"
```

Points to notice:

- `$# -eq 0` means "no arguments given". Show **usage** and exit with a non-zero code.
- `-f` is a file test ("is a regular file"); `!` negates it. (More tests on Day 4.)
- `>&2` sends the message to **stderr**, so it doesn't pollute normal output that another command might be reading.
- Different exit codes for different failures (1 = bad usage, 2 = file missing) make debugging and pipelines easier.
- `exit 0` is success, anything else is failure. Check it with `echo $?`.

Shorter ways to validate:

```bash
[ $# -ge 2 ] || { echo "Usage: $0 <env> <version>" >&2; exit 1; }
version="${2:?version is required}"
```

---

## 9. Putting it together: a small deploy script

This one uses everything from today: arguments, defaults, environment variables, validation.

```bash
#!/bin/bash
# deploy.sh - Usage: ./deploy.sh <environment> <version> [region]
# Optional environment variable: DRY_RUN=true

# --- Validate arguments ---
if [ $# -lt 2 ]; then
    echo "Usage: $0 <environment> <version> [region]" >&2
    exit 1
fi

# --- Read arguments, with defaults ---
environment="$1"
version="$2"
region="${3:-ap-south-1}"
dry_run="${DRY_RUN:-false}"

# --- Required secret from environment ---
: "${DEPLOY_TOKEN:?DEPLOY_TOKEN must be set}"

# --- Do the work ---
echo "Environment: $environment"
echo "Version:     $version"
echo "Region:      $region"
echo "Dry run:     $dry_run"

if [ "$dry_run" = "true" ]; then
    echo "[dry run] would deploy $version to $environment"
else
    echo "Deploying $version to $environment..."
fi
```

Try these:

```bash
./deploy.sh                                       # usage message, exit 1
./deploy.sh prod v1.2.0                           # fails: DEPLOY_TOKEN must be set
DEPLOY_TOKEN=xyz ./deploy.sh prod v1.2.0
DEPLOY_TOKEN=xyz DRY_RUN=true ./deploy.sh prod v1.2.0 us-east-1
```

---

## 10. Common mistakes today

| Mistake | Problem | Fix |
|---|---|---|
| `export PATH="/new/dir"` | Wipes the old PATH | `export PATH="$PATH:/new/dir"` |
| Using `$@` or `$*` without quotes | Splits arguments with spaces | `"$@"` |
| `$10` for the tenth argument | Read as `$1` then `0` | `${10}` |
| `read` in a CI/cron script | Hangs forever | Use arguments / env vars |
| Forgetting `export` | Child process can't see the variable | `export VAR=...` |
| `read` without `-r` | Backslashes get eaten | `read -r` |
| Echoing secrets | Leaks into logs | Don't print them; use `read -s` for input |
| No argument validation | Confusing errors later | Check `$#`, show usage |

---

## 11. Practice tasks

1. Run `export NAME=exported` and `NAME2=local`. In a child (`bash -c 'echo "[$NAME] [$NAME2]"'`), which one is visible?
2. Add `/tmp/mytools` to the end of your PATH. Put a tiny script `hi.sh` in it (with `chmod +x`) and run `hi.sh` from another directory. Then try prepending instead and note the difference in order (`echo $PATH`).
3. Write `config.sh` that sets `LOG_DIR`, `ENVIRONMENT` and `PORT` with `:-` defaults, prints them, and run it with and without overriding values (`PORT=9000 ./config.sh`).
4. Make a script that stops with an error if `API_KEY` is not set, using `${API_KEY:?...}`.
5. Write `greet.sh` that asks for a name with `read -r -p`, falls back to "Guest" if empty, and greets the user.
6. Write `args_demo.sh` from section 6. Run it with `"John Doe" 30 london` and note exactly how `"$@"` and `"$*"` differ.
7. Write `fileinfo.sh` from section 8. Test: no argument, a missing file, an existing file. Check `echo $?` each time.
8. Type out `deploy.sh` from section 9 and run all four test commands.

---

## 12. Cheat sheet

```bash
export VAR=value                  # make visible to child processes
VAR=value ./cmd                   # set for one command only
printenv VAR ; env                # inspect
export PATH="$PATH:/new/dir"      # extend PATH
${v:-default}                     # default if unset/empty
${v:=default}                     # default and assign
${v:?message}                     # required (exit with message)
${v:+alt}                         # alt only if v is set
read -r -p "Prompt: " v           # input (add -s for secrets)
$0 $1 $2 ... ${10}                # script name, arguments
$#                                # argument count
"$@"                              # all args, kept separate (use this)
"$*"                              # all args, one string
shift                             # drop $1
[ $# -lt 2 ] && { echo "Usage..." >&2; exit 1; }
```

## 13. Key takeaways

- **Exported** variables reach child processes; plain shell variables do not.
- `PATH` is searched left to right. Always append/prepend with `$PATH` kept in the value.
- Use `:-` for optional settings and `:?` for required ones.
- `read` is for humans; arguments and environment variables are for automation.
- `"$@"` keeps arguments intact; `"$*"` joins them into one string.
- Validate input early, print usage to stderr, and exit non-zero on failure.

---

**When you have read this and practiced, say "day-2 assignment".**
