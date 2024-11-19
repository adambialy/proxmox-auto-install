# proxmox-auto-install
Description how to use pve auto install, as informations on proxmox website not entirely clear.

TODO
----

PXEboot for PVE, and then follow aut install

https://github.com/morph027/pve-iso-2-pxe



PREREQ
------

For this test I created 3 vm's on ProxmoxVE.

one for dhcp/web server, and two for test subjects (ProxmoxVE nodes auto install)

I noted the mac addresses of the node machines: 

`pveauto1 b6:84:63:f0:37:5d`

`pveauto2 ea:20:fe:06:2f:41`

All machines for simplicity inside same subnet/vlan


ISO generation
--------------

Download latest iso from Proxmox website.

Create iso with autoanswer provided by http request:

`proxmox-auto-install-assistant prepare-iso /mnt/pve/pnfs/template/iso/proxmox-ve_8.2-2-auto-from-iso.iso --fetch-from http`



PKG | install packages
----------------------

`apt install apache2 isc-dhcp-server -y`


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



  dhcpd.conf

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

* NOTE: There are mistakes in documentation with certificate fingerprint and url. 1. URL needs to be specified with full path to anwer file. 2. Certificate fingerprint required in sha254, string without ":"

restart dhcp server

`systemctl restart isc-dhcp-server.service`
