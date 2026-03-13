# Bash Scripting for DevOps / Platform Engineering — Mid-Level Interview Notes

Bash scripting is a **core skill for DevOps engineers** because it enables **automation, system management, CI/CD scripting, monitoring, and infrastructure operations**. Mid-level candidates are expected to understand **how Bash works internally, write reliable scripts, debug failures, and handle production edge cases**.

---

# 1. Core Concepts (Mid-Level Focus)

## What is Bash?

**Bash (Bourne Again Shell)** is a Unix shell and command language used to interact with the operating system.

It acts as:

* Command interpreter
* Scripting language
* Automation tool

Example:

```bash
echo "Hello DevOps"
```

---

## What is a Bash Script?

A **Bash script** is a file containing commands executed sequentially by the shell.

Example:

```bash
#!/bin/bash

echo "Starting deployment..."
docker pull myimage
docker run myimage
```

The first line is called **shebang**.

---

## Shebang (`#!`)

Specifies which interpreter should run the script.

Example:

```bash
#!/bin/bash
```

Other examples:

```
#!/usr/bin/env bash
#!/usr/bin/python
```

Best practice in production:

```
#!/usr/bin/env bash
```

because it finds Bash from environment path.

---

## Execution Flow

A Bash script executes **top to bottom sequentially**.

Execution steps:

```
Read command
↓
Expand variables
↓
Execute command
↓
Return exit code
```

---

## Exit Codes

Every command returns a **status code**.

```
0 = success
non-zero = failure
```

Example:

```bash
echo $?
```

DevOps usage:

CI/CD pipelines rely heavily on exit codes.

Example:

```
0 → pipeline continues
1 → pipeline fails
```

---

## Variables

Variables store values.

Example:

```bash
NAME="DevOps"

echo $NAME
```

Best practice:

```
${VAR}
```

Example:

```bash
echo ${NAME}
```

---

## Environment Variables

Variables available globally to processes.

Examples:

```
PATH
HOME
USER
SHELL
```

Example:

```bash
export ENV=production
```

---

## Command Substitution

Execute command and store output.

Example:

```bash
DATE=$(date)
```

Old syntax:

```
`date`
```

Modern recommended syntax:

```
$(command)
```

---

## Conditional Statements

Used for decision making.

Example:

```bash
if [ $ENV = "prod" ]; then
  echo "Production"
fi
```

Important test operators:

| Operator | Meaning          |
| -------- | ---------------- |
| -f       | file exists      |
| -d       | directory exists |
| -z       | empty string     |
| -n       | non empty        |

Example:

```bash
if [ -f file.txt ]
```

---

## Loops

### For loop

Example:

```bash
for i in 1 2 3
do
  echo $i
done
```

Example used in DevOps:

```bash
for server in $(cat servers.txt)
do
  ssh $server "uptime"
done
```

---

### While loop

Example:

```bash
while true
do
  echo "running"
done
```

---

## Functions

Reusable logic.

Example:

```bash
deploy() {
  echo "Deploying application"
}

deploy
```

---

# 2. Key Terminology & Definitions

### Shell

Command interpreter that interacts with OS.

---

### Script

File containing executable commands.

---

### Shebang

Line defining interpreter.

Example:

```
#!/bin/bash
```

---

### STDIN / STDOUT / STDERR

Standard streams.

```
0 → stdin
1 → stdout
2 → stderr
```

Example:

```bash
command > output.txt
command 2> error.txt
```

---

### Pipe

Pass output from one command to another.

Example:

```bash
cat logs.txt | grep error
```

---

### Cron

Scheduler for running scripts.

Example:

```
0 2 * * * backup.sh
```

Runs every day at 2 AM.

---

# 3. Tools, Frameworks & Technologies

Bash works with many DevOps tools.

### CI/CD

Used in:

* GitHub Actions
* Jenkins pipelines
* GitLab CI

Example:

```bash
docker build -t app .
docker push repo/app
```

---

### Infrastructure Automation

Bash used with:

* Terraform
* Ansible
* Kubernetes
* AWS CLI

Example:

```bash
aws s3 cp file.txt s3://bucket
```

---

### System Monitoring

Used for:

* health checks
* disk monitoring
* log analysis

Example:

```bash
df -h
```

---

# 4. Comprehensive Interview Questions & Answers

---

# Q1: What is the difference between `sh` and `bash`?

Answer:

`sh`

* POSIX shell
* minimal features

`bash`

* extended shell
* supports arrays, advanced scripting

Example:

```
Bash supports arrays
sh does not
```

Mid-level answer includes:

* compatibility
* portability concerns.

---

# Q2: How do you debug a Bash script?

Common methods:

### Enable debug mode

