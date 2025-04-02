# -Monitoring-System-Resources-for-a-Proxy-Server

# Proxy Server Monitoring Dashboard
# Author: Yash Karpe

refresh_rate=2 

generate_bar() {
    local usage=$1
    local max=10
    local filled=$((usage * max / 100))
    local empty=$((max - filled))
    printf "[%s%s] %d%%" "$(printf '#%.0s' $(seq 1 $filled))" "$(printf '-%.0s' $(seq 1 $empty))" "$usage"
}

display_cpu_memory() {
    cpu_usage=$(top -bn1 | grep 'Cpu(s)' | awk '{print $2 + $4}')
    load_avg=$(uptime | awk -F'load average:' '{ print $2 }')
    mem_total=$(free -m | awk '/Mem/ {print $2}')
    mem_used=$(free -m | awk '/Mem/ {print $3}')
    mem_usage=$((mem_used * 100 / mem_total))
    swap_total=$(free -m | awk '/Swap/ {print $2}')
    swap_used=$(free -m | awk '/Swap/ {print $3}')

}


display_disk_usage() {
    echo "| Disk:       |"
    df -h | awk '$5 ~ /[0-9]+%/ {print "| " $6 " " (substr($5,1,length($5)-1) > 80 ? "Warning: " : "") $5 " used |"}'
}

display_top_processes() {
    echo "+------------------------------------------------------------+"
    echo "| Top Processes (CPU & Mem)                                  |"
    echo "|------------------------------------------------------------|"
    echo "|  #  | Process Name      | CPU (%) | Memory (MB)           |"
    echo "|------------------------------------------------------------|"
    ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 6 | awk '{printf "| %2d  | %-15s | %6.1f   | %10.0f           |\n", NR, $2, $3, $4}'
    echo "+------------------------------------------------------------+"
}

display_network() {
    echo "+------------------------------------------------------------+"
    echo "| Network Monitoring                                         |"
    echo "|------------------------------------------------------------|"
    echo "| Active Connections: $(netstat -an | wc -l)  | Packet Drops: $(cat /proc/net/snmp | grep -w 'Icmp' | awk '{print $6}')               |"
    echo "| Data In: $(ifconfig | awk '/RX packets/ {print $5/1024/1024 "MB"}')  | Data Out: $(ifconfig | awk '/TX packets/ {print $5/1024/1024 "MB"}')  |"
    echo "+------------------------------------------------------------+"
}

display_services() {
    echo "+------------------------------------------------------------+"
    echo "| Services Status                                           |"
    echo "|------------------------------------------------------------|"
    for service in sshd nginx iptables; do
        status=$(systemctl is-active $service 2>/dev/null)
        printf "| %-8s: [%s] " "$service" "${status^^}"
    done
    echo "|"
    echo "+------------------------------------------------------------+"
}

display_dashboard() {
    clear
    echo "+------------------------------------------------------------+"
    echo "|                    SYSTEM MONITOR DASHBOARD               |"
    echo "+------------------------------------------------------------+"
    display_cpu_memory
    display_disk_usage
    display_top_processes
    display_network
    display_services
    echo "| Press [CTRL+C] to exit | Refreshing every ${refresh_rate}s...        |"
    echo "+------------------------------------------------------------+"
}

case "$1" in
    -cpu) display_cpu_memory ;;
    -disk) display_disk_usage ;;
    -processes) display_top_processes ;;
    -network) display_network ;;
    -services) display_services ;;
    *)
        while true; do
            display_dashboard
            sleep $refresh_rate
        done
        ;;
esac
