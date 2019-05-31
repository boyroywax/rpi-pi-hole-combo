# Raspberry Pi DNS Server/No-IP/OpenVPN/Pi-Hole/DNSCrypt

## 1. Get a static ip address - no-ip.com
* http://www.darwinbiler.com/dynamic-dns-using-raspberry-pi/

Starting with a fresh Raspbian Install

```bash
sudo apt-get update && \
sudo apt-get upgrade -y
```



```bash
cd /usr/local/src/ && \
sudo wget http://www.no-ip.com/client/linux/noip-duc-linux.tar.gz && \
sudo tar xf noip-duc-linux.tar.gz && \
cd noip-2.1.9-1/ && \
sudo make install
```
If the installer did not already update the noip2 config
```bash
sudo /usr/local/bin/noip2 -C
```
Start the client
```bash
sudo /usr/local/bin/noip2
```
Check the status of the the noip2 service
```bash
tail /var/log/syslog
```

Cleanup the src file

```bash
cd /usr/local/src/ && \
sudo rm -r /usr/local/src/noip*
```



## 2. Install OpenVPN

* https://itchy.nl/raspberry-pi-3-with-openvpn-pihole-dnscrypt
* https://github.com/Nyr/openvpn-install
* https://raspberrytips.com/raspberry-pi-dns-server/

Download the Installer, and begin the installation.  for the external hostname use your no-ip address.

```bash
sudo wget https://git.io/vpn -O openvpn-install.sh && \
sudo chmod 755 openvpn-install.sh && \
sudo ./openvpn-install.sh
```
Find the tun0 interface
```bash
ifconfig tun0 | grep 'inet'
```
Edit OpenVPN server config.
```bash
sudo nano /etc/openvpn/server.conf
```
Add the tun0 interface IP address, PiHole will be using it.
```bash
push "dhcp-option DNS 10.8.0.1"
```
Comment out all other ```push "dhcp-option DNS...``` references by adding a ```#```infront of them.
Restart OpenVPN server.
```bash
sudo systemctl restart openvpn
```

Enable OpenVPN acccess from outside of LAN by port forwarding the openVPN port you selected in setup.  Default port is 1149

* https://medium.freecodecamp.org/running-your-own-openvpn-server-on-a-raspberry-pi-8b78043ccdea

## 3. Install pi-Hole

Easy install using script

```bash
sudo curl -sSL https://install.pi-hole.net | sudo bash
```

Install Notes: Use Level3 Upstream DNS Server, and no-ip ip address.  You should get an output similiar to this: (edited for formatting)

```text
Configure your devices to use the Pi-hole as their DNS server using:                                                            
IPv4: 192.168.0.23                                          
IPv6: 2601:603:207f:aef0:394:638e:4c09:9e1f
If you set a new IP address, you should restart the Pi.            
The install log is in /etc/pihole.
View the web interface at http://pi.hole/admin or                  
http://192.168.0.XX/admin                                          
Your Admin Webpage login password is XXXXXXXX
```

Enable DHCP on the Raspberry Pi-hole:

1. Log into Pi-hole admin panel and enable DHCP in ```Settings > DHCP```
2. Also in Pi-hole admin panel in ```Setttings > DNS``` under ```Interface listening behavior``` tick the last option, **Listen on all interfaces**.
3. Disable DHCP on your modem or router/modem combo. 
4. Save both configurations and restart both devices.

Now all devices on your LAN will automatically use the Pi-Hole service.

## 4. Install DNSCrypt

* https://github.com/jedisct1/dnscrypt-proxy/releases

1. Downlaod, untar, and rename the prebuilt binary.
	```bash
   cd /opt && \
   sudo wget https://github.com/jedisct1/dnscrypt-proxy/releases/download/2.0.23/dnscrypt-proxy-linux_arm-2.0.23.tar.gz && \
   sudo tar -xf dnscrypt-proxy-linux_arm-2.0.23.tar.gz && \
   sudo rm -r dnscrypt-proxy-linux_arm-2.0.23.tar.gz && \
   sudo mv linux-arm dnscrypt-proxy
  ```
