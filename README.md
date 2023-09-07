# This is a setup for domjudge using docker

#### Note: All the information is coming from [domjudge.org](https://www.domjudge.org/about)

#### All tests running on Windows 11 WSL2 Ubuntu 22.04 LTS

# Running Domjudge on local machine

## Install docker 

```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
sudo apt update
sudo apt install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

## Install domjudge docker image 

```
sudo docker pull domjudge/domserver
sudo docker pull domjudge/judgehost
```

Before starting the containers, make sure you have a MySQL / MariaDB database somewhere:

```
sudo docker pull mariadb
```

## MariaDB container

The easiest way to get one up and running is to use the MariaDB Docker container:

```
sudo docker run -it --name dj-mariadb -e MYSQL_ROOT_PASSWORD=rootpw -e MYSQL_USER=domjudge -e MYSQL_PASSWORD=djpw -e MYSQL_DATABASE=domjudge -p 13306:3306 mariadb --max-connections=1000
```

![MariaDB](https://github.com/wincle626/domjudgeDockerSetup/blob/main/pics/MariaDB.png)

This will start a MariaDB container, set the root password to rootpw, create a MySQL user named domjudge with password djpw and create an empty database named domjudge. It will also expose the server on port 13306 on your local machine, so you can use your favorite MySQL GUI to connect to it. If you want to save the MySQL data after removing the container, please read the [MariaDB](https://hub.docker.com/_/mariadb) Docker Hub page for more information.

## DOMserver container

If you are on Linux make sure you have cgroups enabled. See the [DOMjudge documentation about setting up a judgehost](https://www.domjudge.org/docs/manual/main/install-judgehost.html#linux-control-groups) for information about how to do this. Docker on Windows and macOS actually use a small Linux VM which already has these options set.

Run the domserver using the following command:

```
sudo docker run --link dj-mariadb:mariadb -it -e MYSQL_HOST=mariadb -e MYSQL_USER=domjudge -e MYSQL_DATABASE=domjudge -e MYSQL_PASSWORD=djpw -e MYSQL_ROOT_PASSWORD=rootpw -p 12345:80 --name domserver domjudge/domserver:latest
```

![JudgeServer](https://github.com/wincle626/domjudgeDockerSetup/blob/main/pics/JudgeServer.png)

If a specific DOMjudge version is required instead of the 'latest', replace 'latest' with the DOMjudge version (e.g. '5.3.0').

The above command will start the container and set up the database. It will then start nginx and PHP-FPM using supervisord.

The initial passwords for the admin and judgehost users should be printed when starting the domserver, but if not, you can use the following commands to retrieve them:

```
sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret
sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/restapi.secret
```

You can now access the web interface on [http://localhost:12345/](http://localhost:12345/) and log in as admin.

If you lose access to the admin user, see the [DOMjudge documentation on resetting the password](https://www.domjudge.org/docs/manual/main/config-basic.html#resetting-the-password-for-a-user).

Make a note of the password for the judgehost user, it will be used when the judgehost container is configured. The password can be changed from the web interface by editing the judgehost user.

## Judgehost container

To run a single judgehost, run the following command:

```
sudo docker run -it --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro --name judgehost-0 --link domserver:domserver --hostname judgedaemon-0 -e JUDGEDAEMON_PASSWORD=jwcrulWzmX6PWfdKKmi06c6RntOd5vFS -e DAEMON_ID=0 domjudge/judgehost:latest
```

![JudgeHost](https://github.com/wincle626/domjudgeDockerSetup/blob/main/pics/JudgeHost.png)

Again, replace latest with a specific version if desired. Make sure the version matches the version of the domserver.

This will start up a judgehost that is locked to the first CPU core of your machine.

If the judgedaemon stops for whatever reason, you should be able to see the error it produced in the shell where you started the judgehost. If you want to restart the judgehost, run 'docker start judgehost-0', where 'judgehost-0' is the value you passed to '--name' in the 'docker run' command.

##### PS: make sure set the JUDGEDAEMON_PASSWORD parameter using the displayed judgehost password on the judgeserver startup terminal. 

## Docker compose

Or, one could use docker-compose command to read yml files to setup all the database, judgeserver, judgehost at once:

```
sudo docker-compose -f docker_compose.yml up -d
```

## Test at local using the hello world problem

![Helloworld](https://github.com/wincle626/domjudgeDockerSetup/blob/main/pics/hellworld.png)


# !!!! Now, the most excited remote judgehost setup !!!!

## Say, there are two PC A and B, with IP address 192.168.137.2 (A) and 192.168.137.1 (B), and A is going to be the domserver while B is going to be the judgehost. 

## Firstly, install mariadb and domserver on PC A. 

The key thing is, once the domserver container is running, execute the bash on domserver container:

```
sudo docker exec -it domserver /bin/bash
```

In the container, modified the configuration of web API URL in "restapi.secret" from its default to "http://192.168.137.2:12345/api":

```
apt-get update
apt-get install vim
vim /opt/domjudge/domserver/etc/restapi.secret
```

![domserverwebapi](https://github.com/wincle626/domjudgeDockerSetup/blob/main/pics/domserverwebapi.png)

This will allow the judgehost to access remote from the same networ range in 192.168.137.*.

## Secondly, install judgehost on PC B 

Instead of create judgehost container at local, use the modified web API URL to create the container associate to the remote domserver:

```
sudo docker run -it --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro --name judgehost-1 --link domserver:domserver --hostname judgedaemon-1 -e DAEMON_ID=0 -e DOMSERVER_BASEURL='http://192.168.137.2:12345/' -e JUDGEDAEMON_USERNAME='judgehost' -e JUDGEDAEMON_PASSWORD='uCp9VCj5fLUG4CEKtPjDQaVnHF9A1V08' domjudge/judgehost:latest
```

This will create a judgehost container at PC B that could be used for calling from PC A. 

![judgehostremote](https://github.com/wincle626/domjudgeDockerSetup/blob/main/pics/judgehostremote.png)

Now we could submit throug the "demo" user and as shown it can successfully found PC B as judgehost to do the judge of submision. 

![demojudge](https://github.com/wincle626/domjudgeDockerSetup/blob/main/pics/demojudge.png)


