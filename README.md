# Mediaserver
This is a sort of do it yourself guide to migrating your cloudbox setup to an all in one docker-compose setup. This is mainly targeted at people who have found their suite of apps that they need and have gained Linux and docker experience using it. This is targeted at a Usenet setup while a plex torrent setup is in the works. 

# Prerequisites
You will need: 
  - Some dedicated server running Ubuntu 22.04 LTS. (Could technically run any other OS but directions may not translate over.)
  - SSH client.
  - Clean install.

# Hetzner ex42 setup (applies to some people)
If your using hetzner and have the same spec setup you can use the following install image. This also uses BTRFS for /opt. Do verify that your drives are listed as those on your setup. 
```
SWRAID 1
SWRAIDLEVEL 0
DRIVE1 /dev/nvme0n1
DRIVE2 /dev/nvme1n1
HOSTNAME mediaserver
PART /boot  ext4     2G
PART lvm    vg0       all
LV vg0   swap   swap     swap          16G
LV vg0   opt    /opt     btrfs      250G
LV vg0   root   /        ext4         all
IMAGE /root/.oldroot/nfs/install/../images/Ubuntu-1804-bionic-64-minimal.tar.gz
```

# Make your paths
Cloudbox has alot of paths assigned to its service files and docker containers. We can translate this over to this fresh installation by doing the following,
```bash
sudo mkdir /mnt/local
sudo mkdir /mnt/local/transcode
sudo mkdir /mnt/local/transcode/jellyfin
sudo mkdir /mnt/local/downloads
sudo mkdir /mnt/local/downloads/nzbs
sudo mkdir /mnt/local/Media
sudo mkdir /mnt/local/Media/Movies
sudo mkdir /mnt/local/Media/TV
sudo mkdir /mnt/unionfs
sudo mkdir /mnt/unionfs/transcode
sudo mkdir /mnt/unionfs/transcode/jellyfin
sudo mkdir /mnt/unionfs/downloads
sudo mkdir /mnt/unionfs/downloads/nzbs
sudo mkdir /mnt/unionfs/downloads/downloads-amd
sudo mkdir /mnt/unionfs/Media
sudo mkdir /mnt/unionfs/Media/Movies
sudo mkdir /mnt/unionfs/Media/TV
sudo mkdir /mnt/remote
sudo mkdir /mnt/remote/Media
sudo mkdir /mnt/remote/Media/Movies
sudo mkdir /mnt/remote/Media/TV
sudo chown -R $user:$user /mnt
``` 

# Basic System Setup.
We need to do a few things before we start restoring our server to the previous state. We need to recreate the same useraccount we had on cloudbox, if your name was tristen, your username needs to be tristen on this server for the sake of simplicity. To do this run the following (Obviously replace tristen with what your username is for both commands),
```
adduser tristen
# fill out the information from this ^
usermod -aG sudo tristen
```

# Dependency time
Let's update the system and install some prerequisite packages. These include rclone and docker. Adjust according to your setup if your deviating. (these might be reduced? idk most are from the cloudbox dependency list.)
```
# Upgrade from base install.
sudo apt-get update -y && sudo apt-get upgrade -y

# Installing Docker.
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Installing Rclone.
curl https://rclone.org/install.sh | sudo bash -s beta

# Installing Other Packages. 
sudo apt-get install mergerfs nano zip unzip sqlite3 tree lsof man-db git pwgen rsync logrotate htop iotop nload fail2ban ufw ncdu mc dnsutils screen tmux jq moreutils unrar nodejs iperf3
```
Now we need to change our DNS servers for Usenet since hetzners have some issues from the past. 
```
sudo nano /etc/netplan/01-netcfg.yaml
```
You are going to modify under `addresses:` to the following,
```
        - 1.1.1.1
        - 1.0.0.1
        - 8.8.8.8
        - 8.8.4.4
        - 2001:4860:4860::8888
        - 2001:4860:4860::8844
```
It should look something like this (note yours will obviously have different values.),
```
### Hetzner Online GmbH installimage
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s31f6:
      addresses:
        - 666.666.666.666/32
        - 2a69:2a69:2a69:2a69::6/64
      routes:
        - on-link: true
          to: 0.0.0.0/0
          via: 666.666.666.666
      gateway6: je666::66
      nameservers:
        addresses:
          - 1.1.1.1
          - 1.0.0.1
          - 8.8.8.8
          - 8.8.4.4
          - 2001:4860:4860::8888
          - 2001:4860:4860::8844
```
And finally we need to add our user to the docker group so we can access the docker daemon without root/sudo. 
```
sudo groupadd docker
sudo usermod -aG docker tristen
```
You will need to exit your shell and relogin to be able to manage docker. 


# Restore cloudbox backup
Now use rclone to download your backup to the /opt directory. 
I used the following commands to restore my backup (ONLY EXECUTE IN /opt IF YOU DECIDE TO USE THESE COMMANDS.) 
```
# finds all the tar files in the directory and untars.
find . -name "*.tar" -maxdepth 1 -exec sudo tar -xvf {} \;
```
```
# finds the previous tar files and deletes them from the disk.
sudo find . -name "*.tar" -maxdepth 1 -exec sudo rm -f {} \;
```
```
# i needed sudo when working in my /opt directory due to the user not having permissions.
sudo find . -maxdepth 1 -exec sudo chown -R $user:user {} \;
```

# Trim the backup
Now delete any directories you dont use anymore. You can delete both nginx and letsencrypt folders since we will be using caddy instead for the reverse proxy
```
rm -rf /opt/appname
```


