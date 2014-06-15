#如何升级Docker取决于你是如何安装Docker的

##使用 apt-get

<pre>
#Add the Docker repository key to your local keychain
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
#Add the Docker repository to your apt sources list.
sudo sh -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"
#update your sources list
sudo apt-get update
#install the latest
sudo apt-get install lxc-docker
</pre>

##手工安装

<pre>
# kill the running docker daemon
killall docker
# get the latest binary
wget http://get.docker.io/builds/Linux/x86_64/docker-latest -O docker
# make it executable
chmod +x docker
# start the new version -d是指以守护进程的方式执行， & 指在后台执行
sudo ./docker -d &
</pre>