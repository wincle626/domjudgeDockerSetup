# This is a setup for domjudge using docker

#### Note: All the information is coming from [domjudge.org](https://www.domjudge.org/about)

## Install docker (Ubuntu 22.04 LTS)

```
sudo apt install docker.io
```

## Install domjudge docker image 
```
sudo docker pull domjudge/judgehost
```

Before starting the containers, make sure you have a MySQL / MariaDB database somewhere:

```
sudo docker pull mariadb
```

## MariaDB container

The easiest way to get one up and running is to use the MariaDB Docker container:

```
docker run -it --name dj-mariadb -e MYSQL_ROOT_PASSWORD=rootpw -e MYSQL_USER=domjudge -e MYSQL_PASSWORD=djpw -e MYSQL_DATABASE=domjudge -p 13306:3306 mariadb --max-connections=1000
```

This will start a MariaDB container, set the root password to rootpw, create a MySQL user named domjudge with password djpw and create an empty database named domjudge. It will also expose the server on port 13306 on your local machine, so you can use your favorite MySQL GUI to connect to it. If you want to save the MySQL data after removing the container, please read the [MariaDB](https://hub.docker.com/_/mariadb) Docker Hub page for more information.

## DOMserver container

If you are on Linux make sure you have cgroups enabled. See the [DOMjudge documentation about setting up a judgehost](https://www.domjudge.org/docs/manual/main/install-judgehost.html#linux-control-groups) for information about how to do this. Docker on Windows and macOS actually use a small Linux VM which already has these options set.

Run the domserver using the following command:

```
docker run --link dj-mariadb:mariadb -it -e MYSQL_HOST=mariadb -e MYSQL_USER=domjudge -e MYSQL_DATABASE=domjudge -e MYSQL_PASSWORD=djpw -e MYSQL_ROOT_PASSWORD=rootpw -p 12345:80 --name domserver domjudge/domserver:latest
```

If a specific DOMjudge version is required instead of the 'latest', replace 'latest' with the DOMjudge version (e.g. '5.3.0').
