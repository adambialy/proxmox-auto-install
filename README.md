# proxmox-auto-install
Description how to use pve auto install, as informations on proxmox website not entirely clear.

OBJECTIVE
---------

Central server allowing bare metal to pxeboot proxmox image, with baked http autoinstall.
One server will have all necessary services running - apache2, dhcp, tftp, ansible.
In essence, bare metal pve node is booting up from pxe, downloading configuration (answer file) from http based on mac address.
Inside answer file will be generic setup with networking, stroage, ssh key (for ansible) etc.
Once server started, ansible will provide rest of the configuration - configure network bonds etc, install tools and packages ssh keys etc.
After these steps the node will be ready to join the cluster. (theory)



(still) TODO
------------

PXEboot for PVE

https://github.com/morph027/pve-iso-2-pxe



PREREQ
------

For this test I created 3 vm's on ProxmoxVE.

1. One for dhcp/web server.

2. Two for test subjects (ProxmoxVE nodes auto install)

I noted the mac addresses of the node machines: 

`pveauto1 b6:84:63:f0:37:5d`

`pveauto2 ea:20:fe:06:2f:41`

All machines for simplicity inside same subnet/vlan


ISO generation
--------------

Download latest iso from Proxmox website.

Create iso with autoanswer provided by http request:

`proxmox-auto-install-assistant prepare-iso /root/proxmox-ve_8.2-2.iso --fetch-from http`

``
Copying source ISO to temporary location...
Preparing ISO...
Moving prepared ISO to target location...
Final ISO is available at "/root/proxmox-ve_8.2-2-auto-from-http.iso"
``



PKG | install packages
----------------------

Install dhcp server and web server

`apt install apache2 isc-dhcp-server -y`

---

Install proxmox tools:

create /etc/apt/sources.list.d/proxmox.list with content below

``deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription``

add key to apt:

``wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg``

`apt update`

Install proxmox auto install package

`apt install proxmox-auto-install-assistant -y`

Install ansible (optional)

`apt install ansible -y`

Install packages required for pxe boot manipulation

`apt install cpio file zstd gzip genisoimage -y`


SSL | create cert for apache2
-----------------------

Enable ssl module and enable ssl site

`a2enmod ssl`

`a2ensite default-ssl`

generate ssl cert/key

`openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=172.19.3.168"`

copy cert and key:

`cp cert.pem /etc/ssl/certs/`

`cp key.pem /etc/ssl/private/`

Grab sha256 fingerprint from certificate (will be needed in dhcp config later)

`openssl x509 -in cert.pem -fingerprint -sha256 -noout | tr -d ":"
sha256 Fingerprint=8C3558AEF51C4EDE20442C1EBA447724EAC28C5050724E8E2BE56892F481E90A`



install cert:


/etc/apache2/sites-enabled/default-ssl.conf

```
<VirtualHost *:443>

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html
	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

	SSLEngine on
	SSLCertificateFile      /etc/ssl/certs/cert.pem
	SSLCertificateKeyFile   /etc/ssl/private/key.pem


	<FilesMatch "\.(?:cgi|shtml|phtml|php)$">
		SSLOptions +StdEnvVars
	</FilesMatch>
	<Directory /usr/lib/cgi-bin>
		SSLOptions +StdEnvVars
	</Directory>
	<Directory "/var/www/html">
	    Options Indexes MultiViews
	    AllowOverride None
	    Require all granted
	</Directory>

</VirtualHost>

```

restart apache

`systemctl restart apache2`


copy answer.toml to different directories for different hosts:

`cp answer.toml /var/www/html/b68463f0375d/`

`cp answer.toml /var/www/html/ea20fe062f41/`



Example answer.toml below. Obviously at least ip (network/cidr) need to be changed, as well as host name (global/fqdn). All depend of your config.

```
[global]
keyboard = "en-gb"
country = "gb"
fqdn = "pveauto2.testinstall"
mailto = "mail@no.invalid"
timezone = "Europe/London"
root_password = "test123"
root_ssh_keys = [
    "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN9jmxgsR72FMAwzAYMkobHmOvJS09G9QgApiuyiiuyi"
]

[network]
source = "from-answer"
cidr = "192.168.99.102/24"
dns = "192.168.10.15"
gateway = "192.168.99.254"
filter.ID_NET_NAME = "ens18"

[disk-setup]
filesystem = "ext4"
lvm.swapsize = 0
lvm.maxvz = 0
disk_list = ["sda"]

```

answer.toml documentation | https://pve.proxmox.com/wiki/Automated_Installation#Answer_File_Format_2


DHCP
----

Make sure interface is specified in defaults

`/etc/default/isc-dhcp-server`

```
# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server)

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPDv4_PID=/var/run/dhcpd.pid
#DHCPDv6_PID=/var/run/dhcpd6.pid

# Additional options to start dhcpd with.
#	Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#	Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4="ens18"
INTERFACESv6=""
```

Edit dhc conf to your needs

`/etc/dhcp/dhcpd.conf`


```
option domain-name-servers 1.1.1.1, 8.8.8.8;
option pve-auto-url code 250 = text;
option pve-auto-url-fp code 251 = text;

default-lease-time 600;
max-lease-time 7200;

ddns-update-style none;

subnet 172.19.3.128 netmask 255.255.255.128 {
  range 172.19.3.130 172.19.3.150;
  option domain-name-servers 1.1.1.1;
  option routers 172.19.3.254;
# disabled general answer file
# option pve-auto-url "https://172.19.3.168/answer.toml";
  option pve-auto-url-fp "8c3558aef51c4ede20442c1eba447724eac28c5050724e8e2be56892f481e90a";
  option broadcast-address 172.19.3.255;
  default-lease-time 450;
  max-lease-time 600;
  filename "pxelinux.0";
}

host pveauto1 {
  hardware ethernet b6:84:63:f0:37:5d;
  fixed-address 172.19.3.131;
  option pve-auto-url "https://172.19.3.168/b68463f0375d/answer.toml";
}

host pveauto2 {
  hardware ethernet ea:20:fe:06:2f:41;
  fixed-address 172.19.3.132;
  option pve-auto-url "https://172.19.3.168/ea20fe062f41/answer.toml";
}
```

  > NOTE: There are mistakes in documentation with certificate fingerprint and url. 1. URL needs to be specified with full path to anwer file. 2. Certificate fingerprint required in sha254, string without ":"

restart dhcp server

`systemctl restart isc-dhcp-server.service`

PXE boot
========

```
./pve-iso-2-pxe.sh /root/proxmox-ve_8.2-2-auto-from-http.iso 

#########################################################################################################
# Create PXE bootable Proxmox image including ISO                                                       #
#                                                                                                       #
# Author: mrballcb @ Proxmox Forum (06-12-2012)                                                         #
# Thread: http://forum.proxmox.com/threads/8484-Proxmox-installation-via-PXE-solution?p=55985#post55985 #
# Modified: morph027 @ Proxmox Forum (23-02-2015) to work with 3.4                                      #
#########################################################################################################

Using proxmox-ve_8.2-2-auto-from-http.iso...
Using proxmox-ve_8.2-2.iso...
ln: failed to create symbolic link 'proxmox.iso': File exists
extracting kernel...
extracting initrd...
adding iso file ...
2728961 blocks
Finished! pxeboot files can be found in /root.
```



<h1>Links</h1>

https://serverfault.com/questions/1148217/debian-12-bookworm-from-ipxe

https://forum.proxmox.com/threads/proxmox-installation-via-pxe-solution.8484/#post-55985



