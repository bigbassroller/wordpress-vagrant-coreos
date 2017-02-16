
What is Cloud Native WordPress?

Well, Cloud Native is the new hot trendy buzz word in devOps, and WordPress is the software that is used by the most amount of people. So I put the two together. What is Cloud Native? Cloud Native is apps that are born inside a cloud infrastructure. Cloud infrastructure's strongest ally is the Linux Container, which offer easy transport and scalability inside the cloud.

The objective is to make WordPress a backend microservice for a javaScript app by ingesting WordPress data via the JSON-REST-API. Encapsulating WordPress into containers in our local environment will give us the opportunity to not only build our app in a easily distributed manner, but also allow us to architect our cloud based microservices.



What are Linux Containers?

Linux Containers have been around for awhile, but have been recently standardized through the Open Container Initiative (OCI). The most popular linux container runtime is Docker, which I am sure you've seen the cutesy Docker whale: Docker Logo

We all like warm fuzzy animals from the ocean, so in this tutorial I'll be using Docker for containers (And its the most documented and hopefully the most stable).



What is CoreOS?

CoreOS is a linux kernel like Ubuntu, but with bare minimal components to run standardized Linux Containers. CoreOS has many benefits for running cloud native apps, like easy cluster configuration and rolling OS updates with no downtime. We will be running Docker containers inside CoreOS in this tutorial.



What is Vagrant?

Vagrant is a virtual environment manager. We will use it like getting a droplet from Digital Ocean or an instance at AWS. We will be creating a VM environment with Vagrant and install the CoreOS Linux kernal inside a partition created by Vagrant. This way we have an isolated and standardized dev environment that can be replicated on different cloud providers, like Digital Ocean or AWS. No more WOMM (Works On My Machine).



What is VirtualBox?

Virtual Box is a Hypervisor, or in layman terms, a virtual hardware environment. It is needed to run Vagrant on MacOS and Windows.





How to setup local Cloud Native WordPress Environment:



Step 1: Install Vagrant and Virtualbox

First you'll need VirtualBox. Go to https://www.virtualbox.org/wiki/Downloads
and choose which operating system you are on and follow the instructions. It should only take a couple of clicks.

Next install Vagrant by going to https://www.vagrantup.com/downloads.html, choose your operating system and follow the instructions. This should only take a couple of clicks as well.

Now, to confirm successful installation. go into your terminal window and type:

vagrant -v


Step 2: Install CoreOS

Now we are ready to install CoreOS inside a VM created by Vagrant.

Make a directory for your project and change directory into it. In your terminal type:

mkdir cloudnative-wordpress; cd cloudnative-wordpress


Now we are going to clone the CoreOS Vagrant configuration repo from github:

git clone https://github.com/coreos/coreos-vagrant .
Next there are some minor configurations we have to do. In your favorite text editor, open up the folder that the git clone created. You should see a list of files like this:

├── CONTRIBUTING.md
├── DCO
├── LICENSE
├── MAINTAINERS
├── NOTICE
├── README.md
├── Vagrantfile
├── config.rb.sample
└── user-data.sample
config.rb:

First, make a copy of config.rb.sample and rename it config.rb. Open it up add make these changes:

$share_home=false
to be true:

$share_home=true
Next, tell CoreOS which folder from your local computer to share with the VM. First, be sure to get the absolute path of the directory inside the terminal of where you installed your apps VM by typing this:

pwd
Use that result in place of "/Users/YOURUSERNAME/Sites/cloudnative-wordpress" in this command:

$shared_folders = {'/Users/YOURUSERNAME/Sites/cloudnative-wordpress/site' => '/srv/www/space-rocket/public_html/wordpress'}
Vagrantfile:

Next, we are going to set our IP address and tell vagrant sync our local computers folder to a folder inside the CoreOS VM. Open the Vagrantfile and comment out:

ip = "172.17.8.#{i+100}"
config.vm.network :private_network, ip: ip
and add this:

config.vm.network "private_network", ip: "172.17.8.150"
Then add:

config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']
We also have to create a folder for WordPress locally. Do that by typing:

mkdir site


Step 3: Login into CoreOS VM and run Docker

Now type:

vagrant up
It should ask you for your password for the etc/hosts file to be modified.

If you get error saying something about NFS error, run these commands:

sudo rm /etc/exports
sudo touch /etc/exports
vagrant halt
vagrant up --provision
There is also a permission issue with synching files to you local computer. To solve,  open /etc/exports on your computer and change

-mapall=<your-uid>:<your-gid>
to:

-maproot=0
*See this issue regarding this. If you know of better fix, please let me know in the comments below.

Now, restart 'nfsd' by running this command:

sudo nfsd restart
If all goes well, login into your vm by typing:

vagrant ssh
Once inside the VM, you'll need to create a place for mySQL to run. Type:

mkdir -p /home/core/env/mariadb/data
Before we use the Docker CLI that comes with CoreOS to download and configure mySQL/Mariadb, run this command to rm running containers:

docker rm $(docker ps -a -q) -f
Now the Docker command to install MariaDB:

docker run -d -v /home/core/env/mariadb/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -p 3306:3306 --name mariadb mariadb:latest
Now to install WordPress with this command:

docker run -d --name wordpress --link mariadb:mysql -p 8080:80 -v /home/core/share/site:/srv/www/space-rocket/public_html/wordpress wordpress:4.7.0-php7.0-apache
You should now have WordPress running at:

http://172.17.8.150:8080/
There is one last step to enable media file uploads. Log into the docker container by typing:

docker exec -it wordpress /bin/bash
Once in the shell for the wordpress container, run this command:

chown www-data:www-data -R /srv/www/space-rocket/public_html/wordpress/wp-content
Here is a table of of the flags that we just used:

Flag	Type	Action
-d, --detach	n/a	Run container in background and print container ID
-e, --env	value	Set environment variables (default [])
-i, --interactive	N/A	Keep STDIN open even if not attached
-t, --tty	N/A	Allocate a pseudo-TTY (terminal driver)
--name	string	Assign a name to the container
--link	value	Add link to another container (default [])
-p, --publish	value	Publish a container's port(s) to the host (default [])
v, --volume	value	Bind mount a volume (default [])
This is a quick way to create a CloudNative WordPress on your Mac or Windows machine. Next, we'll have the Docker CLI commands broken down into a Dockerfile. We'll save that for our next blog article!