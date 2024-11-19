# proxmox-auto-install
description how to use pve auto install


install packages
----------------

`apt install apache2 isc-dhcp-server -y`


create cert for apache2
-----------------------

`openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=172.19.3.168"`

install cert and restart apache2


copy answer.toml to apache root dir for testing.




  answer.toml

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
#  option pve-auto-url "https://172.19.3.168/answer.toml";
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


