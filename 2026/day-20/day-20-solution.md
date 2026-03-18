# BASH SCRIPTING CHALLENGE: LOG ANALYZER AND REPORT GENERATOR.
`vim log_analyzer.sh`

```bash
#!/bin/bash

# log_analyzer.sh - Automates log analysis and generates a daily summary report

# -------------------------------
# Task 1: Input and Validation
# -------------------------------
if [ $# -eq 0 ]; then
    echo "Error: No log file provided."
    echo "Usage: $0 <logfile>"
    exit 1
fi

LOGFILE=$1

if [ ! -f "$LOGFILE" ]; then
    echo "Error: File '$LOGFILE' does not exist."
    exit 1
fi

DATE=$(date +%Y-%m-%d)
REPORT="log_report_${DATE}.txt"

# -------------------------------
# Task 2: Error Count
# -------------------------------
ERROR_COUNT=$(grep -i -E "ERROR|Failed" "$LOGFILE" | wc -l)
echo "Total error count: $ERROR_COUNT"

# -------------------------------
# Task 3: Critical Events
# -------------------------------
CRITICAL_EVENTS=$(grep -n "CRITICAL" "$LOGFILE")
echo "--- Critical Events ---"
echo "$CRITICAL_EVENTS"

# -------------------------------
# Task 4: Top Error Messages
# -------------------------------
TOP_ERRORS=$(grep "ERROR" "$LOGFILE" | awk '{$1=$2=$3=""; print}' | sort | uniq -c | sort -rn | head -5)
echo "--- Top 5 Error Messages ---"
echo "$TOP_ERRORS"

# -------------------------------
# Task 5: Summary Report
# -------------------------------
TOTAL_LINES=$(wc -l < "$LOGFILE")

{
    echo "Log Analysis Report - $DATE"
    echo "Log File: $LOGFILE"
    echo "Total Lines Processed: $TOTAL_LINES"
    echo "Total Error Count: $ERROR_COUNT"
    echo ""
    echo "--- Top 5 Error Messages ---"
    echo "$TOP_ERRORS"
    echo ""
    echo "--- Critical Events ---"
    echo "$CRITICAL_EVENTS"
} > "$REPORT"

echo "Summary report generated: $REPORT"

# -------------------------------
# Task 6 (Optional): Archive Logs
# -------------------------------
if [ ! -d "archive" ]; then
    mkdir archive
fi

mv "$LOGFILE" archive/
echo "Log file archived to ./archive/"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/31afcdbf34c2556e537622f3a1f7d5b3f8537221/2026/day-20/Screenshots/Screenshot%20(50).png)

- Shebang
    ```bash
    #!/bin/bash
    ```
    - This tells the system to run the script using the **Bash shell**.
    - Without it, the script might run in a different shell (like `sh`), which could cause errors.

- Script Description
    ```bash
    # log_analyzer.sh - Automates log analysis and generates a daily summary report
    ```
    - A comment describing the purpose of the script.
    - Comments (`#`) are ignored by Bash but help humans understand the code.

-  Task 01 - Input Validation
    ```bash
    if [ $# -eq 0 ]; then
        echo "Error: No log file provided."
        echo "Usage: $0 <logfile>"
        exit 1
    fi
    ```
    - `$#` - number of arguments passed to the script.
    - If **no arguments** are given, print an error and exit.
    - `$0` - the script name itself, so `Usage: $0 <logfile>` shows how to run it.
    ```bash
    LOGFILE=$1
    ```
    - `$1` - the first argument (the log file path).
    - Assign it to `LOGFILE` for easier reference.
    ```bash
    if [ ! -f "$LOGFILE" ]; then
        echo "Error: File '$LOGFILE' does not exist."
        exit 1
    fi
    ```
    - `-f` checks if the file exists.
    - If not, print an error and exit.

- Date and Report File
    ```bash
    DATE=$(date +%Y-%m-%d)
    REPORT="log_report_${DATE}.txt"
    ```
    - `date +%Y-%m-%d` - current date in `YYYY-MM-DD` format.
    - `REPORT` - name of the summary report file, e.g., `log_report_2026-03-06.txt`.

- Task 02 - Error Count
    ```bash
    ERROR_COUNT=$(grep -i -E "ERROR|Failed" "$LOGFILE" | wc -l)
    echo "Total error count: $ERROR_COUNT"
    ```
    - `grep -i -E "ERROR|Failed"` - search for lines containing `ERROR` or `Failed` (case-insensitive).
    - `wc -l` - count the number of matching lines.
    - Store result in `ERROR_COUNT` and print it.

- Task 03 - Critical Events
    ```bash
    CRITICAL_EVENTS=$(grep -n "CRITICAL" "$LOGFILE")
    echo "--- Critical Events ---"
    echo "$CRITICAL_EVENTS"
    ```
    - `grep -n "CRITICAL"` - find lines with `CRITICAL` and include line numbers.
    - Store in `CRITICAL_EVENTS` and print.

- Task 04 - Top Error Messages
    ```bash
    TOP_ERRORS=$(grep "ERROR" "$LOGFILE" | awk '{$1=$2=$3=""; print}' | sort | uniq -c | sort -rn | head -5)
    echo "--- Top 5 Error Messages ---"
    echo "$TOP_ERRORS"
    ```
    - `grep "ERROR"` - extract lines with ERROR.
    - `awk '{$1=$2=$3=""; print}'` - remove the first 3 fields (usually timestamp + log level) so only the error message remains.
    - `sort` - sort messages alphabetically.
    - `uniq -c` - count occurrences of each unique message.
    - `sort -rn` - sort numerically in reverse (highest count first).
    - `head -5` - show top 5.
    - Store in `TOP_ERRORS` and print.

- Task 05 - Summary Report
    ```bash
    TOTAL_LINES=$(wc -l < "$LOGFILE")
    ```
    - Count total lines in the log file.
    ```bash
    {
        echo "Log Analysis Report - $DATE"
        echo "Log File: $LOGFILE"
        echo "Total Lines Processed: $TOTAL_LINES"
        echo "Total Error Count: $ERROR_COUNT"
        echo ""
        echo "--- Top 5 Error Messages ---"
        echo "$TOP_ERRORS"
        echo ""
        echo "--- Critical Events ---"
        echo "$CRITICAL_EVENTS"
    } > "$REPORT"
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/31afcdbf34c2556e537622f3a1f7d5b3f8537221/2026/day-20/Screenshots/Screenshot%20(52).png)

    - Use `{ ... } > "$REPORT"` to write multiple lines into the report file.
    - Includes:
        - Date
        - Log file name
        - Total lines processed
        - Error count
        - Top 5 error messages
        - Critical events
    ```bash
    echo "Summary report generated: $REPORT"
    ```
    - Confirmation message.

- Task 06 - Archive Logs (Optional)
    ```bash
    if [ ! -d "archive" ]; then
        mkdir archive
    fi
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/31afcdbf34c2556e537622f3a1f7d5b3f8537221/2026/day-20/Screenshots/Screenshot%20(51).png)
    
    - Check if `archive/` directory exists.
    - If not, create it.
    ```bash
    mv "$REPORT" archive/
    echo "Precessed log archived to ./archive/"
    ```
    - Move the processed log file into `archive/`.
    - Print confirmation.

## Key Learnings
1. **Log parsing efficiency:** Combining grep, awk, and sort makes error analysis fast and scriptable.
2. **Automation best practices:** Input validation and archiving ensure reliability and maintainability.
3. **Reusable workflows:** This script can be adapted for production-like logs (Apache, syslogs, HDFS) with minimal changes.