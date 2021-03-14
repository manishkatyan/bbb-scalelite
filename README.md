# bbb-scalelite
An easy-to-follow step-by-step process to install Scalelite load balancer for BigBlueButton

To learn more about Scalelite, check-out [the official repository](https://github.com/blindsidenetworks/scalelite). 

## Minimum Server Requirements

For the Scalelite Server, the minimum recommended server requirements are:

- 4 CPU Cores
- 8 GB Memory

For the external Redis Cache, the minimum recommended server requirements are:

- 2 CPU Cores
- 0.5GB Memory
- Persistence must be enabled

## Configure your Front-End to use Scalelite

To switch your Front-End application to use Scalelite instead of a single BigBlueButton server, there are 2 changes that need to be made

- BigBlueButton server url should be set to the url of your Scalelite deployment http(s)://<scalelite-hostname>/bigbluebutton/api/
- BigBlueButton shared secret should be set to the LOADBALANCER_SECRET value that you set in /etc/default/scalelite

## Prepare the environment
```sh
sudo -i
apt-get update 
apt-get dist-upgrade
```

### Adding swap memory
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
```
### Installing Docker
```sh
apt-get update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg    

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
apt-get update

apt-get install docker-ce docker-ce-cli containerd.io
```

### Installing Docker Compose
```sh
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

## Installation 

### Fetching the scripts
```sh
git clone https://github.com/jfederico/scalelite-run
cd scalelite-run
```

### Initializing environment variables
```sh
# Create a new .env file based on the dotenv file included.
cp dotenv .env
```
Most required variables are pre-set by default, the ones that must be set before starting are:
- SECRET_KEY_BASE= 
- LOADBALANCER_SECRET= 
- URL_HOST= 
- NGINX_SSL=

Obtain the value for SECRET_KEY_BASE with:
```sh
openssl rand -hex 64
```

You should see something like this:
a7441a3548b9890a8f12b385854743f3101fd7fac9353f689fc4fa4f2df6cdcd1f58 bdf6a02ca0d35a611b9063151d70986bad8123a73244abb2a11763847a45
     
Obtain the value for LOADBALANCER_SECRET with
```sh
openssl rand -hex 24
```

You should see something like this:
c2d3a8e27844d56060436f3129acd945d7531fe77e661716

Set the hostname on URL_HOST (E.g. scalelite.example.com) When using a SSL certificate set NGINX_SSL to true

Your final .env file should look like this:
```sh
SECRET_KEY_BASE=cddacbc1474df5b899a1fb3c614551f301218fa58d9e7804bd05321437a4f1c96fe5438a6ea610fe68204a109fcbc90a1f78b97d2dbf6ef391f845c6ba454043 

LOADBALANCER_SECRET=c2d3a8e27844d56060436f3129acd945d7531fe77e661716 

URL_HOST=scalelite.example.com

NGINX_SSL=true
```

For using a SSL certificate signed by Let’s Encrypt, generate the certificates.
```sh
./init-letsencrypt.sh
```

Start the services.
```sh
docker-compose up -d
```

Now, the scalelite server is running, but it is not quite yet ready. The database must be initialized.
```sh
docker exec -i scalelite-api bundle exec rake db:setup
```

## Configuration

It is important to mention that Scalelite doesn’t have a UI for administration. Instead it comes with a set of back-end scripts (rake tasks) that must be run using the Command Line for configuring and managing the BigBlueButton servers that will be used.

You can check the server status.
```sh
docker exec -i scalelite-api bundle exec rake status
```

You should notice that there are no servers configured. That can also be verified with pulling the list of servers.
```sh
docker exec -i scalelite-api bundle exec rake servers
```

But you can add some BigBlueButton servers.
```sh
docker exec -i scalelite-api bundle exec rake servers:add[https://bbb1.example.com/bigbluebutton/api/,bbb-secret]
```

That should give you an ID for each server added.
```sh
OK
id: 27243e91–35a3–42ee-80a7-bd5980b0728f
```

Use the server ID for enabling the server.
```sh
docker exec -i scalelite-api bundle exec rake servers:enable[27243e91–35a3–42ee-80a7-bd5980b0728f]
```

Servers can be also be disabled.
```sh
docker exec -i scalelite-api bundle exec rake servers:disable[27243e91–35a3–42ee-80a7-bd5980b0728f]
```

Pulled out of the pool.
```sh
docker exec -i scalelite-api bundle exec rake servers:panic[27243e91–35a3–42ee-80a7-bd5980b0728f]
```

Or removed.
```sh
docker exec -i scalelite-api bundle exec rake servers:remove[27243e91–35a3–42ee-80a7-bd5980b0728f]
```

For more detailed information regarding server management, read the README file in [the official git repository](https://github.com/blindsidenetworks/scalelite).

## Using BigBlueButton servers through Scalelite

Your BigBlueButton servers are now ready to be used. You can use Scalelite with any external application (such as Moodle or Wordpress) by setting its hostname as the BigBlueButton URL and the secret generated (LOADBALANCER_SECRET) during the installation as the BigBlueButton Secret.

```sh
URL: https://scalelite.example.com/bigbluebutton/api/
Secret: c2d3a8e27844d56060436f3129acd945d7531fe77e661716
```

## Handling recordings
We use NFS for recordings. 

### (1) On Scalelite Server [NFS Host]
```sh
sudo apt update
sudo apt install nfs-kernel-server
```

#### Setup Firewall
Allow ports 22, 80 and 443 for normal Scalelite functioning. 

Then execute the following to allow connection from BBB server (BBB_SERVER_IP) for NFS:
```sh
$ ufw allow from BBB_SERVER_IP to any port nfs
```

Add following in /etc/exports:
```sh
/mnt/scalelite-recordings BBB_SERVER_IP(rw,sync,no_root_squash) BBB_SECOND_SERVER_IP(rw,sync,no_root_squash)
```

Then execute the following to start NFS server:
```sh
exportfs -r
sudo systemctl start nfs-kernel-server.service 
```

### (2) On BBB Server [NFS Client]

Create scalellite directory as detailed below
```sh
sudo cd /var/bigbluebutton/recording
sudo mkdir scalelite
sudo chown bigbluebutton:bigbluebutton scalelite
```

On each BigBlueButton server, install the following files to the listed paths:

- [scalelite_post_publish.rb](https://raw.githubusercontent.com/blindsidenetworks/scalelite/master/bigbluebutton/scalelite_post_publish.rb): install to the directory /usr/local/bigbluebutton/core/scripts/post_publish
- [scalelite.yml](https://raw.githubusercontent.com/blindsidenetworks/scalelite/master/bigbluebutton/scalelite.yml): install to the directory /usr/local/bigbluebutton/core/scripts
- [scalelite_batch_import.sh](https://raw.githubusercontent.com/blindsidenetworks/scalelite/master/bigbluebutton/scalelite_batch_import.sh): install to the directory /usr/local/bigbluebutton/core/scripts. 

You need to update `scalelite.yml`
```sh
vi scalelite.yml

work_dir: /var/bigbluebutton/recording/scalelite
spool_dir: /mnt/scalelite-recordings/var/bigbluebutton/spool
```

In case you need to manually transfer recordings from BigBlueButton server to Scalelite, execute the following:
```sh
chmod +x scalelite_batch_import.sh
./scalelite_batch_import.sh
```
Run these commands to create the group and add the bigbluebutton user to the group
```sh
# Create a new group with GID 2000
sudo groupadd -g 2000 scalelite-spool

# Add the bigbluebutton user to the group
sudo usermod -a -G scalelite-spool bigbluebutton

# In case you face any permission issue, do the following [reference](https://groups.google.com/g/bigbluebutton-setup/c/LT1IFWG9lQE/m/bpDpaG1UAgAJ)
sudo usermod -a -G bigbluebutton bigbluebutton
```

Now verify that there is no permission issue when bigbluebutton user creates a file, the file should be available on Scalelite server:
```sh
sudo -n -u bigbluebutton touch /mnt/scalelite-recordings/var/bigbluebutton/spool/toto.txt
```

Install NFS client.
```sh
sudo apt update
sudo apt install nfs-common
mkdir /mnt/scalelite-recordings
mount SCALELITE_SERVER_IP:/mnt/scalelite-recordings /mnt/scalelite-recordings
```

Now execute the following command to check recordings folders are correctly mounted on BBB server:
```sh
df -h
```

Add the following entry to your `/etc/fstab` file in each BBB server:
```sh
SCALELITE_SERVER_IP:/mnt/scalelite-recordings /mnt/scalelite-recordings nfs defaults 0 0
```
Restart BigBlueButton server
```sh
sudo bbb-conf --restart
sudo bbb-conf --check
```

## How to test Scalelite

- Visit [API-MATE](https://mconf.github.io/api-mate/)
- Enter Server: http://<your-scalelite-domain>/bigbluebutton/api 
- Enter Secret: LOADBALANCER_SECRET
- Now you can create and join meetings, which will happen to one of the BBB servers that you have added to your Scalelite server

### Credits

A large part of above installation steps is thanks to [Jesus Federico](https://github.com/jfederico)
