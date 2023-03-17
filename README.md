# Setting up a LXC container with NordVPN, Killswitch Firewall and Transmission
Baseline is that you know how to create a LXC container with privileged settings. So create such a thing first. I used 1 CPU, 1GB RAM and 2GB Disk.

In this example "alpine-3.16-default_20220622_amd64.tar.xz" was used.

## Optional: Setup mounted share
If you want to download files to a mounted SMB share you need to do the following:
~~~
apk add cifs-utils
mkdir /torrents
~~~

Add the following to "/etc/fstab". Customize for your environment.
```
//192.168.178.30/torrents    /torrents    cifs    uid=0,gid=0,user=myusername,password=*****,_netdev 0 0
```
In my case enabling netmount didn't ensure that the share was automounted, so I created a custom starter script.

Create "/etc/local.d/mounter.start" with the following content:
```
mount -a
```
And add the service:
~~~
rc-update add local default
~~~
Now  the share should automount upon boot.

## Setup firewall and Openvpn with NordVPN
To ensure the container is only communicating with the internet via the VPN, the following needs to be done. We also allow communication to the container via LAN without the VPN.

Install the firewall.
~~~
apk add ip6tables ufw
~~~
Add the following to "/etc/sysctl.conf" to disable ipv6.
~~~
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1
~~~
Update sysctl
~~~
sysctl -p
~~~
You can now check if "/proc/sys/net/ipv6/conf/all/disable_ipv6" is set to 1.

Also disable ipv6 in ufw by changing "IPV6=yes" in "/etc/default/ufw" to:
~~~
IPV6=no
~~~
And now temporarily disable ufw:
~~~
ufw disable
~~~

In order to setup a VPN kill switch in UFW, you need three pieces of information:
-   The **public IP address** of the VPN server you connect to
-   The **port and protocol** your server uses to communicate
-   The **subnet** of your local network
You can download NordVPN config files from: https://nordvpn.com/da/ovpn/
Download the one you want or all of them and place them in "/etc/openvpn". Name the one you want to use and place it at "/etc/openvpn/nordvpn.ovpn"

The firewall rules to allow LAN and disable everything to WAN except to the VPN-ip(156.67.85.16) on port 1194(udp):
~~~
ufw allow in to **192.168.1.0/24**
ufw allow out to **192.168.1.0/24**
ufw default deny outgoing
ufw default deny incoming
ufw allow out to 156.67.85.16 port 1194 proto udp
ufw allow out on tun0 from any to any
~~~

Then you need your NordVPN username and password. Find at: https://my.nordaccount.com/dashboard/nordvpn/

On proxmox server you need to add this setting to your LXC container. Add to "/etc/pve/lxc/[Container ID].conf" and reboot lxc container
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.hook.autodev: sh -c "modprobe tun; cd ${LXC_ROOTFS_MOUNT}/dev; mkdir net; mknod net/tun c 10 200; chmod 0666 net/tun"
```
Now create the file "/etc/openvpn/creds" and add your own credentials for NordVPN:
~~~
username
password
~~~
And then create a script that starts openvpn upon boot by creating "/etc/local.d/openvpn.start" and add the following:
~~~
openvpn --config /etc/openvpn/nordvpn.ovpn --auth-user-pass /etc/openvpn/creds
~~~

## Transmission
Install Transmission-daemon as bittorrent client:
~~~
apk add transmission-daemon
~~~

If you are using a SMB mounted share, you might run into permissions problems as Transmission-daemon is running as the user "transmission". You can fix this multiple ways, but the quick-fix way was to change the permissions of the daemon in the file "/etc/runlevels/default/transmission-daemon".
From:
~~~
runas_user=${runas_user:-transmission:transmission}
~~~~
To
~~~
runas_user=${runas_user:-root:root}
~~~
And remember to change "/var/lib/transmission/config/settings.json" to reflect your setup. I changed:
~~~
"download-dir": "/torrents/Downloads",
"incomplete-dir": "/torrents/Incomplete",
"incomplete-dir-enabled": true,
"rpc-host-whitelist": "192.168.1.*,127.0.0.1",
"rpc-password": "mypassword",
"rpc-username": "myusername",
"rpc-whitelist": "192.168.1.*,127.0.0.1,::1",
~~~

Now enable the firewall with:
~~~
ufw enable
~~~

And then reboot your container.

After that you should be good to go with your new seedbox.

## Links
https://janhapke.com/blog/mount-cifs-samba-fstab-alpine-linux/
https://www.comparitech.com/blog/vpn-privacy/how-to-make-a-vpn-kill-switch-in-linux-with-ufw/
https://forum.proxmox.com/threads/pve-7-openvpn-lxc-problem-cannot-open-tun-tap-dev.103081/
https://wiki.alpinelinux.org/wiki/Setting_up_Transmission_(bittorrent)_with_Clutch_WebUI
