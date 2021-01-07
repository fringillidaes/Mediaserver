# Mediaserver
This is a sort of do it yourself guide to migrating your cloudbox setup to an all in one docker-compose setup. This is mainly targeted at people who have found their suite of apps that they need and have gained Linux and docker experience using it. This is targeted at a Usenet setup. 

# Prerequisites
You will need: 
  - Some dedicated server running ubuntu 18.04. (Could technically run any other OS but directions may not translate over.)
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
PART /boot  ext4     512M
PART lvm    vg0       all
LV vg0   swap   swap     swap          16G
LV vg0   opt    /opt     btrfs      250G
LV vg0   root   /        ext4         all
IMAGE /root/.oldroot/nfs/install/../images/Ubuntu-1804-bionic-64-minimal.tar.gz
```

# Make your paths
Cloudbox has alot of paths assigned to its service files and docker containers. We can translate this over to this fresh installation by doing the following,
```bash
sudo mkdir /mnt
sudo chown -R $user:$user /mnt
sudo mkdir /mnt/unionfs
sudo chown -R $user:$user /mnt/unionfs
sudo mkdir /mnt/local
sudo chown -R $user:$user /mnt/local
sudo mkdir /mnt/local/transcode
sudo chown -R $user:$user /mnt/local/transcode
sudo mkdir /mnt/local/transcode/jellyfin
sudo chown -R $user:$user /mnt/local/transcode/jellyfin
sudo mkdir /mnt/local/downloads
sudo chown -R $user:$user /mnt/local/downloads
sudo mkdir /mnt/local/downloads/nzbs
sudo chown -R $user:$user /mnt/local/downloads/nzbs
sudo mkdir /mnt/local/Media
sudo chown -R $user:$user /mnt/local/Media
sudo mkdir /mnt/local/Media/Movies
sudo chown -R $user:$user /mnt/local/Media/Movies
sudo mkdir /mnt/local/Media/Music
sudo chown -R $user:$user /mnt/local/Media/Music
sudo mkdir /mnt/local/Media/TV
sudo chown -R $user:$user /mnt/local/Media/TV
sudo mkdir /mnt/remote
sudo chown -R $user:$user /mnt/remote 
``` 

# Dependency time
Let's update the system and install some prerequisite packages. (these might be reduced? idk most are from the cloudbox dependency list.)
```
sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get install rclone mergerfs nano zip unzip curl sqlite3 tree lsof man-db git pwgen rsync logrotate htop iotop nload fail2ban ufw ncdu mc dnsutils screen tmux jq moreutils unrar nodejs iperf3
```

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
# i needed sudo when working in my /opt directory due to the user not having permissions. my fucked up way to undo this was (but beware do not do this on either nginx folders.)
sudo find . -maxdepth 1 -exec sudo chown -R $user:user {} \;
```

