# RaspberryPi5-Setup

# Download rpi-imager
# Choose your device, use Raspberry Pi OS Lite 64-bit if your device supports 64-bits.
Windows: https://downloads.raspberrypi.org/imager/imager_latest.exe
Linux package: sudo apt install rpi-imager
Linux flatpak: https://flathub.org/apps/org.raspberrypi.rpi-imager

# Format SDCard with rpi-imager and set up SSH login in the options
# After the format is done, insert the SDCard in the Pi, connect the Pi with an ethernet cable and power it on
# Go to your router, find the RPi mac adress and allocate a fixed IP to it
# To find your RPi Mac Address lets list your network interfaces
ip link show

# Now assign a fixed IP to the RPi. Reboot the Rpi after you're done to get your new IP

# Login with ssh
ssh username@RPiIPHere

# Update system and reboot
sudo apt update -y && sudo apt upgrade -y && sudo reboot

# By default security in Raspbian is a bit lax.

# Enable asking for sudo password
# Edit the following file and change "yourusername ALL=(ALL) NOPASSWD: ALL" to "yourusername ALL=(ALL) PASSWD: ALL"
# Remove only the NO, make sure you don't change anything else, mind the spaces, exit and save (CTRL+X to leave and Y to confirm)
sudo nano /etc/sudoers.d/010_pi-nopasswd

# Change default openssh port in /etc/ssh/sshd_config
# Uncomment Port 22 and change it to something else.
sudo nano /etc/ssh/sshd_config

# Reboot
sudo reboot

# Install uncomplicated firewall
sudo apt-get install ufw

# Check ports being currently used
sudo ss -tupln

# Check which service is using which portnumber
sudo lsof -i :PORT

# Add the SSH port you chose earlier to UFW
sudo ufw allow SSHPORT/tcp

# Limit the ssh port to not allow six or more connections within 30 seconds.
sudo ufw limit SSHPORT/tcp

# Let's make the RPi unpingable. 
sudo nano /etc/ufw/before.rules
# In "# ok icmp codes for INPUT" add a new line below it and write this "-A ufw-before-input -p icmp --icmp-type echo-request -j DROP"

# Restart RPi5 using sudo reboot and SSH into it using:
ssh username@RPiIPHere -p SSHPORT

# Clear the annoying motd. First own the file for your user.
sudo chown -R $USER:$USER /etc/motd

# Edit .bashrc
nano ~/.bashrc

# Add the following to always clear motd on login and make it persistent after system upgrades
echo -n > /etc/motd

# Give support for other terminals, also add this in .bashrc
export TERM=xterm

# Use CTRL+X and Y to exit and save changes
# Load bashrc the updated bashrc in the system
source ~/.bashrc

# go to http://RPiIP:DOCKGEPORT
# Create your admin account

# Fix constant errors because 64 bits it too new and packages complain about architecture
# These error messages and "expected kernel" comes from the needrestart package, this package has several modules
# The processor microcode module only supports AMD and Intel but not aarch64 (64bits ARM)
# You can remove it entirely with:
sudo apt-get purge needrestart

# Or you can change this value:
sudo sed -i 's/#\$nrconf{ucodehints} = 0;/$nrconf{ucodehints} = 0;/' /etc/needrestart/needrestart.conf

# Disable onboard Wifi
sudo nano /boot/config.txt

# Add this line
dtoverlay=disable-wifi

# Install Pi-Hole and use google or cloudflare DNS for now
curl -sSL https://install.pi-hole.net | bash

# Reset PiHole Password
pihole -a -p

# Install Unbound
sudo apt-get install unbound -y

# Paste the default configuration found in https://docs.pi-hole.net/guides/dns/unbound/
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
