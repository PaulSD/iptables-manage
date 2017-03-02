# iptables Management Scripts and Baseline Configs

The standard iptables-restore program is rather limited in that it requires all rules to be listed in a single file, and it does not support mixing IPv4 and IPv6 rules.  This becomes particularly problematic in large-scale deployments, where a standard set of baseline rules for all systems (that typically apply to both IPv4 and IPv6 traffic), a standard set of baseline rules per system class/type, and a local set of system-specific rules need to be merged and loaded in a simple, clean, and maintainable manner.  Several tools (such as ufw) are available which attempt to solve this problem by hiding iptables behind an abstraction layer.  However, if any non-trivial iptables rules are needed, the abstraction layer typically gets in the way more than it helps.

To facilitate better iptables rule management without adding a new abstraction layer, /usr/local/sbin/ip46tables-manage supports mixed IPv4/IPv6 rules and will load iptables rules from multiple files under /etc/iptables.d/.  To minimize any need to coordinate the creation/ordering/contents of these rules files to ensure proper rule ordering, a set of rules file conventions is laid out in /etc/iptables.d/ReadMe, and some basic framework files and example baseline files are provided.  An ip46tables script is also provided to facilitate configuring mixed IPv4/IPv6 rules from the command line.  If ipset is being used, /usr/local/sbin/ipset-manage supports loading and saving ipsets in files under /etc/ipset.d/.

## Installation

```bash
cp usr/local/sbin/* /usr/local/sbin/
ln -s /usr/local/sbin/ip46tables-manage /usr/local/sbin/iptables-manage
chown root:root /usr/local/sbin/ip*tables* /usr/local/sbin/ipset-manage
chmod 755 /usr/local/sbin/ip*tables* /usr/local/sbin/ipset-manage

mkdir -p /etc/iptables.d/ /etc/ipset.d/
cp etc/iptables.d/* /etc/iptables.d/
chown -R root:root /etc/iptables.d/ /etc/ipset.d/
chmod -R o-rwx /etc/iptables.d/ /etc/ipset.d/

sudo apt-get install netfilter-persistent
ln -s /usr/local/sbin/ip46tables-manage /usr/share/netfilter-persistent/plugins.d/20-ip46tables-manage
ln -s /usr/local/sbin/ipset-manage /usr/share/netfilter-persistent/plugins.d/15-ipset-manage

cp etc/rsyslog.d/* /etc/rsyslog.d/
cp etc/logrotate.d/* /etc/logrotate.d/
# See: https://bugs.launchpad.net/ubuntu/+source/rsyslog/+bug/1531622
# and: http://stackoverflow.com/a/25289586/1476175
perl -i -p \
 -e 'BEGIN { $params =
  "permitnonkernelfacility=\"on\" " .
  "parsekerneltimestamp=\"on\" " .
  "keepkerneltimestamp=\"off\"";
 } ' \
 -e 's/(module\(load="imklog")(?:[^)]*)\)/$1 $params)/;' \
 -e 's/^(?!#)(.*\$KLogPermitNonKernelFacility.*)$/#$1/;' \
 /etc/rsyslog.conf
```

## Basic Usage

```bash
# Load rules from /etc/iptables.d/
ip46tables-manage load

# Flush rules (disable all rules)
ip46tables-manage flush-all

# Show any manual changes that have been made to the running rule set since the
# last call to `ip46tables-manage load`
ip46tables-manage diff
```

## Additional Example Rule Files

### Allow NTP
```bash
cat <<END > /etc/iptables.d/NTP.rules
ip46tables --append INormal --protocol UDP --destination-port 123 --jump ACCEPT
END
```

### Allow SSH, but limit connections to 3/minute short term or 2/minute long term from any single host
```bash
cat <<END > /etc/iptables.d/SSH.rules
ip46tables --new-chain ratelimit-SSH
ip46tables --append ratelimit-SSH --match hashlimit --hashlimit-name SSH --hashlimit-mode srcip --hashlimit-burst 3 --hashlimit-upto 2/m --jump ACCEPT
ip46tables --append ratelimit-SSH --match hashlimit --hashlimit-name ratelimit-SSH --hashlimit-mode srcip --hashlimit-burst 1 --hashlimit-upto 6/m --jump LOG --log-prefix 'iptables rl-SSH: '
ip46tables --append ratelimit-SSH --protocol TCP --jump REJECT --reject-with tcp-reset
ip46tables --append INormal --protocol TCP --destination-port 22 --syn --jump ratelimit-SSH
END
```

### Limit all inbound connections from any single host
```bash
cat <<END > /etc/iptables.d/Limit_All.rules
# Limit inbound connections from any single host
# (Separate limits are applied to INPUT and FORWARD packets)
ip46tables --new-chain connlimit-all
ip46tables --append IValidate --protocol TCP --syn --match connlimit --connlimit-above 100 --jump connlimit-all
ip46tables --append FValidate --protocol TCP --syn --match connlimit --connlimit-above 100 --jump connlimit-all

ip46tables --append connlimit-all --match hashlimit --hashlimit-name connlimit --hashlimit-mode srcip --hashlimit-burst 1 --hashlimit-upto 6/m --jump LOG --log-prefix 'iptables cl: '
ip46tables --append connlimit-all --protocol TCP --jump REJECT --reject-with tcp-reset
ip46tables --append connlimit-all --jump REJECT
END
```

### Allow Ping
```bash
cat <<END > /etc/iptables.d/Ping.rules
# This should generally only match ping (most other ICMP packets should match
# the '--match conntrack --ctstate ESTABLISHED,RELATED' rule instead), however
# it is possible for other ICMP packets to match this in some cases.  To
# strictly match only ping, use '--icmp-type echo-request' and
# '--icmpv6-type echo-request'.
ip4tables --insert INormal --protocol ICMP --jump ACCEPT
ip6tables --insert INormal --protocol ICMPv6 --jump ACCEPT
ip4tables --insert FNormal --protocol ICMP --jump ACCEPT
ip6tables --insert FNormal --protocol ICMPv6 --jump ACCEPT
END
```

# Allow UPnP
```bash
cat <<END > /etc/iptables.d/UPnP.rules
# Additional ports may be required depending on the remote device's implementation
# For example, ChromeCast requires 32768-61000
ip46tables --append INormal --protocol TCP --destination-port 2869 --jump ACCEPT
ip46tables --append INormal --protocol UDP --destination-port 1900 --jump ACCEPT
END
```

## License

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see [http://www.gnu.org/licenses/](http://www.gnu.org/licenses/).
