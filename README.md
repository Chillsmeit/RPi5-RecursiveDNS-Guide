# WIP

## RaspberryPi5-Setup

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
- Find your RPi Mac address using `ip link show`
- Go to your router page, find the RPi mac adress and allocate a fixed IP to it (for ex 192.168.1.5)

### 4. Login with SSH
```
ssh username@fixedIPhere
```
### 5. Update system and reboot
```
sudo apt update -y && sudo apt upgrade -y && sudo reboot
```
### 6. Enable asking for sudo password
- Edit the following file with `sudo nano /etc/sudoers.d/010_pi-nopasswd`
- Change `"yourusername ALL=(ALL) NOPASSWD: ALL"` to `"yourusername ALL=(ALL) PASSWD: ALL"`
- Mind the spaces and don't change anything else, exit and save with CTRL+X then Y

### 7. Change default openssh port for increased security
- Edit the following file with `sudo nano /etc/ssh/sshd_config`
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
- Check ports being currently used `sudo ss -tupln`
- Check which service is using which portnumber `sudo lsof -i :PORT`

### 13. Make the RPi unpingable. 
- Edit the following file `sudo nano /etc/ufw/before.rules`
- In `"# ok icmp codes for INPUT"` add a new line below it with `"-A ufw-before-input -p icmp --icmp-type echo-request -j DROP"`

### 14. Restart RPi5 and SSH into it using the newly defined port
```
sudo reboot
ssh username@RPiIPHere -p SSHPortHere
```
### 15. Remove the motd for the RPi and add support for different terminals
- Own the motd file with `sudo chown -R $USER:$USER /etc/motd`
- Edit the motd with `nano ~/.bashrc`
- Always clear motd on login and make it persistent after system upgrades `echo -n > /etc/motd`
- Add support for kitty terminal `export TERM=xterm`
- Exit and save changes with CTRL+X then Y
- Reload bashrc with `source ~/.bashrc`

### 16. If there's erros messages like "expected kernel" etc.
- The reason is because 64 bits it too new in Raspbian and the packages complain about architecture
- The processor microcode module only supports AMD and Intel but not aarch64 (64bits ARM)
- You can "fix" this by removing the needrestart package `sudo apt-get purge needrestart`
- Or you can change this value:
```
sudo sed -i 's/#\$nrconf{ucodehints} = 0;/$nrconf{ucodehints} = 0;/' /etc/needrestart/needrestart.conf
```
### 17. Disable onboard Wifi if you're using ethernet:
- Edit this file with `sudo nano /boot/config.txt`
- Change this line `dtoverlay=disable-wifi`

### 18. Install Pi-Hole and use Google or Cloudflare DNS for now
```
curl -sSL https://install.pi-hole.net | bash
```
### 19. Reset PiHole Password
pihole -a -p

### 20. Install Unbound (recursive DNS)
sudo apt-get install unbound -y

### 21. Paste the default configuration found in https://docs.pi-hole.net/guides/dns/unbound/
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf

# Disable the following service because it can cause issues in Debian
sudo systemctl disable --now unbound-resolvconf.service
sudo sed -Ei 's/^unbound_conf=/#unbound_conf=/' /etc/resolvconf.conf
sudo rm /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf

# If you want to make PiHole your DHCP server. List your connections:
nmcli connection show

# Edit your ethernet connection
sudo nmcli connection edit "Wired connection 1"

# Now set the parameters according to your internet
set ipv4.method manual
set ipv4.addresses RPiIP/your_subnet
set ipv4.gateway GATEWAYIP
set ipv4.dns 127.0.0.1
save persistent
quit

# If you fucked it up, reset it with
sudo nmcli connection edit "Wired connection 1"
reset
save persistent
quit

