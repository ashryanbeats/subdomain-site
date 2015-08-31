# Meaniscule Digital Ocean Deployer

This script aims to help you deploy your [Meaniscule](https://github.com/meaniscule/meaniscule) site to Digital Ocean with ease and win.

## Contents
- [Requirements](https://github.com/meaniscule/digital-ocean-deployer/blob/master/README.md#requirements)
- [Assumptions](https://github.com/meaniscule/digital-ocean-deployer/blob/master/README.md#assumptions)
- [Installation and execution](https://github.com/meaniscule/digital-ocean-deployer/blob/master/README.md#installation-and-execution)
- [User input](https://github.com/meaniscule/digital-ocean-deployer/blob/master/README.md#user-input)
- [Results](https://github.com/meaniscule/digital-ocean-deployer/blob/master/README.md#results)
  - The nginx server block
  - The tree
  - The post-receive hook
  - The local remote settings

## Requirements
Local
- A *local* git repo containing a [Meaniscule](https://github.com/meaniscule/meaniscule) app

Server
- Ubuntu 14.04 or higher*
- `sudo` privileges
- [nginx](http://nginx.org/en/)**
- [pm2](https://github.com/Unitech/pm2)**

_*Might work on older versions of Ubuntu, but it hasn't been tested_

_**The script will check to see if you have nginx and pm2 installed, and if you don't, it will optionally offer to install them for you._

## Assumptions
The script assumes that:
- you are running the script from somewhere within `/home/user` (not `root`)
- you want to deploy the site within your `/home/user` directory
- your `nginx` file is titled `default` and exists at `/root/etc/nginx/sites-available`
 
## Installation and execution
Install the script on your server. One way to do this is to `git clone` it into your user directory, for example in `/home/user/bin`.

To make the script executable, `cd` down in to the directory containing the script and enter:
```
chmod +x do-deploy.sh
```

Now you can run the script by typing in:
```
sudo ./do-deploy.sh
```
Because the script modifies the nginx file, `sudo` will be required in most cases.

## User input
The script will ask the user for:
- subdomain
- domain
- port
- app directory name

For example:
```
dog
pet.com
4545
dog-site
```
This makes a new nginx server with a URL of `dog.pet.com` listening on port `4545` and sets up your file tree in `/home/user/dog-site`.

## Results
The script does the following:
- adds a new nginx server listening on the specified port and reloads nginx
- makes a new tree and initializes a `--bare` git repo 
- adds a `post-receive` hook for git
- tells you how to set this repo as a remote for your local repo

### The nginx server block
```
server {
  listen 80;
  server_name "dog.pet.com";
  location / {
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   Host      $http_host;
    proxy_pass         http://127.0.0.1:4545;
  }
}
```

### The tree
```
/home/ash/dog-site
├── live
└── repo
    └── site.git
        ├── branches
        ├── config
        ├── description
        ├── HEAD
        ├── hooks
        │   └── post-receive
        ├── info
        │   └── exclude
        ├── objects
        │   ├── info
        │   └── pack
        └── refs
            ├── heads
            └── tags
```

### The post-receive hook
Here is the default `post-receive` hook that this script will make (using our `dog-site` example):
```
#!/bin/sh
git --work-tree=/home/ash/dog-site/live --git-dir=/home/ash/dog-site/repo/site.git checkout -f
cd ~/dog-site/live
npm install
gulp build

pm2 describe dog-site
if [ \$? != 0 ]
then
        pm2 start server/start.js -i 0 --name dog-site
else
        pm2 restart dog-site
fi
```

So, after running `git push live master` from your local repo, the `post-receive` script will:
- configure your git work tree and repo
- install npm modules (if there are new ones to install)
- run a gulp build process then exit gulp
- either restart the pm2 process for the site or create a pm2 process for the site if one doesn't already exist

### The local remote settings
When the Digital Ocean Deployer script is finish running, it will give you these two lines, which are meant to be used in your local repo:
```
git remote add live ssh://ash@SERVER_IP_OR_DOMAIN/home/ash/dog-site/repo/site.git
git push live master
```

The first line sets your local repo's `live` remote to point to your server repo that the script just created for you. Enter this into your local repo right away.

The second line is what you will use any time you want to deploy to your server: `git push live master`.

*Note:* this `git push` has nothing to do with GitHub or any other remote `origin`. If you are open sourcing on GitHub, you can push to GitHub with `git push origin master` without ever affecting your deployed site. This way you can push to and pull from GitHub until you are ready to deploy. Then when you are ready, you can enter `git push live master` which will deploy your *local* repo directly to your server. 