2. Create a config file using ```example-dnscrypt-proxy.toml``` .
  ```bash
  cd dnscrypt-proxy && \
  sudo cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml
  ```
3. Edit the toml file. 
	```bash
	sudo nano dnscrypt-proxy.toml
	```
	* Edit the port, since ```53``` is already being used by Pi-Hole. This is the ```listen_addresses``` line. Set ```listen_addresses = ['127.0.0.1:54','[::1]:54']``` .
	* Set ```require_dnssec = true```.
	* Set ```server_names = ['dnscrypt.nl-ns0']```.
4. Install dnscrypt-proxy service.
  ```bash
  sudo ./dnscrypt-proxy -service install
  ```
5. Start the new service.
   ```bash
   sudo ./dnscrypt-proxy -service start
   ```

## 5. Configure Pi-hole to use DNSCrypt

1. Login to Pi-Hole admin dashboard
2. Settings > DNS under "Upstream DNS Server" header.
   * Set **Custom 1 (IPv4)** to ```127.0.0.1#53``` 
   * Set **Custom 3 (IPv6)** to ```::1#54 ```
3. Reboot Raspberry Pi.

## 6. Hardening Secruity



## Connecting to OpenVPN Server from WAN

### On MacOSX:

1. Download TunnelBlick https://tunnelblick.net/release/Tunnelblick_3.7.8_build_5180.dmg
2. Drag-and-Drop your ```.ovpn``` file into the configuration pane on the left side.

### On iOS:

1. Download OpenVPN app from Apple App Store.
2. Load ```.ovpn``` file into your iCloud files.
3. Open ```.ovpn``` file in OpenVPN app.
4. Enable Connection.

### On Windows:

* https://pimylifeup.com/raspberry-pi-vpn-server/

## Checking OpenVPN connected users

* https://superuser.com/questions/819729/list-which-vpn-clients-are-connected

Install the required packages:

```bash
sudo apt-get update && \
sudo apt-get install -y telnet expect
```

Create ```openVPNuserlist.sh```:

```bash
#!/usr/bin/expect
spawn telnet localhost 7505
set timeout 10
expect "OpenVPN Management Interface"
send "status 3\r"
expect "END"
send "exit\r"
```

Create a ```Makefile``` to run the script easier:

```makefile
default:
	while true; do ./openVPNUserlist.sh |grep -e ^CLIENT_LIST; sleep 1; done
```

Add the management settings to the config file

```bash
echo "management localhost 7505" | sudo tee -a /etc/openvpn/server.conf
```

Also, Add the keepalive settings to the config file

```bash
echo "keepalive 10 60" | sudo tee -a /etc/openvpn/server.conf
```

Start the script by running the Makefile

```bash
make
```



## Working with OpenVPN

### Create another vpn user account using ```openvpn-install.sh```

```bash
sudo ./openvpn-install.sh
```

Press 1 and ENTER. Type in the name of the new user. Done.

### Copy openVPN keys to device

Hit the following command on your raspberry pi.

```bash
sudo cp /root/KEYNAME.ovpn /home/pi
```

Now, on your second computer SFTP the ```KEYNAME.ovpn```.

```bash
sftp pi@raspberrypi.local
> get /home/pi/KEYNAME.ovpn .
> lpwd 
> # STDOUT EXAMPLE '/User/localmachineuser/home/'
```

```lpwd``` displays the directory that the KEYFILE.ovpn was copied into.

### Extra: Create a fresh server.conf manually

Backup your existing server.conf first.

```bash
sudo mv /etc/openvpn/server.conf /etc/openvpn/backup-server.conf
```

Generate and move the server.conf from the sample config files

```bash
sudo bash -c "gunzip -c '/usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz' > '/etc/openvpn/server.conf'"
```

- http://askubuntu.com/questions/823092/ddg#823094

### Extra: Obfuscation Proxy Install (Obfs4)

* https://medium.freecodecamp.org/running-your-own-openvpn-server-on-a-raspberry-pi-8b78043ccdea


