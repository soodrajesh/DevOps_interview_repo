# Python & Bash Coding Guide
## From Fundamentals to Interview-Ready Scripts

---

# PART 1: THE APPROACH (Before You Code)

---

## 1.1 Interview Coding Framework

When given a coding problem, follow this **5-step process**:

```
Step 1: CLARIFY (1-2 minutes)
├── What are the inputs?
├── What are the expected outputs?
├── Any edge cases? (empty input, large files, errors)
├── Any constraints? (time, memory, specific libraries)
└── Can I use external libraries?

Step 2: EXAMPLES (1 minute)
├── Work through a simple example
├── Work through an edge case
└── Confirm understanding with interviewer

Step 3: APPROACH (2-3 minutes)
├── Describe your approach in plain English
├── Identify the data structures needed
├── Estimate time/space complexity
└── Get interviewer buy-in before coding

Step 4: CODE (10-15 minutes)
├── Start with the skeleton/structure
├── Handle the main logic
├── Add error handling
└── Think out loud as you code

Step 5: TEST (2-3 minutes)
├── Walk through with your example
├── Test edge cases
├── Fix any bugs
└── Discuss potential improvements
```

## 1.2 Common SRE/DevOps Coding Tasks

```
Category 1: Log/Text Processing
- Parse log files for patterns
- Extract metrics from logs
- Find errors, count occurrences
- Transform data formats (JSON, CSV, etc.)

Category 2: System Automation
- Health checks
- Service management
- File operations
- API interactions

Category 3: Monitoring/Alerting
- Check thresholds
- Send notifications
- Aggregate metrics
- Generate reports

Category 4: Data Processing
- Process large files efficiently
- Filter and transform data
- Aggregate and summarize
- Compare datasets
```

---

# PART 2: BASH FUNDAMENTALS

---

## 2.1 Bash Script Template

**Always start with this skeleton:**

```bash
#!/usr/bin/env bash
#
# Script: script_name.sh
# Description: What this script does
# Usage: ./script_name.sh [options] <arguments>
#

# Strict mode - ALWAYS USE THIS
set -euo pipefail

# -e: Exit on any error
# -u: Error on undefined variables
# -o pipefail: Catch errors in pipes

# Constants (uppercase)
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly LOG_FILE="/tmp/${SCRIPT_NAME}.log"

# Default values (can be overridden)
VERBOSE=false
DRY_RUN=false

# Functions go here (see below)

# Main logic
main() {
    # Your code here
    echo "Hello, World!"
}

# Run main
main "$@"
```

## 2.2 Essential Bash Patterns

### Variables and Strings

```bash
# Variables - NO SPACES around =
name="value"
count=10

# Using variables - ALWAYS quote
echo "$name"
echo "${name}"

# Default values
value="${VAR:-default}"      # Use default if VAR is unset or empty
value="${VAR:=default}"      # Set VAR to default if unset or empty
value="${VAR:?error msg}"    # Exit with error if VAR is unset

# String operations
string="hello-world-test"

# Length
echo "${#string}"            # 16

# Substring
echo "${string:0:5}"         # hello (start:length)
echo "${string:6}"           # world-test (from position 6)

# Replace
echo "${string//-/_}"        # hello_world_test (replace all)
echo "${string/-/_}"         # hello_world-test (replace first)

# Remove patterns
echo "${string#*-}"          # world-test (remove shortest from start)
echo "${string##*-}"         # test (remove longest from start)
echo "${string%-*}"          # hello-world (remove shortest from end)
echo "${string%%-*}"         # hello (remove longest from end)

# Upper/lowercase (bash 4+)
echo "${string^^}"           # HELLO-WORLD-TEST
echo "${string,,}"           # hello-world-test
```

### Conditionals

```bash
# If statement syntax
if [[ condition ]]; then
    # code
elif [[ condition ]]; then
    # code
else
    # code
fi

# String comparisons (use [[ ]] for strings)
if [[ "$str" == "value" ]]; then    # Equal
if [[ "$str" != "value" ]]; then    # Not equal
if [[ "$str" =~ ^[0-9]+$ ]]; then   # Regex match
if [[ -z "$str" ]]; then            # Is empty
if [[ -n "$str" ]]; then            # Is not empty

# Numeric comparisons (use (( )) for numbers)
if (( num > 10 )); then
if (( num >= 10 )); then
if (( num == 10 )); then
if (( num != 10 )); then

# File tests
if [[ -f "$file" ]]; then           # File exists
if [[ -d "$dir" ]]; then            # Directory exists
if [[ -r "$file" ]]; then           # File is readable
if [[ -w "$file" ]]; then           # File is writable
if [[ -x "$file" ]]; then           # File is executable
if [[ -s "$file" ]]; then           # File is not empty

# Logical operators
if [[ condition1 && condition2 ]]; then   # AND
if [[ condition1 || condition2 ]]; then   # OR
if [[ ! condition ]]; then                # NOT
```

### Loops

```bash
# For loop - iterating over list
for item in apple banana cherry; do
    echo "$item"
done

# For loop - iterating over array
fruits=("apple" "banana" "cherry")
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# For loop - C-style
for ((i=0; i<10; i++)); do
    echo "$i"
done

# For loop - iterating over files
for file in /var/log/*.log; do
    echo "Processing: $file"
done

# For loop - iterating over command output
for user in $(cat users.txt); do
    echo "User: $user"
done

# While loop
count=0
while (( count < 5 )); do
    echo "$count"
    ((count++))
done

# While loop - reading file line by line (CORRECT WAY)
while IFS= read -r line; do
    echo "$line"
done < input.txt

# While loop - reading with delimiter
while IFS=: read -r user _ uid gid _ home shell; do
    echo "User: $user, UID: $uid, Home: $home"
done < /etc/passwd
```