# Open your router page
# Open Pihole interface in the settings->DHCP tab
# Turn off the DHCP server in your router and change primary DNS and secondary DNS to your RPi DNS. (don't forget to save)
# On Pihole DHCP tab, enable "DHCP server enabled"
# Choose an IP range, let's use 192.168.1.21 to 192.168.1.151
# For Router (gateway) IP address put your router IP address. 192.168.1.1
# Go to Settings -> DNS
# Tick Custom 1 (IPv4) and write "127.0.0.1#5335"
# Shuwdown your router and your RaspberryPi. Turn the router on and wait for a bit for it to startup, only then turn on the RPi
# Check if unbound is working
dig pi-hole.net @127.0.0.1 -p 5335

# Change PiHole from port 80 to something else
sudo nano /etc/lighttpd/conf-enabled/external.conf

# Add the following:
server.port := 22410

# Restart lighttpd
sudo service lighttpd restart

# Access admin interface with http://RPiIP:PORT/admin

# Enable Ports used by PiHole and Unbound
sudo ufw allow 53/tcp # DNS Resolution
sudo ufw allow 53/udp # DNS Resolution
sudo ufw allow 67/tcp # only needed if you want to use PiHole as DNS
sudo ufw allow 67/udp # only needed if you want to use PiHole as DNS
sudo ufw allow 443/tcp # for SSL/TLS certification with nginx
sudo ufw allow 5335/tcp # DNS over TLS Communication
sudo ufw allow 24830/tcp # Port you chose for the admin interface

# Enable the firewall
sudo ufw enable

# Install Neofetch
sudo apt install neofetch -y

# Enable temperatures in neofetch. Run this in terminal
sed -i 's/cpu_temp="off"/cpu_temp="C"/' ~/.config/neofetch/config.conf

# Fix neofetch regex to support RPi sensors:
sudo cp "/usr/bin/neofetch" "/usr/bin/neofetch.backup" && sudo sed -i "s/(coretemp|fam15h_power|k10temp)/(cpu_thermal|coretemp|fam15h_power|k10temp)/g" "/usr/bin/neofetch"

# Make neofetch show available disk space
sed -i 's/# info "Disk" disk/info "Disk" disk/' ~/.config/neofetch/config.conf

# Fix neofetch displaying Debian ASCII Art
sed -i 's/ascii_distro="auto"/ascii_distro="Raspbian"/' ~/.config/neofetch/config.conf

# run neofetch or add it to bashrc and load it
neofetch

# Install tldr and nala
sudo apt install nala -y && sudo apt install tldr -y && sudo apt install neovim -y

# Install Docker and Docker-Compose
sudo apt install docker -y
sudo apt install docker-compose -y

# Add User to the docker group
sudo usermod -aG docker "$USER"

# Enable docker autostart
sudo systemctl enable docker

# Reboot so that your user is added to the Docker group

# Check if you have docker-compose v2 by using:
docker-compose --version

# Update docker-compose to V2 directly from Github if you have v1
mkdir -p ~/.docker/cli-plugins/

# For 64 bits use this:
curl -SL https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-linux-aarch64 -o ~/.docker/cli-plugins/docker-compose

# For 32 bits use this:
curl -SL https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-linux-armv7 -o ~/.docker/cli-plugins/docker-compose

# Make the script executable
chmod +x ~/.docker/cli-plugins/docker-compose

# Install Portainer-CE.
# Create the directories
mkdir -p "~/Docker/portainer-ce"

# Cd into portainer folder:
cd Docker/portainer-ce

# Create the following file
nano docker-compose.yml

# Past the following inside, exit and save
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

# After exiting, do the following command:
docker compose up -d

# Go to your browser and enter http://RPiIP:PORTAINERPORT
# Choose a name for your user and password

# Create dockge folder
mkdir -p "~/Docker/dockge"

# cd into folder and create compose file
cd ~/Docker/dockge && nano docker-compose.yml

# Paste the following:
version: "3.8"
services:
  dockge:
    image: louislam/dockge:1
    restart: unless-stopped
    ports:
      # Host Port : Container Port
      - PORT:5001
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
      - /home/YOURUSERNAME/Docker:/home/YOURUSERNAME/Docker
    environment:
      # Tell Dockge where is your stacks directory
      - DOCKGE_STACKS_DIR=/home/YOURUSERNAME/Docker
