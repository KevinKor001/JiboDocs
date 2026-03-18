- - -
# #AtDev , work in progress
- - -
Under /etc/init.d/ we have 

```shell

	# cd /etc/init.d/
# ls
S00fix-os                 S15crond                  S33dbus                   S48avahi-daemon           S63body-board-power       S78jibo-system-manager
S01logging                S18udev                   S36sshd                   S51upload-logs            S66ntp                    S81named
S06coredumps              S21firewall               S39audio-enable           S54modules                S69start-X11              S84identity-syslog
S09wifi-enable            S24cpufreq                S42avahi-setup.sh         S57alsa-volume            S72jibo-apply-update      rcK
S12dns-prime              S30urandom                S45network                S60alsaloopback           S75jibo-service-registry  rcS

```

currently interested in `/etc/init.d/S21firewall`

```log
# cat /etc/init.d/S21firewall
#!/bin/sh
#
# Jibo Firewall init script
#

set -e

IPTABLES_CMDS="/usr/sbin/iptables /usr/sbin/ip6tables"

flush_rules() {
    for iptables in $IPTABLES_CMDS; do
        $iptables -t filter -F
        $iptables -t filter -P INPUT ACCEPT
        $iptables -t filter -P FORWARD ACCEPT
        $iptables -t filter -P OUTPUT ACCEPT
        # add the DYNAMIC_ACCESS chain unconditionally
        $iptables -t filter -X
        $iptables -t filter -N DYNAMIC_ACCESS
    done
}

normal_rules() {
    for iptables in $IPTABLES_CMDS; do
        $iptables -t filter -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
        $iptables -t filter -A INPUT -p icmp -j ACCEPT
        $iptables -t filter -A INPUT -i lo -j ACCEPT
        # allow dynamic access rules from system-manager
        $iptables -t filter -A INPUT -j DYNAMIC_ACCESS
        $iptables -t filter -A INPUT -j REJECT
        $iptables -t filter -A FORWARD -j REJECT
    done
}

developer_rules() {
    for iptables in $IPTABLES_CMDS; do
        # jibo-dev-shell
        $iptables -t filter -A INPUT -p tcp --syn --dport 8686 -j ACCEPT
        # jibo-skills-service
        $iptables -t filter -A INPUT -p tcp --syn --dport 8779 -j ACCEPT
        # jibo-sync
        $iptables -t filter -A INPUT -p tcp --syn --dport 8989 -j ACCEPT
        # jibo-debug-proxy
        $iptables -t filter -A INPUT -p tcp --syn --dport 9191 -j ACCEPT
        # avahi
        $iptables -t filter -A INPUT -p udp --dport 5353 -j ACCEPT
    done
    normal_rules
}

certification_rules() {
    for iptables in $IPTABLES_CMDS; do
        # jibo-certification-service
        $iptables -t filter -A INPUT -p tcp --syn --dport 9292 -j ACCEPT
    done
    normal_rules
}

service_rules() {
    for iptables in $IPTABLES_CMDS; do
        # jibo-certification-service
        $iptables -t filter -A INPUT -p tcp --syn --dport 9292 -j ACCEPT
        # jibo-service-center-service
        $iptables -t filter -A INPUT -p tcp --syn --dport 9797 -j ACCEPT
        # avahi
        $iptables -t filter -A INPUT -p udp --dport 5353 -j ACCEPT
    done
    normal_rules
}

start() {
    echo -n "Configuring firewall: "
    flush_rules
    my_mode=$(/usr/bin/jibo-getmode)
    if [ $? -ne 0 ]; then
        echo "Unspecified mode. SKIP"
    elif [ "$my_mode" == "identified" ]; then
        echo "IDENTIFIED"
    elif [ "$my_mode" == "int-developer" ]; then
        echo "INT-DEVELOPER"
    elif [ "$my_mode" == "developer" ]; then
        developer_rules
        test $? -eq 0 && echo "DEVELOPER" || echo "ERROR"
    elif [ "$my_mode" == "certification" ]; then
        certification_rules
        test $? -eq 0 && echo "CERTIFICATION" || echo "ERROR"
    elif [ "$my_mode" == "service" ]; then
        service_rules
        test $? -eq 0 && echo "SERVICE" || echo "ERROR"
    else
        normal_rules
        test $? -eq 0 && echo "OK" || echo "ERROR"
    fi
}

stop() {
    echo -n "Unconfiguring firewall: "
    flush_rules
    test $? -eq 0 && echo "OK" || echo "ERROR"
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        stop
        start
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}" >&2
        exit 1
        ;;
esac
```

and in `S78jibo-system-manager`

```log
# cat S78jibo-system-manager 
#!/bin/sh
#
# Jibo System Manager init script
#

set -e

PROCESS=jibo-system-manager
BIN_DIR=/usr/local/bin
CFG_DIR=/usr/local/etc

check_mode() {
    my_mode=$(/usr/bin/jibo-getmode)
    if [ $? -ne 0 ]; then
        echo "Unspecified mode. SKIP"
        exit 0;
    fi
    if [ "$my_mode" != "oobe" \
        -a "$my_mode" != "int-developer" \
        -a "$my_mode" != "developer" \
        -a "$my_mode" != "normal" \
        -a "$my_mode" != "certification" \
        -a "$my_mode" != "service" ]; then
        echo "Mode is $my_mode. SKIP"
        exit 0;
    fi
    # only configure coredump generation for internal development modes
    # for all other modes, don't configure as they cannot be used
    if [ "$my_mode" == "int-developer" ]; then 
        echo "Configuring coredumps"
        # all subprocesses should generate core dumps
        ulimit -c unlimited
    fi
}

check_running() {
    pgrep -x jibo-system-man >& /dev/null
    return $?
}

wait_for_stopped() {
    while check_running; do
        echo -n "waiting... "
        sleep 2
    done
}

start() {
    echo -n "Starting $PROCESS: "
    check_mode
    $BIN_DIR/$PROCESS -c $CFG_DIR/$PROCESS.json --daemon
    test $? -eq 0 && echo "OK" || echo "ERROR"
}

stop() {
    echo -n "Stopping $PROCESS: "
    killall $PROCESS
    wait_for_stopped
    test $? -eq 0 && echo "OK" || echo "ERROR"
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    *)
        echo "Usage: $0 {start|stop}" >&2
        exit 1
        ;;
esac
```

to bypass the lockout in normal mode we can add like a filter under the normal rules function

first ima remount with write permissions :

```shell

	mount -o remount,rw /

# vi and append :

normal_rules() {
    for iptables in $IPTABLES_CMDS; do
        $iptables -t filter -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
        $iptables -t filter -A INPUT -p icmp -j ACCEPT
        $iptables -t filter -A INPUT -i lo -j ACCEPT
        # allow dynamic access rules from system-manager



       >>> $iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT <<<
        $iptables -t filter -A INPUT -j DYNAMIC_ACCESS
        $iptables -t filter -A INPUT -j REJECT
        $iptables -t filter -A FORWARD -j REJECT
    done
}


```

i was gonna use telnetd but its not installed

anyway using`jibo-getmode` i will revert back to the `normal` mode

and it works! saving diff for the installer

now that we have normal mode with ssh we have more capabilities, i will re screw the head back!... i broke my face ring