### Arrays

```bash
# Declare array
declare -a fruits=("apple" "banana" "cherry")

# Or simply
fruits=("apple" "banana" "cherry")

# Access elements
echo "${fruits[0]}"          # apple
echo "${fruits[1]}"          # banana

# All elements
echo "${fruits[@]}"          # apple banana cherry

# Number of elements
echo "${#fruits[@]}"         # 3

# Add element
fruits+=("orange")

# Loop through array
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Loop with index
for i in "${!fruits[@]}"; do
    echo "$i: ${fruits[$i]}"
done

# Associative arrays (bash 4+)
declare -A user
user[name]="john"
user[age]=30
user[city]="NYC"

echo "${user[name]}"         # john
echo "${!user[@]}"           # Keys: name age city
echo "${user[@]}"            # Values: john 30 NYC
```

### Functions

```bash
# Function definition
function greet() {
    local name="$1"          # Local variable
    local greeting="${2:-Hello}"  # With default
    
    echo "$greeting, $name!"
    return 0                 # Return status (0 = success)
}

# Or without 'function' keyword
greet() {
    local name="$1"
    echo "Hello, $name!"
}

# Calling functions
greet "World"
greet "World" "Hi"

# Capturing output
result=$(greet "World")
echo "$result"

# Checking return status
if greet "World"; then
    echo "Greet succeeded"
else
    echo "Greet failed"
fi

# Function with validation
process_file() {
    local file="$1"
    
    # Validate input
    if [[ -z "$file" ]]; then
        echo "Error: No file specified" >&2
        return 1
    fi
    
    if [[ ! -f "$file" ]]; then
        echo "Error: File not found: $file" >&2
        return 1
    fi
    
    # Process file
    wc -l "$file"
    return 0
}
```

### Error Handling

```bash
# Exit on error (already in set -e)
# But sometimes you need to handle errors gracefully

# Method 1: || operator
command || echo "Command failed"

# Method 2: if statement
if ! command; then
    echo "Command failed"
    exit 1
fi

# Method 3: trap (cleanup on exit)
cleanup() {
    echo "Cleaning up..."
    rm -f "$TEMP_FILE"
}
trap cleanup EXIT          # Run on any exit
trap cleanup ERR           # Run on error

# Method 4: Explicitly allow failure
set +e                     # Disable exit on error
command_that_might_fail
exit_code=$?
set -e                     # Re-enable
if (( exit_code != 0 )); then
    echo "Command failed with code: $exit_code"
fi
```

### Input/Output

```bash
# Read user input
read -p "Enter your name: " name
read -sp "Enter password: " password  # Silent (no echo)
read -t 5 -p "Quick! " answer         # Timeout 5 seconds

# Output
echo "Regular output"
echo "Error output" >&2                # To stderr

# Redirect
command > file.txt                     # Stdout to file (overwrite)
command >> file.txt                    # Stdout to file (append)
command 2> error.txt                   # Stderr to file
command > out.txt 2>&1                 # Both to file
command &> all.txt                     # Both to file (shorthand)
command 2>&1 | tee output.txt          # Both to file AND screen

# Here document
cat << EOF > config.txt
server=$server
port=$port
EOF

# Here string
grep "pattern" <<< "$variable"
```

## 2.3 Common Bash Interview Tasks

### Task 1: Parse Log File and Count Errors

```bash
#!/usr/bin/env bash
# Count errors by type from log file

set -euo pipefail

LOG_FILE="${1:-/var/log/app.log}"

if [[ ! -f "$LOG_FILE" ]]; then
    echo "Error: Log file not found: $LOG_FILE" >&2
    exit 1
fi

echo "Error counts by type:"
echo "====================="

# Method 1: grep + sort + uniq (simple, readable)
grep -i "error" "$LOG_FILE" | \
    awk '{print $NF}' | \         # Last field (error type)
    sort | \
    uniq -c | \
    sort -rn | \
    head -10

# Method 2: awk only (more efficient for large files)
awk '/[Ee]rror/ {
    # Extract error type (customize based on log format)
    match($0, /Error: ([A-Za-z]+)/, arr)
    if (arr[1]) errors[arr[1]]++
}
END {
    for (e in errors) print errors[e], e
}' "$LOG_FILE" | sort -rn
```

### Task 2: Find Large Files

```bash
#!/usr/bin/env bash
# Find files larger than specified size

set -euo pipefail

DIRECTORY="${1:-.}"
SIZE="${2:-100M}"
TOP_N="${3:-10}"

echo "Top $TOP_N files larger than $SIZE in $DIRECTORY"
echo "================================================="

find "$DIRECTORY" -type f -size +"$SIZE" -exec ls -lh {} \; 2>/dev/null | \
    awk '{print $5, $9}' | \
    sort -rh | \
    head -n "$TOP_N"
```

### Task 3: Monitor Process and Alert

```bash
#!/usr/bin/env bash
# Monitor a process and alert if it dies

set -euo pipefail

PROCESS_NAME="${1:?Usage: $0 <process_name>}"
CHECK_INTERVAL="${2:-30}"
ALERT_EMAIL="oncall@example.com"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

send_alert() {
    local message="$1"
    log "ALERT: $message"
    # echo "$message" | mail -s "Process Alert: $PROCESS_NAME" "$ALERT_EMAIL"
}

check_process() {
    if pgrep -x "$PROCESS_NAME" > /dev/null; then
        return 0
    else
        return 1
    fi
}

restart_process() {
    log "Attempting to restart $PROCESS_NAME..."
    systemctl restart "$PROCESS_NAME" || return 1
    sleep 5
    check_process
}

main() {
    log "Starting monitor for: $PROCESS_NAME"
    
    while true; do
        if ! check_process; then
            send_alert "Process $PROCESS_NAME is not running!"
            
            if restart_process; then
                log "Process restarted successfully"
            else
                send_alert "Failed to restart $PROCESS_NAME - manual intervention required"
            fi
        else
            log "Process $PROCESS_NAME is healthy"
        fi
        
        sleep "$CHECK_INTERVAL"
    done
}

main
```

