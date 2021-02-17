# scalelite-run
A simple deployment as for production using docker-compose

## Prepare the environment Updating the VM
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
apt install apt-transport-https ca-certificates curl software- properties-common
apt install apt-transport-https ca-certificates curl software- properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt- key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
apt update
apt install docker-ce
```

### Installing Docker Compose
```sh
curl -L https://github.com/docker/compose/releases/download/1.21.2/docker- compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

## Installation 

### Fetching the scripts
```sh
git clone https://github.com/jfederico/scalelite-run cd scalelite-run
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
SECRET_KEY_BASE=a7441a3548b9890a8f12b385854743f3101fd7fac9353f689fc4 fa4f2df6cdcd1f58bdf6a02ca0d35a611b9063151d70986bad8123a73244abb2a117 63847a45 LOADBALANCER_SECRET=c2d3a8e27844d56060436f3129acd945d7531fe77e661716 URL_HOST=scalelite.example.com
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

