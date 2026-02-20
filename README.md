# Introduction

This is a simple Bash script for [IPFire](https://ipfire.org) based firewalls that
implements a split tunnel VPN connection for Wireguard based VPN providers like
Mullvad or NordVPN.

It sets up a new Wireguard connection and allows the user to configure which clients
will be routed over the VPN tunnel through the use of Policy Based Routing. This can be
everything, only certain select devices, or the whole Blue/Green/Orange networks with
some exceptions if desired. Firewall rules are also handled automatically via iptables
using IPFire's custom chains.

While many features were added by the author, the idea of the script is based on the
approach taken here in the **nordvpn.sh** script:
[https://github.com/gvelim/ipfire_wg](https://github.com/gvelim/ipfire_wg)

Many thanks to the user **gvelim** for their work on the original script.

I encourage reading the README in its entirity below, and also the above linked
repository's README.

# Contents

- [Background](#background)   
- [Installation](#installation)   
  * [Configuration File](#configuration-file)   
  * [Notes on DNS](#notes-on-dns)   
- [Test It Out](#test-it-out)   
  * [Enabling At Startup](#enabling-at-startup)   

# Background

As an example, suppose one has a subscription with a VPN provider that allows a
standard basic Wireguard based connection (Examples would be Mullvad, NordVPN
or ProtonVPN), and wishes to move that VPN connection to their Firewall.

Moving the VPN connection to the Firewall from individual devices has many
advantages such as only using one device from the VPN providers device list, and
it also goes a long way to take back control from privacy hostile devices like
Android/iOS devices or Windows 11 systems. There are also many other benefits not
considered here.

Wireguard Support in IPFire is relatively new having been added recently. By default,
IPFire supports the following two configuration options for Wireguard connections:

* Roadwarrior
* Net-to-Net

**Roadwarrior** is to allow single or multiple clients local access from the internet
(Red Zone) to the Green/Blue/Orange Zones as if they were physically present.

**Net-to-Net** is intended to connect two LANs together over the internet via an
encrypted Wireguard tunnel seemlessly.

Official Documentation from the IPFire team is here:
[https://www.ipfire.org/docs/configuration/services/wireguard](https://www.ipfire.org/docs/configuration/services/wireguard)

These two cases are well supported on IPFire and work very well. However, a connection
to a VPN provider is not intended to be used like this. In this scenario, one would desire
all standard local services a Firewall provides (DNS, DHCP, Port-Forwarding etc.) to 
work as intended while traffic from the local network exits over the VPN tunnel.

By default IPFire does not support connecting via Wireguard like this. If one attempts
to import a Wireguard Configuration file provided by a VPN provider, IPFire's scripts
modify the main routing table and break essential services like DHCP etc.

Moreover, the Wireguard Configuration files provided by VPN providers are typically intended
to be used with the **wg-quick** popular Bash script which IPFire does not use, likely
due to its heavily customized firewall implementation via **iptables**.

That is where this script comes in. It does the following:

* Takes the Wireguard Configuration File provided by the VPN provider, and brings up a new Wireguard interface with all of the details provided.

* Creates a custom routing table where the default connection is changed from the Red interface to the new Wireguard tunnel interface.

* Adds the appropriate Firewall rules to the CUSTOMFORWARD & CUSTOMPOSTROUTING chains as is fully supported by IPFire to allow traffic to exit via the new Wireguard Tunnel.

* Adds Policy Based Routing rule(s) to route the desired clients/devices over the VPN tunnel instead of the standard Red interface.

The Firewall's internet connection itself is unaffected which allows all local services like
or DHCP, Local Roadwarrior VPN access, Port-Forwarding etc. to continue to function as normal.

# Installation

**First - Note that this script has only been tested on IPFire based systems, anything else**
**has not been tested by the author.**

Firstly SSH into your IPFire firewall. This script is entirely manual over SSH for now.
***(The author would like to explore implementing something via the WebUI, but that is a future goal)***

Then, do the following to install the script.

```bash
# Download the Script and Sample Configuration File
curl --output wireguard-vpn-tunnel https://raw.githubusercontent.com/zoot101/ipfire-wireguard-split-tunnel/refs/heads/main/wireguard-vpn-tunnel
curl --output wireguard-vpn-tunnel.conf https://raw.githubusercontent.com/zoot101/ipfire-wireguard-split-tunnel/refs/heads/main/wireguard-vpn-tunnel.conf

# Make the main script executable
chmod +x wireguard-vpn-tunnel

# Put the Sample Configuration File in the expected location
mkdir /etc/wireguard-vpn-tunnel/
mv wireguard-vpn-tunnel.conf /etc/wireguard-vpn-tunnel/
```

The next step is to set up the configuration file.

## Configuration File

The configuration file **/etc/wireguard-vpn-tunnel/wireguard-vpn-tunnel.conf** has the following
options. Edit it and configure the following options accordingly.

### wg\_vpn\_interface

This is the name of the new Wireguard interface the script will create using the VPN providers
configuration file. This can be anything but the author does not recommend changing this from 
**wgtunnel0** as that has worked well for the author to date. **wg0** should be avoided as
IPFire's default Wireguard implementation (Roadwarrior/Net-to-Net) uses this.

### roadwarrior\_iface

If one is using a Wireguard or OpenVPN connection in a roadwarrior configuration, specify the name
of the interface here. This is to add the necessary routing rules so that devices connected
to the VPN tunnel can still reply to the Roadwarrior interface as expected such that remote access
works as expected.

An example would be talking to a local NVR system while connected to the Roadwarrior interface even
though the NVRs internet connection is routed out over the VPN tunnel.

The default Wireguard interface in IPFire is usually called **wg0**, while the OpenVPN based interface is
typically **tun0**. This should also work if one is using the Net-to-Net configuration. Currently only
a single interface/adapter is supported.

Leave commented out if not using.

### roadwarrior\_network

Network Subnet for the roadwarrior network. Leave commented out if not using a roadwarrior
setup (Wireguard or OpenVPN). Example: **192.168.9.0/24**

### Green, Blue and Orange Interfaces & Network Subnets

Interface names and network subnets for the Green, Blue and Orange (Local) zones. Example:
```txt
green_iface="green0"
green_network="192.168.0.0/24"
blue_iface="blue0"
blue_network="192.168.1.0/24"
orange_iface="orange0"
orange_network="192.168.2.0/24"
```
Every IPFire setup has a Green Zone, the Blue and Orange are optional. If not using those
comment out the corresponding iface & network paramter above.

### wg\_config\_file

This is the path to the configuration file provided by the VPN provider. Given IPFire does not support
ipv6 (yet), its best to select the options for ipv4 only if the VPN provider gives that
option. If both ipv6 & ipv4 are given, the script will only use the ipv4 local address.

No changes to the file downloaded from the VPN provider should be needed. The only information
that is used by the script to bring up the new Wireguard Interface are the Endpoint, its Public Key,
the Private Key of the new Wireguard interface and its local IP address. If any firewall and routing
rules are defined, they are ignored. The script sets up the appropriate firewall rules automatically instead.

A typical example config file would would look like this:

```
[Interface]
PrivateKey = 1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZ/=12345=
Address = 192.168.93.47/32
DNS = 192.168.98.109

[Peer]
PublicKey = 1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZ/=12345=
AllowedIPs = 0.0.0.0/0
Endpoint = 1.2.3.4:51820
```

### tunnel\_ips

An array of the LAN IP addresses that should be routed over the new Wireguard Tunnel
interface. Note that this should be entered as an array per bash syntax. This can be
either single IPs, Example: **192.168.0.9** or the entire Green/Blue/Orange subnets. Example:
**192.168.0.0/24**

Examples are shown below. Must be an array in bash - note the brackets below.
```
# Example: Full Green, Blue and Orange
tunnel_ips=( "192.168.0.0/24" "192.168.1.0/24" "192.168.2.0/24")

# Example: Green only
tunnel_ips=( "192.168.0.0/24" )

# Example: Many single IPs
tunnel_ips=( "192.168.0.10" \
             "192.168.0.11" \
             "192.168.0.12" )
			 
# Example: Single Device only
tunnel_ips=( "192.168.1.7" )
```

### excluded\_ips

Lets say one wishes to route all traffic from all of their local Zones (Green, Blue or Orange)
over the VPN tunnel, but there are a number of devices they want to be excluded from being
routed over the tunnel, that is where this option comes in.

Some examples (not limited to) would be:

* A work computer provided by ones employer where connecting from a VPN IP would be suspicious
* A system where one wants to do online banking where the VPN IP could also raise suspicions (depends on the bank)
* A server that needs to be directly reachable from the internet via Port-Forwarding

The last example is particularly important. If one wishes a service to be directly reachable
from the internet via NAT/Port-Forward like a self-hosted Plex/Jellyfin server, the IP that this
service listens on locally **must** be excluded from being routed over the tunnel as it will
break remote access since the direct route back to the external client is re-directed over the tunnel
which is not desired.

Add as many as desired. Comment out if not needed. See example below - should be an array as per bash.

```
# Example
excluded_ips=( "192.168.0.17" \
               "192.168.0.18  \
               "192.168.0.19" )
```

### block\_tunnel\_dns

This option is intended for those who have local DNS servers on devices other than their
IPFire firewall on their network doing local DNS filtering of Ads, Trackers etc. Examples
would be Pi-hole, Bind9, Unbound etc.

A common way to get around this type of DNS filtering is to simply try and talk to a different
DNS servers directly. Privacy-hostile devices like Smart TVs can often hardcode Google's or
other DNS servers to do this.

The best defence against this is to block all external DNS requests to the Red Zone except from
your local DNS Server(s). This stops the aforementioned devices in their tracks and leaves them
no option but to talk to your local DNS server instead.

IPFire makes this very easy to do via the WebUI, but the script does not do this by default and
will allow all DNS queries to exit via the tunnel to anywhere. Set to "yes" to enable this,
whereby all but your desired DNS servers are blocked from sending DNS requests over the tunnel.
Note that this requires a list of the IP addresses of any local DNS servers - see below.

Leave commented out if one does not need this capability.

### local\_dns\_servers

IP addresses of any Local DNS servers permitted to send DNS request to the Red Zone (Internet)
as mentioned above.

Example, again note the brackets. This should be an array as per bash.
```
local_dns_servers=( "192.168.0.5" \
                    "192.168.0.6" )
```

### route\_firewall\_dns\_into\_tunnel

This option is intended for those who are using IPFire's Unbound DNS server
for all of the clients on their network and desire to have the DNS requests
sent into the tunnel also.

This adds the necessary firewall rules to allow the Firewall itself to send
DNS traffic over the tunnel, and adds an additional Policy Based Routing
rule to set this up. This avoids having to change anything in IPFire's
Unbound configuration.

Note though if one is using IPFire's Unbound server in Recursive mode, one
may have problems with the Root DNS Servers refusing queries from certain
VPN IPs. It might be better to just configure Unbound through the WebUI
to use a forwarding server instead. There is a great list here:
[https://www.ipfire.org/docs/dns/public-servers](https://www.ipfire.org/docs/dns/public-servers)

Should be "yes" or "no". Comment out or set to no if not using.

## Notes on DNS

DNS Leaks when one is using a VPN should be avoided. It kind of defeats the purpose
of using a VPN if your DNS traffic is not traversing over the VPN connection also.
Once the requests do get over the VPN, it does not matter where they go after that
given your IP should be shared with many other users.

If one is using this script there are a number of options:

* Hand out the DNS Sever from the VPN Providers configuration via your DHCP Server. While this
is fine, it's a bit cumbersome as when one changes the VPN Providers config file, the
DHCP Server config has to be changed again.

* Use a Local DNS Server. This is ideal if you're already using something like Pi-hole
on your network. Simply use the **local_dns_servers** and **block_tunnel_dns** options
discussed above to exclude the Pi-hole servers IP to allow it is allowed to send DNS
requests over the tunnel while everything else is blocked from doing so. The DHCP server
config does not need to be changed whether the VPN is active or not.

* Use IPFire's Unbound installation. This is probably the most convienent. Use the
**route_firewall_dns_into_tunnel** option discussed above along with **block_tunnel_dns**
option so that only the Firewall can send out DNS requests over the tunnel.

* Finally note that in the Author's experience if one is running their local DNS server,
be it IPFire's Unbound implementation or something else like Bind9 in resursive mode,
they may run into issues with the Root Servers refusing DNS queries from known VPN
IPs. This will vary by your VPN Provider, but if this does happen, it's best to
configure a forwarding DNS Server instead.

# Test It Out

With the configuration file setup, the next step is to test the script directly. All
supported input parameters to the script are shown below:

## Usage:

```text
Usage: /etc/init.d/wireguard-vpn-tunnel [ OPTIONS ] { ACTION } 

 ACTION (Required) can be one of the following:

  start      : Start the VPN Tunnel - Does the Following:
               * Bring up a new Wireguard Interface using the VPN Providers Config File
               * Create a custom routing table
               * Add the necessary firewall rules to allow routing over the Tunnel
               * Setup firewall rules to block all DNS over the tunnel with the exception
                 of dedicated DNS Servers (if enabled in config file)
               * Activate Policy Based Routing Rules to activate the VPN Tunnel
               * Add additional Policy Based Routing Rules to exclude certain IPs from
                 being routed over the tunnel (if enabled in config file)

  stop       : Stop the VPN Tunnel - Does the Following:
               * Remove the Policy Based Routing rules
               * Remove the main firewall rules created above
               * Remove the Policy Based Routing rules to exclude certain IPs
                 (if enabled in config file)
               * Remove the Policy Based Routing rules to exclude certain IPs from
                 being routed over the tunnel (if enabled in config file)
               * Remove the firewall rules to block all DNS over the tunnel with the exception
                 of dedicated DNS Servers (if enabled in config file)
               * Remove the Wireguard Interface

  status     : Display the Status of the Following (if --debug is used - see below:)
               * Custom Routing Table
               * Policy Based Routing Rules
               * Custom Firewall Rules
               * Wireguard Interface Status

  reload     : Stop & Restart the VPN Tunnel

  restart    : Stop & Restart the VPN Tunnel

 OPTIONS (Optional) can be the following:

  -c|--config PATH-TO-CONFIG-FILE : Override default config file path (below)

  -d|--debug : Print out detailed information about configuration and each of theabove steps
               as by default the script has a quiet output

  -h|--help  : Print this message

 CONFIG-FILE : Default: /etc/wireguard-vpn-tunnel/wireguard-vpn-tunnel.conf
```

Start the tunnel with: `wireguard-vpn-tunnel --debug start`

Check the status with: `wireguard-vpn-tunnel --debug status`

Then, test out the internet IP websites see with a service like [https://ifconfig.co](https://ifconfig.co)
on a device that one intends to route over the tunnel. You should see the VPN IP,
assuming you are using a device that was configured to be routed over the VPN
tunnel.

Stop the tunnel: `wireguard-vpn-tunnel --debug stop`

The external IP should now go back to your regular IP. If one gets to here,
the script is working as intended. Note the use of the \--debug option, this
is to print out detailed information to allow one to see what is going on.

Without this option, the output is quiet and does not print out anything
such that it behaves correctly if it is called by the system on startup.

Note that the script does quite a bit of error checking on the IPs, interfaces and everything
else that is specified in the config file so the risk of something going wrong
with an accidental error there should be low.

The next step is to configure the script to start upon poweron, however one should be
sure that calling the script appropriately in **/etc/init.d/** and produces the expected
result (see below).

## Enabling at Startup

IPFire does not use Systemd, but rather uses the older SysV init system. Once one
has confirmed the script is working as above proceed below.

Do the following to enable it on startup:
```Bash
# Place a copy in /etc/init.d/
cp wireguard-vpn-tunnel /etc/init.d/

# Update the permissions
chown root:root /etc/init.d/wireguard-vpn-tunnel
chmod 754 /etc/init.d/wireguard-vpn-tunnel
```

It's now a good idea to re-test the script works correctly again by calling
it exactly as it will be called by the system upon startup.
```bash
/etc/init.d/wireguard-vpn-tunnel start # Start the tunnel
/etc/init.d/wireguard-vpn-tunnel stop  # Stop the tunnel
```

If the above produces the desired result, move on below.

Next, to enable the tunnel to be started upon powerup and stopped upon shutdown
and reboot do the following:
```
# Start the Tunnel on Startup of the Firewall
ln -s /etc/init.d/wireguard-vpn-tunnel /etc/rc.d/rc3.d/S60wireguard-vpn-tunnel

# Stop the Tunnel on Reboot of the Firewall
ln -s /etc/init.d/wireguard-vpn-tunnel /etc/rc.d/rc6.d/K65wireguard-vpn-tunnel

# Stop the TUnnel on Poweroff of the Firewall
ln -s /etc/init.d/wireguard-vpn-tunnel /etc/rc.d/rc0.d/K65wireguard-vpn-tunnel
```

Now reboot the IPFire based firewall to test it out. The VPN connection, and
associated routing should all now be activated by default.

Hopefully this script is useful to someone!