### Task 4: Backup Script with Rotation

```bash
#!/usr/bin/env bash
# Backup directory with rotation

set -euo pipefail

SOURCE_DIR="${1:?Usage: $0 <source_dir> <backup_dir>}"
BACKUP_DIR="${2:?Usage: $0 <source_dir> <backup_dir>}"
RETENTION_DAYS="${3:-7}"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${TIMESTAMP}.tar.gz"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# Validate inputs
if [[ ! -d "$SOURCE_DIR" ]]; then
    log "ERROR: Source directory not found: $SOURCE_DIR"
    exit 1
fi

# Create backup directory if needed
mkdir -p "$BACKUP_DIR"

# Create backup
log "Creating backup: $BACKUP_PATH"
if tar -czf "$BACKUP_PATH" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"; then
    log "Backup created successfully"
    log "Size: $(du -h "$BACKUP_PATH" | cut -f1)"
else
    log "ERROR: Backup failed"
    exit 1
fi

# Rotate old backups
log "Removing backups older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -name "backup_*.tar.gz" -type f -mtime +"$RETENTION_DAYS" -delete

# List current backups
log "Current backups:"
ls -lh "$BACKUP_DIR"/backup_*.tar.gz 2>/dev/null || log "No backups found"

log "Backup complete"
```

### Task 5: Health Check Script

```bash
#!/usr/bin/env bash
# Comprehensive health check

set -euo pipefail

# Thresholds
CPU_THRESHOLD=80
MEMORY_THRESHOLD=85
DISK_THRESHOLD=90

# Results
declare -a ISSUES=()

check_cpu() {
    local cpu_usage
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print int($2)}')
    
    if (( cpu_usage > CPU_THRESHOLD )); then
        ISSUES+=("CPU usage high: ${cpu_usage}%")
        return 1
    fi
    echo "CPU: ${cpu_usage}% ✓"
    return 0
}

check_memory() {
    local memory_usage
    memory_usage=$(free | awk '/Mem:/ {printf "%.0f", $3/$2 * 100}')
    
    if (( memory_usage > MEMORY_THRESHOLD )); then
        ISSUES+=("Memory usage high: ${memory_usage}%")
        return 1
    fi
    echo "Memory: ${memory_usage}% ✓"
    return 0
}

check_disk() {
    local disk_usage
    disk_usage=$(df -h / | awk 'NR==2 {gsub(/%/,""); print $5}')
    
    if (( disk_usage > DISK_THRESHOLD )); then
        ISSUES+=("Disk usage high: ${disk_usage}%")
        return 1
    fi
    echo "Disk: ${disk_usage}% ✓"
    return 0
}

check_service() {
    local service="$1"
    
    if systemctl is-active --quiet "$service"; then
        echo "Service $service: running ✓"
        return 0
    else
        ISSUES+=("Service $service is not running")
        return 1
    fi
}

check_url() {
    local url="$1"
    local timeout="${2:-5}"
    
    if curl -sf --max-time "$timeout" "$url" > /dev/null; then
        echo "URL $url: reachable ✓"
        return 0
    else
        ISSUES+=("URL $url is not reachable")
        return 1
    fi
}

main() {
    echo "=== System Health Check ==="
    echo "Time: $(date)"
    echo ""
    
    # Run checks (|| true prevents exit on failure)
    check_cpu || true
    check_memory || true
    check_disk || true
    check_service "nginx" || true
    check_url "http://localhost/health" || true
    
    echo ""
    
    # Report results
    if (( ${#ISSUES[@]} > 0 )); then
        echo "=== ISSUES FOUND ==="
        for issue in "${ISSUES[@]}"; do
            echo "  ❌ $issue"
        done
        exit 1
    else
        echo "=== ALL CHECKS PASSED ==="
        exit 0
    fi
}

main
```

---

# PART 3: PYTHON FUNDAMENTALS

---

## 3.1 Python Script Template

**Always start with this skeleton:**

```python
#!/usr/bin/env python3
"""
Script: script_name.py
Description: What this script does
Usage: python script_name.py [options] <arguments>
"""

import argparse
import logging
import sys
from pathlib import Path

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger(__name__)


def main():
    """Main entry point."""
    args = parse_args()
    
    try:
        # Your code here
        logger.info("Starting...")
        result = do_something(args.input)
        logger.info(f"Completed: {result}")
        
    except FileNotFoundError as e:
        logger.error(f"File not found: {e}")
        sys.exit(1)
    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        sys.exit(1)


def parse_args():
    """Parse command line arguments."""
    parser = argparse.ArgumentParser(
        description='What this script does'
    )
    parser.add_argument(
        'input',
        help='Input file or directory'
    )
    parser.add_argument(
        '-o', '--output',
        default='output.txt',
        help='Output file (default: output.txt)'
    )
    parser.add_argument(
        '-v', '--verbose',
        action='store_true',
        help='Enable verbose output'
    )
    return parser.parse_args()


def do_something(input_path):
    """Your main logic here."""
    pass


if __name__ == '__main__':
    main()
```

## 3.2 Essential Python Patterns

### Variables and Data Types

