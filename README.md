# linux-config

Files needed for configuration on Linux. 

**NOTE**: You will need to add an appropriate WebTV server IP into `/etc/rc.local` for all rules to work.

## The Dial-in Server

Much of this is courtesy Doge Microsystems, so credit where credit is due: https://dogemicrosystems.ca/wiki/Dial-up_pool

These instructions are specific to Debian-based Linux distributions, but should be similar for other distributions or Unixes.

1. Connect a USB to RS-232 adapter and confirm it shows up as `/dev/ttyUSBXXX` (Run `ls /dev/` or `dmesg` to check). In my case, it presents as `/dev/ttyUSB0`.
2. Install `ppp` (and `mgetty` if your distribution doesn't have it by default)  
 `$ sudo apt-get install ppp mgetty`

3. Create a `systemd` service for mgetty, by editing `/lib/systemd/system/mgetty@.service` (note the @) with your text editor of choice as root or sudo.

```
[Unit]
Description=External Modem %I
Documentation=man:mgetty(8)
Requires=systemd-udev-settle.service
After=systemd-udev-settle.service

[Service]
Type=simple
ExecStart=/sbin/mgetty /dev/%i
Restart=always
PIDFile=/var/run/mgetty.pid.%i

[Install]
WantedBy=multi-user.target
```

4. Configure `mgetty` by editing `/etc/mgetty/mgetty.config`  
 Comment out everything except the debug level by prepending lines with a pound/hash (`#`), and append the section for configuring the serial devices: I have 4 USB to serial devices (ttyUSB0, ttyUSB1, ttyUSB2, ttyUSB3):

```
debug 9

port ttyUSB0
 port-owner root
 port-group dialout
 port-mode 0660
 data-only yes
 ignore-carrier no
 toggle-dtr yes
 toggle-dtr-waittime 500
 rings 1
 speed 115200
 modem-check-time 160

port ttyUSB1
 port-owner root
 port-group dialout
 port-mode 0660
 data-only yes
 ignore-carrier no
 toggle-dtr yes
 toggle-dtr-waittime 500
 rings 2
 speed 115200
 modem-check-time 60

port ttyUSB2
 port-owner root
 port-group dialout
 port-mode 0660
 data-only yes
 ignore-carrier no
 toggle-dtr yes
 toggle-dtr-waittime 500
 rings 3
 speed 115200
 modem-check-time 60

port ttyUSB3
 port-owner root
 port-group dialout
 port-mode 0660
 data-only yes
 ignore-carrier no
 toggle-dtr yes
 toggle-dtr-waittime 500
 rings 4
 speed 115200
 modem-check-time 60
```
The `rings` parameter tells mgetty to answer the call after that many rings. They increase in the config so that all the modems dont try to answer at once, and you can prioritize which modems you want to be used most often.
 
5. Enable the `mgetty` service so it starts on boot for each device:

```
$ sudo systemctl enable mgetty@ttyUSB0.service
$ sudo systemctl enable mgetty@ttyUSB1.service
$ sudo systemctl enable mgetty@ttyUSB2.service
$ sudo systemctl enable mgetty@ttyACM0.service
```

6. Start `mgetty`:

```
$ sudo systemctl start mgetty@ttyUSB0.service
$ sudo systemctl start mgetty@ttyUSB1.service
$ sudo systemctl start mgetty@ttyUSB2.service
$ sudo systemctl start mgetty@ttyACM0.service
```

7. Configure `ppp` by editing `/etc/ppp/options`  
 Like above, comment out everything except these settings:

```
# Define the DNS server for the client to use
ms-dns 8.8.8.8
# async character map should be 0
asyncmap 0
# Require authentication
auth
# Use hardware flow control
crtscts
# We want exclusive access to the modem device
lock
# Show pap passwords in log files to help with debugging
show-password
# Require the client to authenticate with pap
+pap
# If you are having trouble with auth enable debugging
debug
# Heartbeat for control messages, used to determine if the client connection has dropped
lcp-echo-interval 30
lcp-echo-failure 4
# Cache the client mac address in the arp system table
proxyarp
# Disable the IPXCP and IPX protocols.
noipx
```

8. Create a device option file for each device by editing:  
 `/etc/ppp/options.ttyUSB0`  
 **Note**: The `192.168.32.xxx` network is simulated within `getty` and `ppp`. It only exists within this setup and does not have to match your local LAN!
  
 ```
 local
 lock
 nocrtscts
 192.168.32.1:192.168.32.2
 netmask 255.255.255.252
 noauth
 proxyarp
 lcp-echo-failure 60
 ```

`/etc/ppp/options.ttyUSB1`

```
local
lock
nocrtscts
192.168.32.5:192.168.32.6
netmask 255.255.255.252
noauth
proxyarp
lcp-echo-failure 60
```

`/etc/ppp/options.ttyUSB2`

```
local
lock
nocrtscts
192.168.32.9:192.168.32.10
netmask 255.255.255.252
noauth
proxyarp
lcp-echo-failure 60
```

`/etc/ppp/options.ttyUSB3`

```
local
lock
nocrtscts
192.168.32.13:192.168.32.14
netmask 255.255.255.252
noauth
proxyarp
lcp-echo-failure 60
```

Ensure that the IP addresses do not overlap across the device configurations. I'm using small /30 subnets (4 IP addresses, 2 usable) to separate each client.
 
9. Create the user for PAP authentication: `sudo useradd -G dialout,dip,users -m -g users -s /usr/sbin/pppd world`
10. Set a password: `sudo passwd world`
11. Edit `/etc/ppp/pap-secrets` and append the username and password (same as you entered above, quotes included):  
`world * "dialup" *`
12. Enable packet forwarding for IP4 by appending `net.ipv4.ip_forward=1` to `/etc/sysctl.conf`.
To enable the changes made in `sysctl.conf`, run `sysctl -p /etc/sysctl.conf`
13. The last step for the dial-up server is to configure the firewall to allow traffic forwarding from PPP out onto the network (and off to the Internet).
 * On Linux distributions with `iptables-nf` switch back to `iptables-legacy` by running `sudo update-alternatives --config iptables` and choosing `iptables-legacy` then follow the step below.
 * On Linux distributions with iptables (legacy), you need to add a line to /etc/rc.local to enable masquerading. If your Ethernet interface is named eth0, you would add this line:
   `iptables -t nat -A POSTROUTING -s 192.168.32.0/24 -o eth0 -j MASQUERADE`
 * On modern Ubuntu installs, `ufw` is used as a front-end to `iptables`, so the procedure is a bit different. Follow this guide, but you can omit `-o eth0` and use `-s 192.168.32.0/24`.

## WebOne Proxy

We are running a [WebOne Proxy](https://github.com/atauenis/webone) at `192.168.31.31:8080`. If it is added to the system/browser as an HTTP proxy, it will tunnel HTTPS sites over HTTP. For example, visiting `http://google.com` will transparently access `https://google.com`.

Installation can be performed as follows. This assumes an ARM-based system, but you can find the latest binary for your architecture here, https://github.com/atauenis/webone/releases.

```
$ wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
$ sudo dpkg -i packages-microsoft-prod.deb
$ sudo apt update
$ wget http://ftp.us.debian.org/debian/pool/main/i/icu/libicu63_63.1-6+deb10u3_arm64.deb
$ dpkg -i libicu63_63.1-6+deb10u3_arm64.deb
$ wget https://github.com/atauenis/webone/releases/download/v0.15.3/webone.0.15.3.linux-arm64.deb
$ sudo apt install ./webone.0.15.3.linux-arm64.deb
$ sudo systemctl start webone
$ sudo systemctl enable webone
$ sudo echo "net.ipv4.conf.all.route_localnet=1" >> /etc/sysctl.conf
$ sudo sysctl -p /etc/sysctl.conf
```

Now we need new firewall rules. The first rule is for our clients while the second is for the local system.

```
iptables -t nat -I PREROUTING --src 0/0 --dst 192.168.31.31 -p tcp --dport 8080 -j REDIRECT --to-ports 8080
iptables -t nat -I OUTPUT --src 0/0 --dst 192.168.31.31 -p tcp --dport 8080 -j REDIRECT --to-ports 8080
```
