# SHELL SCRIPTING: FUNCTIONS & INTERMEDIATE CONCEPTS.

## Task 1: Basic Functions.
`vim functions.sh`

```bash
#!/bin/bash

# greet function
greet() {
    echo "Hello, $1!"
}

# add function
add() {
    local sum=$(( $1 + $2 ))
    echo "Sum: $sum"
}

# Call functions
greet "Atul"
add 5 7
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/005715a512d9566e431010284c0bbae949df9e12/2026/day-18/Screenshots/Screenshot%20(533).png)

## Task 2: Functions with Return Values
`vim disk_check.sh`

```bash
#!/bin/bash

check_disk() {
    df -h /
}

check_memory() {
    free -h
}

# Main
echo "=== Disk Usage ==="
check_disk

echo "=== Memory Usage ==="
check_memory
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/005715a512d9566e431010284c0bbae949df9e12/2026/day-18/Screenshots/Screenshot%20(534).png)

## Task 3: Strict Mode
`vim strict_demo.sh`

```bash
#!/bin/bash
set -euo pipefail

echo "Testing strict mode..."

# Undefined variable (set -u will cause error)
echo "Value: $UNDEFINED_VAR"

# Command that fails (set -e will exit)
false

# Pipe failure (set -o pipefail will propagate error)
ls /nonexistent | grep "test"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/005715a512d9566e431010284c0bbae949df9e12/2026/day-18/Screenshots/Screenshot%20(535).png)

- Documentation<br>
    - set -e → Exit immediately if a command fails

    - set -u → Treat unset variables as errors

    - set -o pipefail → Return error if any command in a pipeline fails

## Task 4: Local Variables
`vim local_demo.sh`

```bash
#!/bin/bash

with_local() {
    local msg="I am local"
    echo "Inside function: $msg"
}

without_local() {
    msg="I am global"
    echo "Inside function: $msg"
}

with_local
echo "Outside after with_local: ${msg:-not defined}"

without_local
echo "Outside after without_local: $msg"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/005715a512d9566e431010284c0bbae949df9e12/2026/day-18/Screenshots/Screenshot%20(536).png)

## Task 5: System Info Reporter
`vim system_info.sh`

```bash
#!/bin/bash
set -euo pipefail

print_system_info() {
    echo "=== System Info ==="
    hostnamectl
}

print_uptime() {
    echo "=== Uptime ==="
    uptime
}

print_disk_usage() {
    echo "=== Disk Usage (Top 5) ==="
    df -h | sort -k3 -r | head -n 5
}

print_memory_usage() {
    echo "=== Memory Usage ==="
    free -h
}

print_top_cpu() {
    echo "=== Top 5 CPU Processes ==="
    ps -eo pid,comm,%cpu --sort=-%cpu | head -n 6
}

main() {
    print_system_info
    print_uptime
    print_disk_usage
    print_memory_usage
    print_top_cpu
}

main
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/005715a512d9566e431010284c0bbae949df9e12/2026/day-18/Screenshots/Screenshot%20(537).png)

## Key Learnings
1. Functions make scripts modular and reusable.

2. Strict mode (set -euo pipefail) prevents silent failures.

3. Local variables avoid accidental overwrites and keep scope clean.