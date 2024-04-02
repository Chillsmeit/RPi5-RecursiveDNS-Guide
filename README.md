# WIP

## RPi5-RecursiveDNS-Guide and security practices

### 1. Download rpi-imager
- For the device choose RaspberryPi5<br>
- For the OS, go to "Raspberry Pi OS (other)" and choose Raspberry Pi OS Lite 64-bit.<br>

#### Links:
- [Windows Installer](https://downloads.raspberrypi.org/imager/imager_latest.exe)<br>
- [macOS Package](https://downloads.raspberrypi.org/imager/imager_latest.dmg)<br>
- [Linux Ubuntu package](https://downloads.raspberrypi.org/imager/imager_latest_amd64.deb)<br>
- [Linux flatpak](https://flathub.org/apps/org.raspberrypi.rpi-imager)<br>

### 2. Format SDCard with rpi-imager
- Choose the device + OS + storage
- Set up SSH login in the imager

### 3. After formatting
- Insert the SDCard in the Pi, connect the Pi with an ethernet cable and power it on
- Find your RPi Mac address with:
```
ip link show
```
- Go to your router page, find the RPi mac adress and allocate a static IP to it (for ex 192.168.1.5)

### 4. Login with SSH
```
ssh username@staticIPhere
```
### 5. Update system and reboot
```
sudo apt update -y && sudo apt upgrade -y && sudo reboot
```
### 6. Enable asking for sudo password
- Edit the following file with:
```
sudo nano /etc/sudoers.d/010_pi-nopasswd
```
- Change `"yourusername ALL=(ALL) NOPASSWD: ALL"` to `"yourusername ALL=(ALL) PASSWD: ALL"`
- Mind the spaces and don't change anything else, exit and save with CTRL+X then Y

### 7. Change default openssh port for increased security
- Edit the following file with:
```
sudo nano /etc/ssh/sshd_config
```
- Uncomment Port 22 and change it to your liking

### 8. Reboot to apply your changes
```
sudo reboot
```
### 9. Install uncomplicated firewall
```
sudo apt-get install ufw
```
### 10. Add the SSH port you chose earlier to UFW
```
sudo ufw allow SSHPort/tcp
```
### 11. Dissalow  six or more SSH connections within 30 seconds
```
sudo ufw limit SSHPort/tcp
```
### 12. Extra commands that might be useful later
- Check ports being currently used with:
```
sudo ss -tupln
```
- Check which service is using which portnumber with:
```
sudo lsof -i :PORT
```
### 13. Make the RPi unpingable. 
```
sudo nano /etc/ufw/before.rules
```
- In `"# ok icmp codes for INPUT"` add a new line below it with `"-A ufw-before-input -p icmp --icmp-type echo-request -j DROP"`

### 14. Restart RPi5 and SSH into it using the newly defined port
```
sudo reboot
ssh username@RPiIPHere -p SSHPortHere
```
### 15. Remove the motd for the RPi and add support for different terminals
- Own the motd file with:
```
sudo chown -R $USER:$USER /etc/motd
```
- Edit the motd with:
```
nano ~/.bashrc
```
- To clear motd on login and make it persistent after system upgrades with add this to .bashrc `echo -n > /etc/motd`
- To add support for kitty terminal add this too `export TERM=xterm`
- Exit and save changes with CTRL+X then Y
- Reload bashrc with:
```
source ~/.bashrc
```

### 16. If there's erros messages like "expected kernel" etc.
- At the time of writting, some Debian 12 (bookworm) packages complain about the architecture
- The processor microcode module only supports AMD and Intel but not aarch64 (64bits ARM)
- You can "fix" this by removing the needrestart package with:
```
sudo apt-get purge needrestart
```
- Or you can change the following value:
```
sudo sed -i 's/#\$nrconf{ucodehints} = 0;/$nrconf{ucodehints} = 0;/' /etc/needrestart/needrestart.conf
```
### 17. Disable onboard Wifi if you're using ethernet:
```
sudo nano /boot/config.txt
```
- Change the line `dtoverlay=disable-wifi`
### 18. Install Pi-Hole
- Install Pi-Hole with (choose Google or Cloudflare DNS for now):
```
curl -sSL https://install.pi-hole.net | bash
```
- Reset PiHole Password with:
```
pihole -a -p
```
### 19. Install Unbound (recursive DNS)
- Install Unbound with:
```
sudo apt-get install unbound -y
```
- Copy the config found [here](https://docs.pi-hole.net/guides/dns/unbound/#configure-unbound)
- Paste it into here:
```
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

### 20. Disable the following Unbound service because it can cause issues in Debian
```
sudo systemctl disable --now unbound-resolvconf.service
sudo sed -Ei 's/^unbound_conf=/#unbound_conf=/' /etc/resolvconf.conf
sudo rm /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
```

### 21. If you want to make PiHole your DHCP server (optional)
- Open PiHole interface -> Go to the settings tab and then DHCP tab
- Enable "DHCP server enabled"
- Choose an IP range, let's use 192.168.1.21 to 192.168.1.151
- For Router (gateway) IP address put your router IP address. 192.168.1.1
- Open your router page
- Turn off the DHCP server in your router and change primary DNS and secondary DNS to your RPi . (don't forget to save)

### 22. Configure Unbound (recursive DNS)
- List your connections on RPi with `nmcli connection show`
- Edit your ethernet connection `sudo nmcli connection edit "Wired connection 1"` in my case
- Now set the parameters according to your connection:
```
set ipv4.method manual
set ipv4.addresses RPiIP/your_subnet
set ipv4.gateway RouterIP
set ipv4.dns 127.0.0.1
save persistent
quit
```
- To revert any changes:
```
sudo nmcli connection edit "Wired connection 1"
reset
save persistent
quit
```
- Go to PiHole Interface -> Settings -> DNS
- Tick Custom 1 (IPv4) and write `127.0.0.1#5335`

### 23. Check if unbound is working
- Shuwdown your router and your RaspberryPi.
- Turn your router on first.
- Turn on the RPi
- Use the following command to check if unbound is working `dig pi-hole.net @127.0.0.1 -p 5335`

### 24. Change PiHole WebInterface from port 80 to something else
```
sudo nano /etc/lighttpd/conf-enabled/external.conf
```
- Add the following: `server.port := YOURPORTHERE`
- Restart lighttpd `sudo service lighttpd restart`

### 25. Enable Ports used by PiHole and Unbound in UFW (optional)
```
sudo ufw allow 53/tcp # DNS Resolution
sudo ufw allow 53/udp # DNS Resolution
sudo ufw allow 67/tcp # only needed if you want to use PiHole as DNS
sudo ufw allow 67/udp # only needed if you want to use PiHole as DNS
sudo ufw allow 443/tcp # for SSL/TLS certification with nginx
sudo ufw allow 5335/tcp # DNS over TLS Communication
sudo ufw allow YOURPORT/tcp # Port you chose for the admin interface
```
- Enable the firewall with:
```
sudo ufw enable
```

### 26. Install Neofetch & configure neofetch (Optional)
```
sudo apt install neofetch -y
```
- Enable temperatures in neofetch with:
```
sed -i 's/cpu_temp="off"/cpu_temp="C"/' ~/.config/neofetch/config.conf
```
- Fix neofetch regex to support RPi sensors:
```
sudo cp "/usr/bin/neofetch" "/usr/bin/neofetch.backup" && sudo sed -i "s/(coretemp|fam15h_power|k10temp)/(cpu_thermal|coretemp|fam15h_power|k10temp)/g" "/usr/bin/neofetch"
```
- Make neofetch show available disk space:
```
sed -i 's/# info "Disk" disk/info "Disk" disk/' ~/.config/neofetch/config.conf
```
- Make neofetch display RaspbianOS ASCII Art instead of Debian with:
```
sed -i 's/ascii_distro="auto"/ascii_distro="Raspbian"/' ~/.config/neofetch/config.conf
```
### 27. Add neofetch it to bashrc (Optional)
- Open .bashrc with:
```
nano ~/.bashrc
```
- Add this line in the end of the file, save and exit `neofetch`
- Reload bashrc with:
```
source ~/.bashrc
```
### 28. Install tldr, nala and neovim (Optional)
```
sudo apt install nala -y && sudo apt install tldr -y && sudo apt install neovim -y
```
### 29. Install & configure Docker and Docker-Compose
```
sudo apt install docker -y && sudo apt install docker-compose -y
```
- Add your current user to the docker group with:
```
sudo usermod -aG docker "$USER"
```
- Enable docker autostart
```
sudo systemctl enable docker
```
- Reboot your device so that your user is added to the Docker group
```
sudo reboot
```
- Check if you have docker-compose v2 by using:
```
docker-compose --version
```
- Update docker-compose to V2 directly from Github if you have v1
```
mkdir -p ~/.docker/cli-plugins/
```
- For a 64 bits ARM RPi use this:
```
curl -s https://api.github.com/repos/docker/compose/releases/latest | grep -o '"tag_name": "[^"]*"' | sed 's/"tag_name": "\(.*\)"/\1/' | xargs -I{} curl -LO https://github.com/docker/compose/releases/download/{}/docker-compose-linux-aarch64
```
- For a 32 bits ARMv7 RPi use this:
```
curl -s https://api.github.com/repos/docker/compose/releases/latest | grep -o '"tag_name": "[^"]*"' | sed 's/"tag_name": "\(.*\)"/\1/' | xargs -I{} curl -LO https://github.com/docker/compose/releases/download/{}/docker-compose-linux-armv7
```
- For a 32 bits ARMv6 RPi use this:
```
curl -s https://api.github.com/repos/docker/compose/releases/latest | grep -o '"tag_name": "[^"]*"' | sed 's/"tag_name": "\(.*\)"/\1/' | xargs -I{} curl -LO https://github.com/docker/compose/releases/download/{}/docker-compose-linux-armv6
```
- Make the script executable with:
```
chmod +x ~/.docker/cli-plugins/docker-compose
```
### 31. Install Portainer-CE (Community Edition)
- Create the directories and cd into them:
```
mkdir -p "~/Docker/portainer-ce" && cd "~/Docker/portainer-ce"
```
- Create the following file
```
nano docker-compose.yml
```
- Paste the following inside (Change the PORT into a port you want), exit and save:
```
version: '3.9'
services:
    portainer-ce:
        image: portainer/portainer-ce
        container_name: portainer
        command: -H unix:///var/run/docker.sock
        ports:
            - 'PORT:9000'
        volumes:
            - 'portainer_data:/data'
            - '/var/run/docker.sock:/var/run/docker.sock'
        restart: always
volumes:
    portainer_data:
```
- After exiting, do the following command:
```
docker compose up -d
```
- Go to your browser and enter http://RPiIP:PORTAINERPORT
- Choose a name for your user and password