```python
# Strings
name = "hello"
name = 'hello'
multiline = """
This is a
multiline string
"""

# f-strings (Python 3.6+) - USE THESE
name = "World"
message = f"Hello, {name}!"
number = 42
formatted = f"The answer is {number:05d}"  # 00042
price = 19.99
formatted = f"Price: ${price:.2f}"          # $19.99

# Numbers
integer = 42
floating = 3.14
scientific = 1e6      # 1000000.0

# Lists (mutable, ordered)
fruits = ["apple", "banana", "cherry"]
fruits.append("orange")
fruits.extend(["grape", "melon"])
fruits.insert(0, "apricot")
fruits.remove("banana")
last = fruits.pop()

# Tuples (immutable, ordered)
point = (10, 20)
x, y = point          # Unpacking

# Dictionaries (key-value)
user = {
    "name": "John",
    "age": 30,
    "city": "NYC"
}
user["email"] = "john@example.com"
name = user.get("name", "Unknown")  # With default

# Sets (unique values)
unique = {1, 2, 3, 3, 3}  # {1, 2, 3}
unique.add(4)
unique.remove(1)

# Type hints (Python 3.5+)
def greet(name: str) -> str:
    return f"Hello, {name}"

from typing import List, Dict, Optional
def process(items: List[str]) -> Dict[str, int]:
    return {item: len(item) for item in items}
```

### Conditionals

```python
# If statement
if condition:
    do_something()
elif other_condition:
    do_other()
else:
    do_default()

# Comparison operators
x == y    # Equal
x != y    # Not equal
x > y     # Greater than
x >= y    # Greater or equal
x < y     # Less than
x <= y    # Less or equal
x is y    # Same object
x is not y
x in y    # Membership

# Logical operators
x and y
x or y
not x

# Ternary operator
result = "yes" if condition else "no"

# None checking
if value is None:
    value = default

# Better: use or for defaults (careful with falsy values)
value = value or default

# Best for None specifically
value = value if value is not None else default

# Truthiness
# Falsy: None, False, 0, 0.0, "", [], {}, set()
# Everything else is truthy

if my_list:        # Check if list is not empty
    process(my_list)
```

### Loops

```python
# For loop - list
fruits = ["apple", "banana", "cherry"]
for fruit in fruits:
    print(fruit)

# For loop with index
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")

# For loop - range
for i in range(5):           # 0, 1, 2, 3, 4
    print(i)
for i in range(2, 5):        # 2, 3, 4
    print(i)
for i in range(0, 10, 2):    # 0, 2, 4, 6, 8
    print(i)

# For loop - dictionary
user = {"name": "John", "age": 30}
for key in user:
    print(key)
for key, value in user.items():
    print(f"{key}: {value}")

# While loop
count = 0
while count < 5:
    print(count)
    count += 1

# Loop control
for i in range(10):
    if i == 3:
        continue    # Skip this iteration
    if i == 7:
        break       # Exit loop
    print(i)

# List comprehension (very Pythonic)
squares = [x**2 for x in range(10)]
evens = [x for x in range(10) if x % 2 == 0]

# Dictionary comprehension
word_lengths = {word: len(word) for word in ["hello", "world"]}

# Generator expression (memory efficient)
sum_of_squares = sum(x**2 for x in range(1000000))
```

### Functions

```python
# Basic function
def greet(name):
    return f"Hello, {name}"

# With default arguments
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}"

# With type hints
def greet(name: str, greeting: str = "Hello") -> str:
    return f"{greeting}, {name}"

# *args - variable positional arguments
def sum_all(*numbers):
    return sum(numbers)

result = sum_all(1, 2, 3, 4, 5)

# **kwargs - variable keyword arguments
def create_user(**kwargs):
    return kwargs

user = create_user(name="John", age=30, city="NYC")

# Both together
def func(*args, **kwargs):
    print(f"args: {args}")
    print(f"kwargs: {kwargs}")

# Lambda (anonymous functions)
square = lambda x: x**2
sorted_list = sorted(items, key=lambda x: x['name'])

# Docstrings
def complex_function(data: list, threshold: float = 0.5) -> dict:
    """
    Process data and return results.
    
    Args:
        data: List of items to process
        threshold: Minimum value threshold (default: 0.5)
    
    Returns:
        Dictionary with processed results
    
    Raises:
        ValueError: If data is empty
    """
    if not data:
        raise ValueError("Data cannot be empty")
    return {"result": "processed"}
```

### Error Handling

```python
# Basic try/except
try:
    result = risky_operation()
except SomeError:
    handle_error()

# Multiple exceptions
try:
    result = risky_operation()
except FileNotFoundError:
    print("File not found")
except PermissionError:
    print("Permission denied")
except Exception as e:
    print(f"Unexpected error: {e}")

# With else and finally
try:
    file = open("data.txt")
    data = file.read()
except FileNotFoundError:
    data = None
else:
    # Runs if no exception
    process(data)
finally:
    # Always runs
    file.close()

# Context manager (better for resources)
with open("data.txt") as file:
    data = file.read()
# File automatically closed

# Raising exceptions
def validate(value):
    if value < 0:
        raise ValueError(f"Value must be positive: {value}")
    return value

# Custom exceptions
class MyAppError(Exception):
    """Custom application error."""
    pass

class ValidationError(MyAppError):
    """Validation failed."""
    pass
```

### File Operations

