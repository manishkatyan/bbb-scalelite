# scalelite-run
A simple deployment as for production using docker-compose

## Prepare the environment Updating the VM
```sh
sudo -i
apt-get update 
apt-get dist-upgrade
```

## Adding swap memory
```sh
# Check if you already have sawp memory.
swapon --show
# If above shows empty, it means you don't have swap memory. Follow the steps below to add swap memory. 
fallocate -l 1G /swapfile
dd if=/dev/zero of=/swapfile bs=1024 count=1048576 chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Make the change permanent
# Edit /etc/fstab to add /swapfile swap swap defaults 0 0
```sh
