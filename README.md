# Run a Self Hosted Cloud with Nextcloud

This repository outlines how to prepare your Linux System (preferable a Raspberry Pi) to run a Self Hosted Cloud

## Prerequisites

### Install Docker

Follow the instruction in this [link](https://pimylifeup.com/raspberry-pi-docker/)

### Adding additional storage and setting up RAID

Follow the instructions in this [link](https://www.youtube.com/watch?v=Y_l1BCCqZSQ)

Once setup, ensure that the the folder pointing to your RAID array is setup in `/etc/fstab`

In case you decide to use a Quad SATA Hat, follow the instructions by the manufacturer.

Chances are that entry in `/etc/fstab` won't work as there might be a service that first needs to run to detect your Quad SATA Hat before drives can be detected and the file system mounted.
You will need to create automount files under `/etc/systemd/system`

I use a Quad SATA Hat called `Rock Pi` which has a service which first needs to start up before I can mount the filesystem
Here are the sample files I use. Paste them inside `/etc/systemd/system` Note that my RAID array is setup as a filesystem under `/portainer`

- `portainer.mount` file - 
   ```bash
   [Unit]
   Description=Quad Sata
   After=rockpi-sata.service
   Requires=rockpi-sata.service

   [Mount]
   What=/dev/disk/by-uuid/<UUID of RAID Array>
   Where=/portainer
   Type=ext4
   Options=defaults,nofail
   LazyUnmount=True

   [Install]
   WantedBy=multi-user.target
   ```

- `portainer.automount` file -
   ```bash
   [Automount]
   Where=/portainer

   [Install]
   WantedBy=multi-user.target
   ```
After pasting them, run the command - `sudo systemctl enable portainer.mount`. Now, your filesystem is mounted after your SATA Hat service runs after startup automatically

### Buy a domain

Buy a domain and set it to forward requests to your public IP. Address of your public IP can be detected by running `curl checkip.amazonaws.com`

### Setup Port Forwarding

This is so that Nextcloud can respond to requests from the Open Internet

Install MiniUPNPc - `sudo apt-get install -y miniupnpc`

Then, you can use the script below to setup automatic port forwarding

```bash
#!/usr/bin/env bash

###  Create .update-port-forward.log file of the last run for debug
parent_path="$(dirname "${BASH_SOURCE[0]}")"
FILE=${parent_path}/update-port-forward.log
if ! [ -x "$FILE" ]; then
  touch "$FILE"
fi

LOG_FILE=${parent_path}'/update-port-forward.log'
tail -1000 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"

### Function to log messages
log() {
    echo "$(date "+%Y-%m-%d %H:%M:%S") - $1" >> "$LOG_FILE"
}

### Function to get the current IP address
get_ip() {
    # Replace with your preferred method of obtaining IP, e.g., using ifconfig or ip addr
    local_ip=$(upnpc -s | grep "Local LAN ip address" | awk '{print $6}')
    echo "$local_ip"
}

### Function to remove existing port forwards
remove_port_forwards() {
    upnpc -l | grep "Raspberry Pi" | awk '{print $3}' | while read port; do
        external_port=$(echo "$port" | cut -d'-' -f1)
        upnpc -d "$external_port" TCP >/dev/null 2>&1
    done
}

# Function to check if port forwards exist
ports_exist() {
    local ip="$1"
    local ports=(80 443)  # Adjust ports as needed

    for port in "${ports[@]}"; do
        upnpc -l | grep -q "$ip:$port" && return 0  # True if port exists
    done

    return 1  # False if none of the ports exist
}

# Function to add port forwards only if they don't exist
add_port_forwards_if_needed() {
    local ip=$(get_ip)
    if ! ports_exist "$ip"; then
        log "Change in IP Detected. Changing port forwards..."
        remove_port_forwards # This network only sets port forwarding to my Raspberry Pi.
        local ports=(80 443)
        for port in "${ports[@]}"; do  # Adjust ports as needed
            description="$port on Raspberry Pi"
            protocol="TCP"
            log "Running upnpc command: upnpc -e '$description' -a $ip $port $port $protocol"
            upnpc_output=$(upnpc -e \'"$description"\' -a "$ip" "$port" "$port" "$protocol" 2>&1)
            log "upnpc output: $upnpc_output"
        done
    fi
}

### Run script once
current_ip=$(get_ip)
log "Current IP: $current_ip"
add_port_forwards_if_needed
```

Save it as `/usr/local/bin/update-port-forward`. And then set it up as a Cron Job by running command `sudo crontab -e` and pasting the following at the end

```
*/2 * * * * sudo /usr/local/bin/update-port-forward
```

This runs it every 2 minutes.

### Updating DNS entry

When you bought your domain, you will have to make an entry to the DNS pointing to your public IP address. This way, when someone types your domain, they'll be forwarded to your public IP address.
However, your public IP address is not static. 

To ensure that this entry is automatically updated if your public IP changes, follow the instructions in this [repo](https://github.com/fire1ce/DDNS-Cloudflare-Bash) if you use Cloudflare for your DNS management.


## Install Nextcloud

- Prepare your necessary folders in your mounted filesystem. My mounted filesystem is called `/portainer`.
- Create a docker-compose file anywhere you want and replace with the passwords you want.

```yaml
version: "3"
volumes:
  nextcloud-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /portainer/nextcloud
  mysql-db:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /portainer/mysql
  proxymanager-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /portainer/proxymanager/data
  proxymanager-ssl:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /portainer/proxymanager/ssl

networks:
  frontend:
  backend:

services:
  nextcloud-app:
    image: nextcloud
    container_name: nextcloud-app
    restart: always
    volumes:
      - nextcloud-data:/var/www/html
    environment:
      - MYSQL_PASSWORD=<redacted>
      - MYSQL_DATABASE=common
      - MYSQL_USER=prateek-nextcloud
      - MYSQL_HOST=mysql-db
      - PHP_UPLOAD_LIMIT=10G
      - PHP_MEMORY_LIMIT=3G
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_HOST_PASSWORD=<redacted>
    depends_on:
      - mysql-db
      - redis
    networks:
      - frontend
      - backend

  mysql-db:
    image: mysql
    container_name: mysql-db
    restart: always
    # ports:
    #  - "3306:3306"
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - mysql-db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=<redacted>
      - MYSQL_PASSWORD=<redacted>
      - MYSQL_DATABASE=common
      - MYSQL_USER=prateek-nextcloud
    networks:
      - backend

  proxymanager:
    image: jc21/nginx-proxy-manager:2.10.2
    container_name: proxymanager
    restart: always
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - proxymanager-data:/data
      - proxymanager-ssl:/etc/letsencrypt
    depends_on:
      - nextcloud-app
    networks:
      - frontend

  redis:
   image: redis
   container_name: redis
   restart: always
   command: redis-server --requirepass <redacted>
   networks:
     - backend

```

- Go to your Raspberry Pis' IP with the port 81 to access Nginx Reverse Proxy and setup a proxy host for your domain to http://nextcloud-app:80
(Default credentials are `admin@example.com | changeme`)
- Go to your domain and continue setup of Nextcloud :)

