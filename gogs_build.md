# Gogs Build

## Assumptions

* Host running CentOS
* Access to yum repo(s)
* The Gogs binary archive downloaded to /tmp/ on the target system
    * This can be found at https://gogs.io/docs/installation/install_from_binary

## Install the MariaDB (MYSQL) Server

1. `$ sudo yum install -y mariadb mariadb-server`
1. `$ sudo systemctl start mariadb`
1. `$ sudo systemctl enable mariadb`
1. `$ mysql_secure_installation`
    * Accept all default answers, and supply a password IAW local SOP

## Install the git Tools

1. `$ sudo yum install -y git`

## Create Database and User for Gogs

1. `$ mysql -u root -p`
    1. Enter password set in previous step when prompted
    1. At the MariaDB prompt, enter the following:
        1. `MariaDB [(none)]> CREATE DATABASE gogs;`
            * This creates a new database to hold the git repo info
        1. `MariaDB [(none)]> GRANT ALL ON gogs.* TO 'git'@'localhost' IDENTIFIED BY 'password';`
            * This creates a new MySQL user name 'git'. Replace 'password' above with desired password.
        1. `MariaDB [(none)]> FLUSH PRIVILEGES;`
            * Flush the MySQL privileges to ensure that the new git user can log in
    1. Press **Ctrl+D** to exit MariaDB

## Firewall

### Install `firewalld` Service (Optional)

The firewall service should be installed on the box as part of the STIG, but in the event it isn't (the command to open the port will fail), you'll need to install it:

1. `$ sudo install -y firewalld`
1. `$ sudo systemctl start firewalld`
1. `$ sudo systemctl enable firewalld`

### Open Firewall Port for Gogs

Our installation of Gogs will listen on port 8080. The local firewall must be opened accordingly.

1. `$ sudo firewall-cmd --add-port=8080/tcp --permanent`
1. `$ sudo systemctl restart firewalld`

## Install Gogs

### Create `git` User and Install Gogs to its Home Folder

1. `$ sudo useradd git`
    * This creates a new user on the system named 'git'
1. `$ sudo passwd git`
    * Set password IAW local SOP
1. `$ sudo chown git:git /tmp/gogs_v0.8.25_linux_amd64.tar.gz`
1. `$ su git`
1. If not in the git user's home folder, navigate there:  
    `$ cd ~`
1. `$ cp /tmp/gogs_v0.8.25_linux_amd64.tar.gz .`
1. `$ tar -xzf gogs_v0.8.25_linux_amd64.tar.gz`

### Run Gogs for the First Time and Configure

1. `$ gogs/gogs web -p 8080`
1. Launch a web browser and navigate to the IP of the Gogs host on port 8080. This will allow you to initialize Gogs.
    * Fill out the First-time Run form with the following values:

        | Setting              | Value                            |
        |----------------------|----------------------------------|
        | Database Type        | MySQL                            |
        | Host                 | `127.0.0.1:3306`                 |
        | User                 | git                              |
        | Password             | *set in earlier step*            |
        | Database Name        | gogs                             |
        | Application Name     | Gogs                             |
        | Repository Root Path | `/home/git/gogs-repositories`    |
        | Run User             | git                              |
        | Domain               | localhost                        |
        | SSH Port             | 22                               |
        | HTTP Port            | 8080                             |
        | Application URL      | `http://gogs.domain.local:8080/` |  

    * Optional Settings
        * Server and Other Services Settings
            * Check "Disable Gravatar Service"
            * Check "Disable Self-registration"
        * Admin Account Settings

            | Setting     | Value                        |
            |-------------|------------------------------|
            | Username    | gogs-admin *(or as desired)* |
            | Password    | *IAW local SOP*              |
            | Admin Email | gogs-admin@domain.local      |  

    * Click **Install Gogs**

### Further Configuration

Additional configuration can be done by modifying `/home/git/gogs/custom/conf/app.ini`

## Running Gogs on Boot (Making Gogs a Service)

Create a `systemd` script to launch Gogs on boot

1. Stop the Gogs server by pressing **Ctrl+C**
1. Log out the git user by pressing **Ctrl+D**
1. `$ sudo vi /etc/systemd/system/gogs.service`

```
[Unit]
Description=Gogs
After=syslog.target
After=network.target
After=mariadb.service

[Service]
Type=simple
User=git
Group=git
WorkingDirectory=/home/git/gogs
ExecStart=/home/git/gogs/gogs web
Restart=always
Environment=USER=git HOME=/home/git

[Install]
WantedBy=multi-user.target
```

1. `sudo systemctl start gogs`
    * Confirm the server is running by refreshing the browser
1. `sudo systemctl enable gogs`
1. Restart the host and confirm the Gogs service comes up properly