```python
from pathlib import Path

# Reading files
# Method 1: Basic (remember to close)
file = open("data.txt", "r")
content = file.read()
file.close()

# Method 2: Context manager (preferred)
with open("data.txt", "r") as file:
    content = file.read()

# Read lines
with open("data.txt", "r") as file:
    lines = file.readlines()           # List of lines
    
with open("data.txt", "r") as file:
    for line in file:                  # Memory efficient
        process(line.strip())

# Writing files
with open("output.txt", "w") as file:  # Overwrite
    file.write("Hello\n")

with open("output.txt", "a") as file:  # Append
    file.write("World\n")

# Using pathlib (modern, preferred)
path = Path("data.txt")

if path.exists():
    content = path.read_text()
    
path.write_text("Hello, World!")

# File operations with pathlib
path = Path("/var/log/app.log")
path.name        # "app.log"
path.stem        # "app"
path.suffix      # ".log"
path.parent      # Path("/var/log")
path.is_file()   # True/False
path.is_dir()    # True/False

# List directory contents
for file in Path("/var/log").iterdir():
    print(file)

# Glob patterns
for log_file in Path("/var/log").glob("*.log"):
    print(log_file)

# Recursive glob
for py_file in Path(".").rglob("*.py"):
    print(py_file)
```

### Working with JSON

```python
import json

# Parse JSON string
json_string = '{"name": "John", "age": 30}'
data = json.loads(json_string)

# Convert to JSON string
data = {"name": "John", "age": 30}
json_string = json.dumps(data)
json_pretty = json.dumps(data, indent=2)

# Read JSON file
with open("data.json", "r") as file:
    data = json.load(file)

# Write JSON file
with open("output.json", "w") as file:
    json.dump(data, file, indent=2)
```

### Regular Expressions

```python
import re

text = "Error: Connection failed at 10:30:45"

# Search (first match)
match = re.search(r'\d+:\d+:\d+', text)
if match:
    print(match.group())  # "10:30:45"

# Find all matches
times = re.findall(r'\d+:\d+:\d+', text)

# Match groups
pattern = r'Error: (\w+) at (\d+:\d+:\d+)'
match = re.search(pattern, text)
if match:
    error_type = match.group(1)    # "Connection"
    time = match.group(2)          # "10:30:45"
    all_groups = match.groups()    # ("Connection", "10:30:45")

# Named groups
pattern = r'Error: (?P<type>\w+) at (?P<time>\d+:\d+:\d+)'
match = re.search(pattern, text)
if match:
    error_type = match.group('type')
    time = match.group('time')

# Replace
cleaned = re.sub(r'\d+', 'X', text)  # Replace all numbers

# Split
parts = re.split(r'[,\s]+', "a, b  c,d")  # ['a', 'b', 'c', 'd']

# Compile for reuse (more efficient)
pattern = re.compile(r'\d+:\d+:\d+')
matches = pattern.findall(text)
```

### HTTP Requests

```python
import requests

# GET request
response = requests.get('https://api.example.com/users')
if response.status_code == 200:
    data = response.json()
    
# With parameters
response = requests.get(
    'https://api.example.com/users',
    params={'page': 1, 'limit': 10}
)

# With headers
response = requests.get(
    'https://api.example.com/users',
    headers={'Authorization': 'Bearer token123'}
)

# POST request
response = requests.post(
    'https://api.example.com/users',
    json={'name': 'John', 'email': 'john@example.com'},
    headers={'Content-Type': 'application/json'}
)

# With timeout
response = requests.get(
    'https://api.example.com/users',
    timeout=10  # seconds
)

# Error handling
try:
    response = requests.get(url, timeout=10)
    response.raise_for_status()  # Raises exception for 4xx/5xx
    data = response.json()
except requests.exceptions.Timeout:
    print("Request timed out")
except requests.exceptions.HTTPError as e:
    print(f"HTTP error: {e}")
except requests.exceptions.RequestException as e:
    print(f"Request failed: {e}")
```

## 3.3 Common Python Interview Tasks

### Task 1: Parse Log File and Count Errors

```python
#!/usr/bin/env python3
"""Parse log file and count errors by type."""

import re
import sys
from collections import Counter
from pathlib import Path


def parse_log_file(log_path: str) -> Counter:
    """
    Parse log file and count errors by type.
    
    Args:
        log_path: Path to log file
        
    Returns:
        Counter with error types and counts
    """
    error_pattern = re.compile(r'ERROR[:\s]+(\w+)', re.IGNORECASE)
    error_counts = Counter()
    
    path = Path(log_path)
    if not path.exists():
        raise FileNotFoundError(f"Log file not found: {log_path}")
    
    with open(path, 'r') as file:
        for line in file:
            match = error_pattern.search(line)
            if match:
                error_type = match.group(1)
                error_counts[error_type] += 1
    
    return error_counts


def main():
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <log_file>")
        sys.exit(1)
    
    log_file = sys.argv[1]
    
    try:
        error_counts = parse_log_file(log_file)
        
        print("Error counts by type:")
        print("=" * 40)
        
        for error_type, count in error_counts.most_common(10):
            print(f"{error_type:<30} {count:>5}")
            
        print("=" * 40)
        print(f"{'Total':<30} {sum(error_counts.values()):>5}")
        
    except FileNotFoundError as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)


if __name__ == '__main__':
    main()
```

### Task 2: Monitor Endpoint and Alert

