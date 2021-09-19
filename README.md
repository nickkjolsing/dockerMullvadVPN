# mullvadVPN Docker container
![Image of Docker](https://d1q6f0aelx0por.cloudfront.net/product-logos/644d2f15-c5db-4731-a353-ace6235841fa-registry.png) ![Image of Mullvad](https://www.aeres-evaluation.fr/wp-content/uploads/2019/07/mullvad-vpn-logo.png)
### Mullvad VPN container for docker. Example on how to setup Transmission with container at the bottom of the page.  
#### [Docker container that this relys on](https://github.com/yacht7/docker-openvpn-client)

# 

#### Prerequisites
- Docker installed (I'm using 19.03.8 Desktop on macOS)
- [Mullvad](https://mullvad.net/) account (can be done with other providers, I completed with Mullvad) 

# 

###### Assuming environment is setup and you know drive mount locations. 
### Step 1: Getting Mullvad configuration .zip file
1. Login [to account on Mullvad.net](https://mullvad.net/en/account/#/)
2. Visit: [OpenVPN configuration file generator on their website](https://mullvad.net/en/account/#/openvpn-config/?platform=linux)
3. Select your favorite country and city within that country!
4. Under advanced settings: toggle ```UDP 53```
5. Download zip archive, unarchive it into a regular folder, and place within a directory accessable by your Docker containers
6. Return to ```My account``` and click [Manage ports and WireGuard keys](https://mullvad.net/en/account/#/ports)
7. Next to a public key click the green ```+```. This adds a port that will be used for configuring OpenVPN. REMEMBER THIS NUMBER FOR LATER :)

### Step 2: Setup ```docker-compose``` file
 
    ---
    version: "2"
    services:
      openvpn-client:
        image: yacht7/openvpn-client            # Image on Docker. Shoutout to yacht7
        container_name: openvpn-client
        cap_add:
            - NET_ADMIN                         # Needs to be here
        environment: 
            - KILL_SWITCH=on                         # Turns off internet access if the VPN connection drops
            - FORWARDED_PORTS=5794                   # NUMBER TO REMEMBER FROM BEFORE, READ STEP 7 under STEP 1 (THIS IS CONFUSING AS IM TYPING IT, BUT READ IT)
            - SUBNETS=192.168.0.0/24,192.168.1.0/24  # Allows for the service to be accessed through LAN
        devices:
            - /dev/net/tun                      
        volumes:
            - /Volumes/Luigi/docker/mullvadVPN/config/mullvad_config_linux_ch_zrh:/data/vpn   
            # File unzipped before from Mullvad, it's location. Make sure to keep the ":/data/vpn" part at the end
        ports:
            - 5665:5665                         # Opening port for to access hypothetical Transmission container that would be routing through this VPN
            - 1500:1500                         # Opening port for other application routing through VPN
        restart: unless-stopped

### Step 3: Confirming VPN connection is active within container
1. ```cd``` into folder where the ```docker-compose.yml``` for this container is stored
2. Awaken the beast with ```docker-compose up```
3. Let's get [jiggy](https://youtu.be/3JcmQONgXJM?t=1) wit that sparkly new container:
    1. In a new terminal window, find docker container ID ```docker ps```
    2. Type ```docker exec -it <container ID from above> /bin/sh```
    3. Now that you're into the shell of your VPN container we're going to check it's public IP
    4. ```wget -qO- http://ipecho.net/plain | xargs echo``` will return your container's public IP
    5. Lookup this IP's information to see if it's the same country/city you seutp in your docker compose file. I'll let you find a site


Now go browse the internet from 🇨🇭Switzerland or somethin
# 
## Bonus section: Route other container's connection through this VPN
So you want to allow other containers to use this connection? Ok fine...
#### Add ```network_mode: container:openvpn-client``` to the container's compose file
#### Add ```ports:``` to the VPN's compose file

### Hypothetical Transmission example
1. Add ```network_mode: container:openvpn-client``` to docker compose file
2. Make sure to add ports to VPN docker compose file, like in my example above
    1. These ports will be the ports required by the application running in the container you're routing through the VPN. Ex: 5665 would be to access the Transmission Web UI in this situation

---
    version: "2.1"
    services:
      transmission:
        image: linuxserver/transmission
        container_name: transmission
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=America/Denver
        volumes:
          - <Config location>:/config
          - <Download location>:/downloads
          - <Watch location>:/watch
        network_mode: container:openvpn-client      # The addition to add to all containers that you want to route through VPN container
        restart: unless-stopped
