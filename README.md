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

Buy a domain and set it to forward requests to your public IP. Address of your public IP can be detected by 