```bash
set -x
```

Shows executed commands.

---

### Stop script on error

```bash
set -e
```

Stops script when command fails.

---

### Treat undefined variables as errors

```bash
set -u
```

Best practice in production scripts:

```bash
set -euo pipefail
```

---

# Q3: What is `pipefail`?

By default:

Pipeline exit status is **last command**.

Example:

```
cmd1 | cmd2 | cmd3
```

If `cmd1` fails but `cmd3` succeeds → pipeline succeeds.

`pipefail` fixes this.

```
set -o pipefail
```

Pipeline fails if any command fails.

---

# Q4: Difference between `$@` and `$*`?

Used for passing arguments.

```
"$@" → preserves argument boundaries
"$*" → merges arguments
```

Example:

```bash
script.sh a b c
```

```
"$@" → "a" "b" "c"
"$*" → "a b c"
```

---

# Q5: How would you check if a service is running?

Example script:

```bash
if systemctl is-active --quiet nginx
then
  echo "Service running"
else
  echo "Service stopped"
fi
```

---

# Q6: How do you handle errors in Bash?

Use exit codes.

Example:

```bash
if ! docker pull image; then
   echo "Pull failed"
   exit 1
fi
```

---

# 5. Code Snippets / Practical Examples

---

## Example 1: Disk Monitoring Script

```bash
#!/usr/bin/env bash

THRESHOLD=80

usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')

if [ "$usage" -gt "$THRESHOLD" ]; then
    echo "Disk usage critical"
fi
```

Used for monitoring.

---

## Example 2: Deployment Script

```bash
#!/usr/bin/env bash

APP="myapp"

docker build -t $APP .
docker stop $APP || true
docker rm $APP || true

docker run -d --name $APP -p 80:80 $APP
```

---

## Example 3: Loop Through Servers

```bash
for server in $(cat servers.txt)
do
  ssh $server "uptime"
done
```

---

# 6. Real-World DevOps Use Cases

### CI/CD Pipeline Scripts

Example:

```
Build
↓
Test
↓
Docker build
↓
Push to registry
↓
Deploy to Kubernetes
```

Bash orchestrates steps.

---

### Log Analysis

Example:

```bash
grep "ERROR" app.log | wc -l
```

---

### Automated Backups

Example:

```bash
tar -czf backup.tar.gz /data
aws s3 cp backup.tar.gz s3://backup
```

---

# 7. Performance & Edge Cases

Common issues:

### Word splitting

Bad:

```bash
for file in $(ls)
```

Correct:

```bash
for file in *
```

---

### Quoting variables

Bad:

```bash
rm $file
```

Good:

```bash
rm "$file"
```

---

### Infinite loops

Example:

```bash
while true
```

Must include exit condition.

---

# 8. Debugging & Troubleshooting

### Scenario: Script failing in CI/CD

Steps:

1. Enable debug

```
set -x
```

2. Print environment variables

```
env
```

3. Check exit codes

```
echo $?
```

4. Check permissions

```
chmod +x script.sh
```

---

### Scenario: Cron job not running

Check:

```
crontab -l
```

Check logs:

```
/var/log/syslog
```

---

# 9. Security Considerations

### Avoid command injection

Bad:

```bash
eval "$user_input"
```

---

### Sanitize inputs

Example:

```
grep -- "$pattern"
```

---

### Protect secrets

Do not hardcode credentials.

Use:

```
environment variables
vault
secret managers
```

---

# 10. Follow-Up Interview Questions

Interviewers often ask:

* How do you make Bash scripts production-ready?
* What is trap in Bash?
* Difference between subshell and shell?
* How do you parallelize scripts?
* What happens if a command inside a pipeline fails?

---

# 11. Common Mistakes & Red Flags

### Mistake 1

Using `ls` in scripts.

Bad:

```
for file in $(ls)
```

---

### Mistake 2

Ignoring exit codes.

---

### Mistake 3

No error handling.

---

### Mistake 4

Unquoted variables.

---

# 12. Behavioral Signals

Strong candidates:

* automate repetitive tasks
* think about reliability
* consider logging and observability

Example:

```
echo "$(date) Deployment started"
```

---

# 13. Evaluation Guide

### Junior

* basic commands
* simple scripts

---

### Mid-Level

* error handling
* CI/CD automation
* debugging scripts

---

### Senior

* production frameworks
* large automation pipelines
* reliability and security considerations

---

# 14. Flashcards (Quick Revision)

### How to stop script on error?

```
set -e
```

---

### Debug Bash script?

```
set -x
```

---

### Get exit code?

```
$?
```

---

### Best shebang?

```
#!/usr/bin/env bash
```

---

### Check file exists?

```
if [ -f file ]
```
---