```python
#!/usr/bin/env python3
"""Monitor HTTP endpoint and alert on failures."""

import argparse
import logging
import time
import requests
from dataclasses import dataclass
from typing import Optional

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger(__name__)


@dataclass
class HealthCheck:
    url: str
    timeout: int = 5
    expected_status: int = 200
    expected_text: Optional[str] = None


@dataclass  
class CheckResult:
    success: bool
    status_code: Optional[int] = None
    response_time: Optional[float] = None
    error: Optional[str] = None


def check_endpoint(health_check: HealthCheck) -> CheckResult:
    """Check if endpoint is healthy."""
    try:
        start_time = time.time()
        response = requests.get(
            health_check.url,
            timeout=health_check.timeout
        )
        response_time = time.time() - start_time
        
        # Check status code
        if response.status_code != health_check.expected_status:
            return CheckResult(
                success=False,
                status_code=response.status_code,
                response_time=response_time,
                error=f"Unexpected status: {response.status_code}"
            )
        
        # Check response text if specified
        if health_check.expected_text:
            if health_check.expected_text not in response.text:
                return CheckResult(
                    success=False,
                    status_code=response.status_code,
                    response_time=response_time,
                    error=f"Expected text not found: {health_check.expected_text}"
                )
        
        return CheckResult(
            success=True,
            status_code=response.status_code,
            response_time=response_time
        )
        
    except requests.exceptions.Timeout:
        return CheckResult(success=False, error="Request timed out")
    except requests.exceptions.ConnectionError:
        return CheckResult(success=False, error="Connection failed")
    except Exception as e:
        return CheckResult(success=False, error=str(e))


def send_alert(message: str):
    """Send alert (implement your preferred method)."""
    logger.warning(f"ALERT: {message}")
    # Could add: Slack webhook, email, PagerDuty, etc.


def monitor(health_check: HealthCheck, interval: int = 30, 
            failure_threshold: int = 3):
    """Continuously monitor endpoint."""
    consecutive_failures = 0
    
    logger.info(f"Starting monitor for {health_check.url}")
    
    while True:
        result = check_endpoint(health_check)
        
        if result.success:
            consecutive_failures = 0
            logger.info(
                f"OK - {health_check.url} - "
                f"Status: {result.status_code}, "
                f"Time: {result.response_time:.3f}s"
            )
        else:
            consecutive_failures += 1
            logger.error(
                f"FAIL - {health_check.url} - "
                f"Error: {result.error} "
                f"({consecutive_failures}/{failure_threshold})"
            )
            
            if consecutive_failures >= failure_threshold:
                send_alert(
                    f"Endpoint {health_check.url} has failed "
                    f"{consecutive_failures} times: {result.error}"
                )
        
        time.sleep(interval)


def main():
    parser = argparse.ArgumentParser(description='Monitor HTTP endpoint')
    parser.add_argument('url', help='URL to monitor')
    parser.add_argument('-i', '--interval', type=int, default=30,
                        help='Check interval in seconds')
    parser.add_argument('-t', '--timeout', type=int, default=5,
                        help='Request timeout in seconds')
    parser.add_argument('-f', '--failures', type=int, default=3,
                        help='Consecutive failures before alert')
    
    args = parser.parse_args()
    
    health_check = HealthCheck(
        url=args.url,
        timeout=args.timeout
    )
    
    try:
        monitor(health_check, args.interval, args.failures)
    except KeyboardInterrupt:
        logger.info("Monitor stopped")


if __name__ == '__main__':
    main()
```

### Task 3: Process CSV and Generate Report

```python
#!/usr/bin/env python3
"""Process CSV file and generate summary report."""

import csv
import sys
from collections import defaultdict
from pathlib import Path
from typing import Dict, List


def read_csv(file_path: str) -> List[Dict]:
    """Read CSV file and return list of dictionaries."""
    path = Path(file_path)
    if not path.exists():
        raise FileNotFoundError(f"File not found: {file_path}")
    
    with open(path, 'r', newline='') as file:
        reader = csv.DictReader(file)
        return list(reader)


def analyze_data(data: List[Dict], group_by: str, 
                 sum_field: str) -> Dict[str, float]:
    """Group data and sum values."""
    results = defaultdict(float)
    
    for row in data:
        key = row.get(group_by, 'Unknown')
        try:
            value = float(row.get(sum_field, 0))
            results[key] += value
        except ValueError:
            continue
    
    return dict(sorted(results.items(), key=lambda x: x[1], reverse=True))


def generate_report(data: List[Dict], analysis: Dict[str, float],
                    group_by: str, sum_field: str) -> str:
    """Generate text report."""
    total = sum(analysis.values())
    
    report = []
    report.append("=" * 50)
    report.append(f"DATA ANALYSIS REPORT")
    report.append("=" * 50)
    report.append(f"Total records: {len(data)}")
    report.append(f"Total {sum_field}: {total:,.2f}")
    report.append("")
    report.append(f"Breakdown by {group_by}:")
    report.append("-" * 50)
    
    for key, value in analysis.items():
        percentage = (value / total * 100) if total > 0 else 0
        report.append(f"{key:<30} {value:>10,.2f} ({percentage:>5.1f}%)")
    
    report.append("=" * 50)
    
    return "\n".join(report)


def main():
    if len(sys.argv) < 4:
        print(f"Usage: {sys.argv[0]} <csv_file> <group_by_field> <sum_field>")
        print(f"Example: {sys.argv[0]} sales.csv region amount")
        sys.exit(1)
    
    csv_file = sys.argv[1]
    group_by = sys.argv[2]
    sum_field = sys.argv[3]
    
    try:
        data = read_csv(csv_file)
        
        if not data:
            print("No data found in file")
            sys.exit(1)
        
        # Validate fields exist
        sample_row = data[0]
        if group_by not in sample_row:
            print(f"Field '{group_by}' not found. Available: {list(sample_row.keys())}")
            sys.exit(1)
        if sum_field not in sample_row:
            print(f"Field '{sum_field}' not found. Available: {list(sample_row.keys())}")
            sys.exit(1)
        
        analysis = analyze_data(data, group_by, sum_field)
        report = generate_report(data, analysis, group_by, sum_field)
        
        print(report)
        
    except FileNotFoundError as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"Error processing file: {e}", file=sys.stderr)
        sys.exit(1)


if __name__ == '__main__':
    main()
```

### Task 4: Find and Kill Zombie Processes

