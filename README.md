# RaspBerry-PiHole
Use Raspberry as ads block


## Assign static ip
Proceed to modify the “*dhcpcd.conf*” configuration file by running the command below. This config file allows us to modify the way the Raspberry Pi handles the network.
```
sudo nano /etc/dhcpcd.conf
```
First, you have to decide if you want to set the static IP for your “eth0” (Ethernet) connector or you “wlan0” (WiFi) connection. Decide which one you want and replace “&lt;NETWORK&gt;” with it.

Make sure you replace “&lt;STATICIP&gt;” with the IP address that you want to assign to your Raspberry Pi. Make sure this is not an IP that could be easily attached to another device on your network.

Replace “&lt;ROUTERIP&gt;” with the IP address that you retrieved in step 1 of this tutorial

Finally, replace “&lt;DNSIP&gt;” with the IP of the domain name server you want to utilize. This is either the IP you got in step 2 of this tutorial or another one such as Googles “8.8.8.8” or Cloudflare’s “1.1.1.1“.
```
interface <NETWORK>
static ip_address=<STATICIP>/24
static routers=<ROUTERIP>
static domain_name_servers=<DNSIP>
```
Now that we have modified our Raspberry Pi’s DHCP configuration file so that we utilize a static IP address, we need to go ahead and restart the Raspberry Pi.

Restarting the Raspberry Pi will allow our configuration changes to be loaded in and the old ones flushed out.

Upon rebooting, the Raspberry Pi will attempt to connect to the router using the static IP address we defined in our “dhcpd.conf” file.

Run the following command to restart your Raspberry Pi.
```
sudo reboot
```

## Disable Bluetooth and WiFi

```
sudo nano boot/config.txt
```
If you want to disable both wifi and bluetooth, you need to add these 2 lines :
```
dtoverlay=disable-wifi
dtoverlay=disable-bt
```

## Install PiHole
To setup Pi Hole, from the command prompt (locally or remotely through SSH) use the following commands in sequence:
```
wget -O basic-install.sh https://install.pi-hole.net
sudo bash basic-install.sh
```

### Pi-hole as All-Around DNS Solution
```
wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints
```
Then edit the file `/etc/unbound/unbound.conf.d/pi-hole.conf` as follow:
```
erver:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to yes if you have IPv6 connectivity
    do-ip6: no

    # You want to leave this to no unless you have *native* IPv6. With 6to4 and
    # Terredo tunnels your web browser should favor IPv4 for the same reasons
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    # If you use the default dns-root-data package, unbound will find it automatically
    #root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # Suggested by the unbound man page to reduce fragmentation reassembly problems
    edns-buffer-size: 1472

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```
Start your local recursive server and test that it's operational:
```
sudo service unbound restart
dig pi-hole.net @127.0.0.1 -p 5335
```
Finally, configure Pi-hole to use your recursive DNS server by specifying 127.0.0.1#5335 as the Custom DNS (IPv4):
![RecursiveResolver](./img/RecursiveResolver.png)

#### Add logging to unbound
There are five levels of verbosity
```
Level 0 means no verbosity, only errors
Level 1 gives operational information
Level 2 gives  detailed operational  information
Level 3 gives query level information
Level 4 gives  algorithm  level  information
Level 5 logs client identification for cache misses
```
First, specify the log file and the verbosity level in the *server* part of `/etc/unbound/unbound.conf.d/pi-hole.conf`:
```
server:
    # If no logfile is specified, syslog is used
    logfile: "/var/log/unbound/unbound.log"
    verbosity: 1
```
Second, create log dir and file, set permissions:
```
sudo mkdir -p /var/log/unbound
sudo touch /var/log/unbound/unbound.log
sudo chown unbound /var/log/unbound/unbound.log
```
Third, restart unbound:
```
sudo service unbound restart
```