# bash-scripting-project


#!/bin/bash

# Timestamp and log name
timestamp=$(date +"%Y-%m-%d_%H-%M-%S")
log_tag="system_report"
report_name="system_report_$timestamp.txt"

# Try to use Desktop, fallback to /tmp
if [ -d "$HOME/Desktop" ]; then
    user_desktop="$HOME/Desktop"
else
    user_desktop="/tmp"
fi

report_file="$user_desktop/$report_name"

disk_threshold=80

# Function to log and echo a message
log_msg() {
    echo "$1" | tee -a "$report_file"
    logger -t "$log_tag" "$1"
}

# Function to print section headers
print_section() {
    log_msg ""
    log_msg "================== $1 =================="
    log_msg ""
}

# Create report file or exit
touch "$report_file" 2>/dev/null
if [ $? -ne 0 ]; then
    logger -t "$log_tag" "❌ Failed to create $report_file"
    echo "❌ Failed to create report file at $report_file"
    exit 1
fi

logger -t "$log_tag" "📄 Starting report: $report_name"

############################################
# 1. System Information
############################################
print_section "System Information"
log_msg "Hostname: $(hostname)"
log_msg "IP Address: $(hostname -I | awk '{print $1}')"
log_msg "Uptime: $(uptime -p)"
log_msg "Kernel Version: $(uname -r)"

############################################
# 2. Disk Usage Check
############################################
print_section "Disk Usage Check"
df -h >> "$report_file"

print_section "Disk Usage Alerts"
df -hP | awk -v threshold="$disk_threshold" '
    NR>1 {
        usage = substr($5, 1, length($5)-1);
        if (usage+0 >= threshold)
            print "WARNING: " $6 " is at " $5 " usage.";
    }
' | tee -a "$report_file" | while read -r line; do logger -t "$log_tag" "$line"; done

############################################
# 3. Logged-in Users & Security Check
############################################
print_section "Logged-in Users"
who | tee -a "$report_file" | while read -r line; do logger -t "$log_tag" "$line"; done

print_section "Accounts With Empty Passwords (Potential Security Risk)"
awk -F: '($2 == "" || $2 == "*") { print "User: " $1 }' /etc/shadow 2>/dev/null \
    | tee -a "$report_file" | while read -r line; do logger -t "$log_tag" "$line"; done

############################################
# 4. Top Memory-Consuming Processes
############################################
print_section "Top 5 Memory-Consuming Processes"
ps aux --sort=-%mem | head -n 6 | tee -a "$report_file" | while read -r line; do logger -t "$log_tag" "$line"; done

############################################
# 5. Essential Services Status
############################################
print_section "Essential Services Status"
for service in systemd auditd cron systemd-journald ufw; do
    if systemctl is-active --quiet "$service"; then
        log_msg "$service is running."
    else
        log_msg "WARNING: $service is NOT running!"
    fi
done

############################################
# 6. Failed Login Attempts
############################################
print_section "Recent Failed Login Attempts"
logfile=""
if [ -f /var/log/auth.log ]; then
    logfile="/var/log/auth.log"
elif [ -f /var/log/secure ]; then
    logfile="/var/log/secure"
fi

if [ -n "$logfile" ]; then
    grep -i "failed" "$logfile" | tail -n 20 \
        | tee -a "$report_file" | while read -r line; do logger -t "$log_tag" "$line"; done
else
    log_msg "Authentication log not found."
fi

############################################
# Completion Message
############################################
log_msg "✅ Report completed and saved to: $report_file"
logger -t "$log_tag" "✅ System report completed."

echo -e "\n✅ Report available at: $report_file"