```python
#!/usr/bin/env python3
"""Find zombie processes and optionally kill their parents."""

import subprocess
import argparse
import logging
import os
import signal
from dataclasses import dataclass
from typing import List, Optional

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger(__name__)


@dataclass
class ZombieProcess:
    pid: int
    ppid: int
    user: str
    command: str


def find_zombies() -> List[ZombieProcess]:
    """Find all zombie processes."""
    zombies = []
    
    try:
        # Get process list
        result = subprocess.run(
            ['ps', 'aux'],
            capture_output=True,
            text=True,
            check=True
        )
        
        for line in result.stdout.strip().split('\n')[1:]:  # Skip header
            parts = line.split(None, 10)  # Split into max 11 parts
            if len(parts) >= 11:
                stat = parts[7]  # STAT column
                if 'Z' in stat:  # Zombie
                    pid = int(parts[1])
                    user = parts[0]
                    command = parts[10]
                    
                    # Get parent PID
                    ppid = get_ppid(pid)
                    
                    zombies.append(ZombieProcess(
                        pid=pid,
                        ppid=ppid,
                        user=user,
                        command=command
                    ))
                    
    except subprocess.CalledProcessError as e:
        logger.error(f"Failed to get process list: {e}")
    
    return zombies


def get_ppid(pid: int) -> Optional[int]:
    """Get parent PID of a process."""
    try:
        with open(f'/proc/{pid}/status', 'r') as f:
            for line in f:
                if line.startswith('PPid:'):
                    return int(line.split()[1])
    except (FileNotFoundError, PermissionError):
        pass
    return None


def kill_parent(ppid: int, force: bool = False) -> bool:
    """Kill parent process to clean up zombie."""
    if ppid is None or ppid <= 1:
        logger.warning(f"Cannot kill PID {ppid}")
        return False
    
    try:
        sig = signal.SIGKILL if force else signal.SIGTERM
        os.kill(ppid, sig)
        logger.info(f"Sent {sig.name} to PID {ppid}")
        return True
    except PermissionError:
        logger.error(f"Permission denied to kill PID {ppid}")
        return False
    except ProcessLookupError:
        logger.warning(f"Process {ppid} not found")
        return False


def main():
    parser = argparse.ArgumentParser(
        description='Find and optionally clean up zombie processes'
    )
    parser.add_argument(
        '--kill', action='store_true',
        help='Kill parent processes to clean up zombies'
    )
    parser.add_argument(
        '--force', action='store_true',
        help='Use SIGKILL instead of SIGTERM'
    )
    args = parser.parse_args()
    
    zombies = find_zombies()
    
    if not zombies:
        logger.info("No zombie processes found")
        return
    
    logger.info(f"Found {len(zombies)} zombie process(es)")
    print("\n" + "=" * 70)
    print(f"{'PID':<8} {'PPID':<8} {'USER':<12} {'COMMAND'}")
    print("=" * 70)
    
    for z in zombies:
        print(f"{z.pid:<8} {z.ppid:<8} {z.user:<12} {z.command[:40]}")
    
    print("=" * 70)
    
    if args.kill:
        # Get unique parent PIDs
        parent_pids = set(z.ppid for z in zombies if z.ppid)
        
        for ppid in parent_pids:
            if input(f"Kill parent PID {ppid}? [y/N] ").lower() == 'y':
                kill_parent(ppid, args.force)
    else:
        print("\nTo clean up zombies, run with --kill flag")


if __name__ == '__main__':
    main()
```

### Task 5: API Health Checker with Retry

```python
#!/usr/bin/env python3
"""Check multiple API endpoints with retry logic."""

import time
import json
import requests
import concurrent.futures
from dataclasses import dataclass, asdict
from typing import List, Optional
from functools import wraps


def retry(max_attempts: int = 3, delay: float = 1.0, backoff: float = 2.0):
    """Retry decorator with exponential backoff."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            current_delay = delay
            
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    if attempt < max_attempts - 1:
                        time.sleep(current_delay)
                        current_delay *= backoff
            
            raise last_exception
        return wrapper
    return decorator


@dataclass
class Endpoint:
    name: str
    url: str
    method: str = 'GET'
    timeout: int = 10
    expected_status: int = 200


@dataclass
class HealthResult:
    name: str
    url: str
    healthy: bool
    status_code: Optional[int] = None
    response_time_ms: Optional[float] = None
    error: Optional[str] = None


@retry(max_attempts=3, delay=1.0)
def check_endpoint(endpoint: Endpoint) -> HealthResult:
    """Check single endpoint health."""
    start_time = time.time()
    
    try:
        response = requests.request(
            method=endpoint.method,
            url=endpoint.url,
            timeout=endpoint.timeout
        )
        response_time = (time.time() - start_time) * 1000
        
        healthy = response.status_code == endpoint.expected_status
        
        return HealthResult(
            name=endpoint.name,
            url=endpoint.url,
            healthy=healthy,
            status_code=response.status_code,
            response_time_ms=round(response_time, 2),
            error=None if healthy else f"Status {response.status_code}"
        )
        
    except requests.exceptions.Timeout:
        return HealthResult(
            name=endpoint.name,
            url=endpoint.url,
            healthy=False,
            error="Timeout"
        )
    except requests.exceptions.ConnectionError:
        return HealthResult(
            name=endpoint.name,
            url=endpoint.url,
            healthy=False,
            error="Connection failed"
        )


def check_all_endpoints(endpoints: List[Endpoint], 
                        max_workers: int = 5) -> List[HealthResult]:
    """Check all endpoints in parallel."""
    results = []
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_to_endpoint = {
            executor.submit(check_endpoint, ep): ep 
            for ep in endpoints
        }
        
        for future in concurrent.futures.as_completed(future_to_endpoint):
            try:
                result = future.result()
                results.append(result)
            except Exception as e:
                endpoint = future_to_endpoint[future]
                results.append(HealthResult(
                    name=endpoint.name,
                    url=endpoint.url,
                    healthy=False,
                    error=str(e)
                ))
    
    return results


def print_report(results: List[HealthResult]):
    """Print health check report."""
    print("\n" + "=" * 80)
    print("API HEALTH CHECK REPORT")
    print("=" * 80)
    
    healthy_count = sum(1 for r in results if r.healthy)
    total_count = len(results)
    
    for result in sorted(results, key=lambda x: x.healthy):
        status = "✓" if result.healthy else "✗"
        time_str = f"{result.response_time_ms}ms" if result.response_time_ms else "N/A"
        error_str = f" ({result.error})" if result.error else ""
        
        print(f"{status} {result.name:<30} {time_str:<10} {error_str}")
    
    print("=" * 80)
    print(f"Summary: {healthy_count}/{total_count} endpoints healthy")
    
    return healthy_count == total_count


def main():
    # Define endpoints to check
    endpoints = [
        Endpoint(name="Google", url="https://www.google.com"),
        Endpoint(name="GitHub", url="https://api.github.com"),
        Endpoint(name="JSONPlaceholder", url="https://jsonplaceholder.typicode.com/posts/1"),
        Endpoint(name="HTTPBin", url="https://httpbin.org/get"),
        # Add your endpoints here
    ]
    
    results = check_all_endpoints(endpoints)
    all_healthy = print_report(results)
    
    # Output JSON for further processing
    json_output = [asdict(r) for r in results]
    print("\nJSON Output:")
    print(json.dumps(json_output, indent=2))
    
    # Exit code based on health
    exit(0 if all_healthy else 1)


if __name__ == '__main__':
    main()
```

