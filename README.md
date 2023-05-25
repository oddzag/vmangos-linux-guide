This wiki was created for those who want to use Linux (Debian) to host a VMaNGOS server. Step-by-step instructions are surprisingly in short supply, so I decided to write this as I figured out VMaNGOS myself.

`*` - optional

## Required software
* g++ Compiler
* CMake
* Git

## Required dependencies
* ACE
* TBB
* MySQL Connector/C or Mariadb
* OpenSSL
* Zlib

## Initial setup*
This guide assumes a fresh, brand new install of Debian, so you may need to deviate. I'm using an LXC container with Debian 11 as a template on Proxmox.

```
apt update
apt upgrade -y
apt install sudo
adduser oddzag
//make a new username so everything isn't done under root
usermod -aG sudo oddzag
su - oddzag

```

## Installing software/dependencies 
```
sudo apt install g++
sudo apt install -qq libace-dev -y
export ACE_ROOT=/usr/include/ace
sudo apt install -y libtbb-dev
export TBB_ROOT_DIR=/usr/include/tbb
sudo apt install git -y
sudo apt install cmake -y
sudo apt-get install default-libmysqlclient-dev -
sudo apt install openssl
sudo apt install libssl-dev
sudo apt install build-essential checkinstall zlib1g-dev -y
mkdir vmangos
cd vmangos
git clone -b development https://github.com/vmangos/core
mkdir build
cd build
cmake ~/vmangos/core -DDEBUG=0 -DSUPPORTED_CLIENT_BUILD=5875 -DUSE_EXTRACTORS=0 -DCMAKE_INSTALL_PREFIX=~/vmangos
make -j4
make install
```

## Setup an SFTP client to the server*
I primarily use Windows 11, so it's easiest to just use an FTP client with a GUI. Check out [WinSCP](https://winscp.net/eng/download.php). Makes copying and managing files on the server much easier. If I'm not mistaken, you cannot ssh into root on a fresh install of Debian, so you'll either have to configure access for root or make a new username as I did above.

## Configure server firewall*
Debian does not automatically have a firewall enabled. You can just use [ufw](https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29)
```
sudo apt install ufw -y
sudo ufw enable -y
sudo ufw allow 22       //this allows ssh
sudo ufw allow 3306     //this allows mysql
sudo ufw allow 3724     //this port and the one below are for WoW
sudo ufw allow 8085
```

## Extracting game data
"You can't natively run the .exe files for extracting data from the client, but you can just get those files from the repack anyway." - codestothestars
1. Download the 1.12 repack from [here](https://www.mediafire.com/?dwr3wcqx90f9xnm)
2. Copy every in the `data` folder from the repack into `~/vmangos` on the server

## Configure MySQL/Mariadb
```
sudo apt install mariadb-server
sudo mysql_secure_installation
```

These are my answers for the MySQL secure installation prompts:
* Switch to unix_socket authentication [Y/n] n
* Change the root password? [Y/n] n
* Remove anonymous users? [Y/n] y
* Disallow root login remotely? [Y/n] y
* Remove test database and access to it? [Y/n] y
* Reload privilege tables now? [Y/n] y

Now to configure an administrative username for Mariadb
```
sudo mariadb
> GRANT ALL ON *.* TO 'username'@'X.X.X.X' IDENTIFIED BY 'password' WITH GRANT OPTION;
Query OK, 0 rows affected (0.004 sec)

> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.000 sec)
```

Change:
* **username** to the username you created earlier (or already have created)
* **X.X.X.X** to whatever IP you're connecting to the server from
* **password** self-explanatory

You'll need to allow remote connections to Mariadb. 
```
sudo nano /etc/mysql/my.cnf
```
Add the following to the bottom of the file:
```
skip-networking=0
skip-bind-address
```
_there may be security concerns with this method, this is just for a LAN setup_

### Create VManGOS databases
I'm doing this through [HeidiSQL](https://www.heidisql.com/download.php?download=installer). If you're a psychopath and want to configure them through terminal, you're on your own

_for a video demonstration, check out [this video](https://www.youtube.com/watch?v=h3E3wwNN4qM&t=39m49s) from [Digital Scriptorium](https://www.youtube.com/@Digital-Scriptorium)_

1. Install [HeidiSQL](https://www.heidisql.com/download.php?download=installer)
2. Connect to your server using the administrative username you setup earlier
3. Create 4 new databases: logs, realmd, characters and mangos
4. Load and execute the relevant SQL file for each database
   * logs = ~/vmangos/core/sql/logs.sql
   * realmd = ~/vmangos/core/sql/logon.sql
   * characters = ~/vmangos/core/sql/characters.sql
   * mangos = [June 2021](https://github.com/brotalnia/database/raw/master/world_full_14_june_2021.7z)

## Edit your config files
For both of these parts, keep this in mind: if you're hosting the server at a different IP, when you try connecting to it from your client on your personal PC, you'll need to set the realmlist to the IP of the server. You can leave it localhost (127.0.0.1) here if the server is on the same IP as the database.

### mangos config
1. Open ~/vmangos/etc/mangosd.conf.dist
2. Find the following lines:
   * LoginDatabase.Info              = "127.0.0.1;3306;mangos;mangos;realmd"
   * WorldDatabase.Info              = "127.0.0.1;3306;mangos;mangos;mangos"
   * CharacterDatabase.Info          = "127.0.0.1;3306;mangos;mangos;characters"
   * LogsDatabase.Info               = "127.0.0.1;3306;mangos;mangos;logs"
3. Edit them appropriately; the first `mangos` is the username, the second is your password

### realmd config
1. Open ~/vmangos/etc/realmd.conf.dist
2. Find the following line:
   * LoginDatabaseInfo = "127.0.0.1;3306;mangos;mangos;realmd"

## Apply latest migrations
If you attempt to run the server at this point, you'll get an error stating it's missing migrations. As I said, I'm using Windows 11 to configure this all remotely. So I copied ~/vmangos/core/sql/migrations/ to my PC and then ran `merge.bat`. It'll merge all the migration sql files into a single file for each database. Load and execute the sql files into their respective databases: 
* characters_db_updates.sql => characters
* logon_db_updates.sql => realmd
* logs_db_updates.sql => logs
* world_db_updates.sql => mangos - choose "load files directly into editor" and then execute

## Run the server
You'll need two terminals open for this. 

```
cd ~/vmangos/bin
./mangosd
```

```
cd ~/vmangos/bin
./realmd
```
If you get the following error:
```
ERROR: Check existing of map file './maps/0004331.map': not exist!
ERROR: Correct *.map files not found in path './maps' or *.vmtree/*.vmtile
files in './vmaps'. Please place *.map and vmap files in appropriate directories or correct the DataDir value in the mangosd.conf file.
```

Try editing `~/vmangos/etc/mangosd.conf` where `DataDir = "."` to `DataDir = "../"`. That's what worked for me.

In the terminal for `mangosd`, create a test account with the command: `account create username password`. Change `username` and `password` to your desired settings.

## Account rank
"gmlevels are now stored in account_access table, you can set it manually in SQL but it's easier through console" - Wall

To change a user's rank, in the mangosd console, type `account set gmlevel [account name] [rank]`. 6 is the highest rank
