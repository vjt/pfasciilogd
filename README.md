# pfasciilogd

Daemon that spawns `tcpdump` on `pflog0` and writes its ascii
representation of captured packets to a log file.

## Motivation

I block all incoming traffic to non user-serviceable ports, and I employ
fail2ban to block incoming traffic from badly behaving peers tinkering with the
services I have open.

I wanted to prevent port scanners to detect the open ports on my system, so I
needed to feed fail2ban a text representation of incoming blocked packets, so
that originating IPs could be extracted and blocked.

BSDs pflogd produce a binary pcap dump in `/var/log/pflog` that is not suitable
for parsing by fail2ban, so I created this daemon that spawns tcpdump, opens
`pflog0` and writes the text represenation of anything that passes through it
in a text file, that fail2ban can read.

## What does this do

In a nutshell, this is a fancy asyncio python program for 

```
tcpdump -Z nobody -eni pflog0 | tee -a /var/log/pf.log
```

But this respawns `tcpdump` should it crash, and supports the HUP signal so you
can configure `newsyslog` to rotate its log file and have the daemon re-open
it.

## Installation

* Install python3.9 - but you should have it already if you have fail2ban
* Install `pfasciilogd` as `/usr/local/sbin/pfasciilogd`
* Install `pfasciilogd.rc` as `/usr/local/etc/rc.d/pfasciilogd`
* Set `pfasciilogd_enable="YES"` in `/etc/rc.conf`
* Start it via `/usr/local/etc/rc.d/pfasciilogd start`
* Create the following `newsyslog` configuration in `/usr/local/etc/newsyslog.conf.d/pfasciilogd.newsyslog.conf`:
```
# logfilename     [owner:group]    mode count size when  flags [/pid_file]              [sig_num]
/var/log/pf.log   root:wheel       640  5     1000 *     BJ    /var/run/pfasciilogd.pid 1
```
* Create the following fail2ban filter in `/usr/local/etc/fail2ban/filter.d/tcpdump.conf`:
```
# Fail2Ban filter for tcpdump-style logs
#
[Definition]
failregex = rule \d/\d\(match\): block in on \w+: <HOST>\.\d+
ignoreregex = 

# Author: Marcello Barnaba
```
* Create a jail configuration similar to the following, that'll block anybody
who sends 10 packets that end up being blocked by your `pf` config
```
[pf]
enabled  = true
filter   = tcpdump
action   = pf[name=pf, actiontype=<allports>]
logpath  = /var/log/pf.log
maxretry = 10
```
* Restart fail2ban and enjoy it blocking incoming scanners :)

## Caveats

### `pf` rules order

Place the `anchor f2b/*` rule before your `block log all` rule, because f2b
generates rules that block with `quick` so there's no need to do any further
processing on pkts from banned clients.

### Potential false positives

When connections on legitimate peers' get out of sync from pf's view of the
connection state, pf may block incoming PSH/FIN packets leading to false
positives and legit clients being blocked.

Feel free to tinker with the `failregex` to ignore TCP packets with `PF` flags
set, or you can increase the `maxretry` to increase the number of packets
required to result in a ban.
