# Conduro Ubuntu 20.04
Linux is well-known for being one of the most secure operating systems available. But that doesn't mean you can count on it to be as secure as possible right out of the box. Conduro (_Hardening in Latin_) will automate this process to ensure your platform is secure.

> ⚠ We recommend to not execute this script on servers with existing firewall configurations.

# Getting Started
This script is designed to be executed on a freshly installed **Ubuntu Server 20.04** server.

```bash
wget -O ./install.sh https://condu.ro/install.sh && chmod +x ./install.sh && sudo ./install.sh
```
![](https://i.imgur.com/PXK7Ctk.gif)

# What does it do?
The purpose of Conduro is to optimize and secure your system to run web applications. It does this by disabling unnecessary services, bootstrapping your firewall, secure your system settings and other things. Continue reading if you want to know exactly what's being executed.

#### update dependencies
```bash
apt-get install wget sed git -y
```

#### update system
Keeping the system updated is vital before starting anything on your system. This will prevent people to use known vulnerabilities to enter in your system.
```bash
apt-get update -y && apt-get full-upgrade -y
```

#### update golang **optional**
```bash
rm -rf /usr/local/go
wget -q -c https://dl.google.com/go/$(curl -s https://golang.org/VERSION?m=text).linux-amd64.tar.gz -O go.tar.gz
tar -C /usr/local -xzf go.tar.gz
echo "export GOROOT=/usr/local/go" >> /etc/profile
echo "export PATH=/usr/local/go/bin:$PATH" >> /etc/profile
source /etc/profile
rm go.tar.gz
```

#### update nameservers
We change the default nameservers to cloudflare because https://www.dnsperf.com/#!dns-resolvers
```bash
truncate -s0 /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee -a /etc/resolv.conf
echo "nameserver 1.0.0.1" | sudo tee -a /etc/resolv.conf
```
#### update ntp servers
```bash
truncate -s0 /etc/systemd/timesyncd.conf
echo "[Time]" | sudo tee -a /etc/systemd/timesyncd.conf
echo "NTP=time.cloudflare.com" | sudo tee -a /etc/systemd/timesyncd.conf
echo "FallbackNTP=ntp.ubuntu.com" | sudo tee -a /etc/systemd/timesyncd.conf
```

#### update sysctl.conf
```bash
wget -q -c https://raw.githubusercontent.com/conduro/ubuntu/main/sysctl.conf -O /etc/sysctl.conf
```
```conf
# IP Spoofing protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Ignore ICMP broadcast requests
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Disable source packet routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0 
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Ignore send redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Block SYN attacks
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# Log Martians
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0 
net.ipv6.conf.default.accept_redirects = 0

# Ignore Directed pings
net.ipv4.icmp_echo_ignore_all = 1

# Disable IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# Hide kernel pointers
kernel.kptr_restrict = 2

# Enable panic on OOM
vm.panic_on_oom = 1

# Reboot kernel ten seconds after OOM
kernel.panic = 10
```

#### update sshd_config
```bash
wget -q -c https://raw.githubusercontent.com/conduro/ubuntu/main/sshd.conf -O /etc/ssh/sshd_config
```
```conf
# To disable tunneled clear text passwords, change to no here!
PasswordAuthentication yes

# Depending on your 2FA option, you may need to enable some of these options, but they should be disabled by default
ChallengeResponseAuthentication no
PasswordAuthentication no

# Allow client to pass locale environment variables
AcceptEnv LANG LC_*

# Disable connection multiplexing which can be used to bypass authentication
MaxSessions 1

# Block client 10 minutes after 3 failed login attempts
MaxAuthTries 3
LoginGraceTime 10

# Do not allow empty passwords
PermitEmptyPasswords no

# Enable PAM authentication
UsePAM yes

# Disable Kerberos based authentication
KerberosAuthentication no
KerberosGetAFSToken no
KerberosOrLocalPasswd no
KerberosTicketCleanup yes
GSSAPIAuthentication no
GSSAPICleanupCredentials yes

# Disable user environment forwarding
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitUserRC no
PermitUserEnvironment no

# We want to log all activity
LogLevel INFO
SyslogFacility AUTHPRIV

# What messages do you want to present your users when they log in?
Banner none
PrintMotd no
PrintLastLog yes

# override default of no subsystems
Subsystem sftp  /usr/lib/openssh/sftp-server
```


#### disable system logging
```bash
systemctl stop systemd-journald.service
systemctl disable systemd-journald.service
systemctl mask systemd-journald.service

systemctl stop rsyslog.service
systemctl disable rsyslog.service
systemctl mask rsyslog.service
```


#### configure firewall
```bash
ufw disable
echo "y" | sudo ufw reset
ufw logging off
ufw default deny incoming
ufw default allow outgoing
ufw allow 80/tcp
ufw allow 443/tcp

# optional prompt to change ssh port
    ufw allow ${prompt}/tcp
    sed -i "/Port /Id" /etc/ssh/sshd_config
    echo "Port ${prompt}" | sudo tee -a /etc/ssh/sshd_config
# defaults to port 22
    ufw allow 22/tcp

sed -i "/ipv6=/Id" /etc/default/ufw
echo "IPV6=no" | sudo tee -a /etc/default/ufw

sed -i "/GRUB_CMDLINE_LINUX_DEFAULT=/Id" /etc/default/grub
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"ipv6.disable=1 quiet splash\"" | sudo tee -a /etc/default/grub
```

#### free disk space
```bash
find /var/log -type f -delete
rm -rf /usr/share/man/*
apt-get autoremove -y
apt-get autoclean -y
```

#### reload system
```bash
sysctl -p
update-grub2
systemctl restart systemd-timesyncd
ufw --force enable
service ssh restart
```

## QUESTIONS:

1.What is the command to set the grub password?
solution:

    To set a GRUB password (for GRUB2), follow these steps:

    1. Generate a GRUB password hash:
    Use the grub-mkpasswd-pbkdf2 command to generate a hashed password:


    grub-mkpasswd-pbkdf2
    Enter your desired password when prompted.

    You’ll get output like this:


    PBKDF2 hash of your password is grub.pbkdf2.sha512.10000.XXXXXXXXXXXXXXXXX
    2. Edit the GRUB configuration file:
    Open /etc/grub.d/40_custom with a text editor:


    sudo nano /etc/grub.d/40_custom
    Add the following lines:

    set superuser="admin"
    password_pbkdf2 admin grub.pbkdf2.sha512.10000.XXXXXXXXXXXXXXXXX
    Replace "admin" with your desired username and paste the hashed password.

    3. Update GRUB:
    After editing the config, update GRUB so the changes take effect:


    sudo update-grub
    4. (Optional) Restrict menu entries:
    To restrict specific menu entries (like recovery mode), edit /etc/grub.d/10_linux or other appropriate scripts and add:

    menuentry 'Your Entry' --users admin {
        ...
    }

2.How to run the script ubuntu.sh?
Solution:

```
#!/bin/bash

# shellcheck disable=1090
# shellcheck disable=2009
# shellcheck disable=2034

set -u -o pipefail

if ! ps -p $$ | grep -si bash; then
  echo "Sorry, this script requires bash."
  exit 1
fi

if ! [ -x "$(command -v systemctl)" ]; then
  echo "systemctl required. Exiting."
  exit 1
fi

function main {
  clear

  REQUIREDPROGS='arp dig ping w'
  REQFAILED=0
  for p in $REQUIREDPROGS; do
    if ! command -v "$p" >/dev/null 2>&1; then
      echo "$p is required."
      REQFAILED=1
    fi
  done

  if [ $REQFAILED = 1 ]; then
    apt-get -qq update
    apt-get -qq install bind9-dnsutils iputils-ping net-tools procps --no-install-recommends
  fi

  ARPBIN="$(command -v arp)"
  DIGBIN="$(command -v dig)"
  PINGBIN="$(command -v ping)"
  WBIN="$(command -v w)"
  WHOBIN="$(command -v who)"
  LXC="0"

  if resolvectl status >/dev/null 2>&1; then
    SERVERIP="$(ip route get "$(resolvectl status |\
      grep -E 'DNS (Server:|Servers:)' | tail -n1 |\
      awk '{print $NF}')" | grep -Eo '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' |\
      tail -n1)"
  else
    SERVERIP="$(ip route get "$(grep '^nameserver' /etc/resolv.conf |\
      tail -n1 | awk '{print $NF}')" |\
      grep -Eo '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | tail -n1)"
  fi

  if grep -qE 'container=lxc|container=lxd' /proc/1/environ; then
    LXC="1"
  fi

  if grep -s "AUTOFILL='Y'" ./ubuntu.cfg; then
    USERIP="$($WHOBIN | awk '{print $NF}' | tr -d '()' |\
      grep -E '^[0-9]' | head -n1)"

    if [[ "$USERIP" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
      ADMINIP="$USERIP"
    else
      ADMINIP="$(hostname -I | sed -E 's/\.[0-9]+ /.0\/24 /g')"
    fi

    sed -i "s/FW_ADMIN='/FW_ADMIN='$ADMINIP /" ./ubuntu.cfg
    sed -i "s/SSH_GRPS='/SSH_GRPS='$(id "$($WBIN -ih | awk '{print $1}' | head -n1)" -ng) /" ./ubuntu.cfg
    sed -i "s/CHANGEME=''/CHANGEME='$(date +%s)'/" ./ubuntu.cfg
    sed -i "s/VERBOSE='N'/VERBOSE='Y'/" ./ubuntu.cfg
  fi

  source ./ubuntu.cfg

  readonly ADDUSER
  readonly ADMINEMAIL
  readonly ARPBIN
  readonly AUDITDCONF
  readonly AUDITD_MODE
  readonly AUDITD_RULES
  readonly AUDITRULES
  readonly AUTOFILL
  readonly CHANGEME
  readonly COMMONACCOUNT
  readonly COMMONAUTH
  readonly COMMONPASSWD
  readonly COREDUMPCONF
  readonly DEFAULTGRUB
  readonly DISABLEFS
  readonly DISABLEMOD
  readonly DISABLENET
  readonly FAILLOCKCONF
  readonly FW_ADMIN
  readonly JOURNALDCONF
  readonly KEEP_SNAPD
  readonly LIMITSCONF
  readonly LOGINDCONF
  readonly LOGINDEFS
  readonly LOGROTATE
  readonly LOGROTATE_CONF
  readonly LXC
  readonly NTPSERVERPOOL
  readonly PAMLOGIN
  readonly PSADCONF
  readonly PSADDL
  readonly RESOLVEDCONF
  readonly RKHUNTERCONF
  readonly RSYSLOGCONF
  readonly SECURITYACCESS
  readonly SERVERIP
  readonly SSHDFILE
  readonly SSHFILE
  readonly SSH_GRPS
  readonly SSH_PORT
  readonly SYSCTL
  readonly SYSCTL_CONF
  readonly SYSTEMCONF
  readonly TIMEDATECTL
  readonly TIMESYNCD
  readonly UFWDEFAULT
  readonly USERADD
  readonly USERCONF
  readonly VERBOSE
  readonly WBIN

  for s in ./scripts/*; do
    [[ -f $s ]] || break

    source "$s"
  done

  f_pre
  f_kernel
  f_firewall
  f_disablenet
  f_disablefs
  f_disablemod
  f_systemdconf
  f_resolvedconf
  f_logindconf
  f_journalctl
  f_timesyncd
  f_fstab
  f_prelink
  f_aptget_configure
  f_aptget
  f_hosts
  f_issue
  f_sudo
  f_logindefs
  f_sysctl
  f_limitsconf
  f_adduser
  f_rootaccess
  f_package_install
  f_psad
  f_coredump
  f_usbguard
  f_postfix
  f_apport
  f_motdnews
  f_rkhunter
  f_sshconfig
  f_sshdconfig
  f_password
  f_cron
  f_ctrlaltdel
  f_auditd
  f_aide
  f_rhosts
  f_users
  f_lockroot
  f_package_remove
  f_suid
  f_restrictcompilers
  f_umask
  f_path
  f_aa_enforce
  f_aide_post
  f_aide_timer
  f_aptget_noexec
  f_aptget_clean
  f_systemddelta
  f_post
  f_checkreboot

  echo
}
```
LOGFILE="hardening-$(hostname --short)-$(date +%y%m%d).log"
echo "[HARDENING LOG - $(hostname --fqdn) - $(LANG=C date)]" >> "$LOGFILE"

main "$@" | tee -a "$LOGFILE"


3..What is the command to download conduro?
Solution:
    Conduro Ubuntu 20.04
    Linux is well-known for being one of the most secure operating systems available. But that doesn't mean you can count on it to be as secure as possible right out of the box. Conduro (Hardening in Latin) will     automate this process to ensure your platform is secure.

    ⚠     We recommend to not execute this script on servers with existing firewall configurations.

    Getting Started
    This script is designed to be executed on a freshly installed Ubuntu Server 20.04 server.

    wget -O ./install.sh https://condu.ro/install.sh && chmod +x ./install.sh && sudo ./install.sh

4.How to change the permission of the script install.sh?
Solution:
    
    To change the permission of the script install.sh, you typically use the chmod command in the terminal. This command modifies file permissions on Unix-like operating systems (Linux, macOS, etc.).

    To make install.sh executable:

    chmod +x install.sh
    This adds execute permission to the script for the file's owner (and potentially group and others, depending on existing permissions).

    Explanation:
    chmod: change mode (permissions)

    +x: add execute permission

    install.sh: the file you're modifying

    Optional: View current permissions
    You can check current permissions with:


    ls -l install.sh
    It will output something like:


    -rw-r--r-- 1 user user 1234 May 21 10:00 install.sh
    After running chmod +x install.sh, it should change to:


    -rwxr-xr-x 1 user user 1234 May 21 10:00 install.sh
    Now you can run it using:


    ./install.sh