---

# PART 4: QUICK REFERENCE CHEAT SHEETS

---

## Bash Cheat Sheet

```bash
# Variables
var="value"                  # Assign (no spaces!)
echo "$var"                  # Use (always quote)
readonly CONST="value"       # Constant
export VAR="value"           # Environment variable

# String operations
${#var}                      # Length
${var:0:5}                   # Substring
${var/old/new}               # Replace first
${var//old/new}              # Replace all
${var#pattern}               # Remove prefix
${var%pattern}               # Remove suffix

# Conditionals
[[ -z "$var" ]]              # Is empty
[[ -n "$var" ]]              # Not empty
[[ "$a" == "$b" ]]           # String equal
[[ "$a" =~ regex ]]          # Regex match
(( num > 10 ))               # Numeric comparison
[[ -f "$file" ]]             # File exists
[[ -d "$dir" ]]              # Directory exists

# Loops
for item in "${array[@]}"; do ...; done
for i in {1..10}; do ...; done
for ((i=0; i<10; i++)); do ...; done
while read -r line; do ...; done < file
while (( count < 5 )); do ...; done

# Functions
func() { local var="$1"; echo "$var"; }
result=$(func "arg")

# Error handling
set -euo pipefail            # Strict mode
command || echo "failed"     # On error
trap cleanup EXIT            # Cleanup on exit

# Common commands
grep -r "pattern" /path      # Search recursively
find /path -name "*.log"     # Find files
awk '{print $1}' file        # Print first column
sed 's/old/new/g' file       # Replace text
sort | uniq -c               # Count unique
xargs -I {} cmd {}           # Execute for each
```

## Python Cheat Sheet

```python
# Variables & Types
name = "string"
num = 42
items = [1, 2, 3]            # List
data = {"key": "value"}      # Dict
unique = {1, 2, 3}           # Set

# String formatting (f-strings)
f"Hello, {name}!"
f"Value: {num:05d}"          # Zero-padded
f"Price: ${price:.2f}"       # 2 decimals

# List operations
items.append(4)              # Add to end
items.extend([5, 6])         # Add multiple
items.insert(0, 0)           # Insert at index
items.pop()                  # Remove last
items.remove(2)              # Remove by value
[x for x in items if x > 2]  # List comprehension

# Dict operations
data.get("key", "default")   # Get with default
data.keys()                  # All keys
data.values()                # All values
data.items()                 # Key-value pairs
{k: v for k, v in items}     # Dict comprehension

# File operations
with open("file.txt") as f:
    content = f.read()       # Read all
    lines = f.readlines()    # Read lines
    for line in f:           # Iterate lines
        process(line)

Path("file.txt").read_text() # Pathlib read
Path("file.txt").write_text("content")

# Error handling
try:
    risky_code()
except SomeError as e:
    handle_error(e)
finally:
    cleanup()

# Common imports
from pathlib import Path
from collections import Counter, defaultdict
from typing import List, Dict, Optional
import json, csv, re, sys, os
import logging, argparse
import requests
```

## Common Interview Patterns

```python
# Count occurrences
from collections import Counter
counts = Counter(items)
most_common = counts.most_common(10)

# Group by key
from collections import defaultdict
grouped = defaultdict(list)
for item in items:
    grouped[item.category].append(item)

# Process file line by line (memory efficient)
with open("large_file.txt") as f:
    for line in f:
        process(line.strip())

# Parallel execution
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(process, items))

# Retry with backoff
import time
def retry(func, max_attempts=3, delay=1):
    for i in range(max_attempts):
        try:
            return func()
        except Exception:
            if i < max_attempts - 1:
                time.sleep(delay * (2 ** i))
            else:
                raise

# HTTP request with error handling
import requests
try:
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    data = response.json()
except requests.RequestException as e:
    print(f"Request failed: {e}")
```

---

*Use these templates and patterns as building blocks. Practice by solving problems on paper first, then implement. Remember: clarity > cleverness in interviews.*