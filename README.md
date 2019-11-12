#### MORUMOTTO

1. Installation
    1. With installer
    2. Manual installation
2. Getting Started
3. Read the docs
## Installation

### METHOD 1 : With installer

- Step 1, in mysql _(not required, but recommanded)_

    ```sql
    CREATE DATABASE SIQACO;
    CREATE USER 'siqaco_user'@'localhost' IDENTIFIED BY 'some password';
    GRANT ALL ON SIQACO.* TO 'siqaco_user'@'localhost';
    FLUSH PRIVILEGES;
    ```

- Step 2

    ```sh
    git clone https://k2.ipgp.fr/geber/siqaco
    cd siqaco  
    ./install.sh
    ```

_You will need to confirm some things during installation, don't run away from your keyboard_

- Required for the install script to run correctly : __mysql__ OR __mariadb__, __python3__ _(Even if your python system is 2.7, you must be able to install libpython3.6-dev)_
- Installer tested on __Ubuntu 16.10__, __18.04__, __18.10__, __19.04__ & __Debian 9__

### METHOD 2 : Manual installation



##### Get source code

```sh
git clone https://k2.ipgp.fr/geber/siqaco
cd siqaco
```

For the following steps, you must stay in this folder


##### Dependencies

1. Before starting, make sure you have the following packages installed :

    - python
    - python-pip
    - mysql-server OR mariadb-dev libmariadb-dev-compat (depending on your linux distribution)
    - wget
    - libmysqlclient-dev OR libmariadb-dev (depending on your linux distribution)
    - wget
    - libpython3.6-dev
    - supervisor


    In Ubuntu:  

    ```sh
    sudo apt-get install -y python wget make python-pip mysql-server libmysqlclient-dev python3-dev libpython3.6-dev python3-venv supervisor
    ```

    In Debian:
    ```sh
    sudo apt-get install -y python wget make python-pip mariadb-dev libmariadb-dev-compat libmariadb-dev python3-dev libpython3.6-dev python3-venv supervisor
    ```


2. Install dataselect (version >= 3.20 ) in /siqaco/bin : ([Visit the IRIS GitHub page to install](https://github.com/iris-edu/dataselect "github.com/iris-edu/dataselect"))


3. Install qmerge in /siqaco/bin : ([Download tar file here](http://www.ncedc.org/qug/software/ucb/qmerge.2014.329.tar.gz "ncedc.org/qug/software/ucb/qmerge.2014.329.tar.gz")). Then copy the qmerge executable to /siqaco/bin.

4. Install msi in /siqaco/bin : ([Visit the IRIS GitHub page to install](https://github.com/iris-edu/msi "github.com/iris-edu/msi"))

5. Install sdrsplit in /siqaco/bin : ([Download tar file here](http://www.ncedc.org/qug/software/ucb/sdrsplit.2013.260.tar.gz)) Then copy the sdrsplit executable to /siqaco/bin.


6. Install RabbitMQ (The parallel task are handled by the task queue manager [Celery](http://docs.celeryproject.org/en/latest/getting-started/first-steps-with-celery.html#first-steps "First steps with celery").
Celery requires to install a broker, SiQaCo uses RabbitMQ)  

    In Ubuntu/Debian :    

    ```sh
    sudo apt-get install rabbitmq-server
    ```


##### Initialise the environment

1. You must know your system python version. For that, in a terminal window, type:  

    ```sh
    python --version
    ```  

2. Create the virtual environment (inside the _siqaco_ root directory)

    - If your python is < 3.3 :  

        ```sh
        virtualenv siqaco-env -p python3.6
        ```  

    - If your python is >= 3.4 :  

        ```sh
        python3 -m venv siqaco-env
        ```

3. Install requirements :  

    ```sh
    source siqaco-env/bin/activate  
    pip install numpy # needs to be installed first, doesn't work within requirements
    pip install -r requirements.txt
    ```

##### Initialise LEAPSECONDS

  - Check if you have already initialised the leapsecond var in your shell :

    ```sh
    echo $LEAPSECONDS
    ```

  - If prompt returns an empty string, you may use the leapsecond list provided here ()

    ```sh
    echo "export LEAPSECONDS='${HOME}/siqaco/leapsecond.list'" >> ${HOME}/.bashrc
    ```

    - Be sure to add new leap seconds to your LEAPSECONDS file when they are issued ! To update the leapsecond file : go to http://www.ncedc.org/ftp/pub/programs/leapseconds



##### Initialise your MySQL database

Now, you will have to create a SiQaCo database within MySQL. To do so:

+ connect to MySQL as root:  

    ```sh
    mysql -u root -p
    ```

+ enter root password
+ in the _mysql_ prompt, type:


    ```sql
    CREATE DATABASE SIQACO;
    CREATE USER 'siqaco_user'@'localhost' IDENTIFIED BY 'your_password';
    GRANT ALL ON SIQACO.* TO 'siqaco_user'@'localhost';
    FLUSH PRIVILEGES;
    ```

Then add to your custom settings (siqaco/siqaco/custom_settings.py) :

```python
DATABASE_ENGINE = 'django.db.backends.mysql'
DATABASE_NAME = 'SIQACO'
DATABASE_USER_NAME = 'siqaco_user'
DATABASE_PASSWORD = 'your_password'
DATABASE_HOST = 'your_host' #defaults to 127.0.0.1 to use local database
```  

There is a template in the siqaco/siqaco folder. Just duplicate it and remove the .template at the end of the file name and put your own settings in it. For the Allowed hosts, see below ("Access from another computer in the same network")


_Note: you can change the database, user and password at your convinience, but they must be the same in the database and in the custom settings file_

If you want to use another RDBMS (PostGreSQL for example), please check the [Django documentation](https://docs.djangoproject.com/en/dev/topics/install/#database-installation "Django Database Installation")


##### Bind database tables to SiQaCo objects

+ Inside the _siqaco_ root directory, execute

    ```sh
    python manage.py migrate  
    python manage.py makemigrations seismicarchive monitoring qualitycontrol logdb
    python manage.py migrate
    ```  


##### Create an admin user :

+ Inside the _siqaco_ directory, execute

    ```sh
    python manage.py createsuperuser
    ```

Fill the required informations. Don't forget it, it will be your administrator user for the SIQACO software

All good, you're ready to go now !


##### Deamonize it :
  + Automatically :

<details>
<summary>Click to display script</summary>

In a terminal, paste the following code (or run is in a script) :

```sh
dir=$(pwd)
cat <<EOT >> morumotto.conf
[program:morumotto_celery]
command = ${dir}/siqaco-env/bin/celery -A siqaco worker -l info
user = ${USER}
directory = ${dir}
logfile = /var/log/supervisor/morumotto_celery.log
logfile_maxbytes = 50MB
logfile_backups=10
loglevel = info
autostart = true
autorestart = true

[program:morumotto_flower]
command = ${dir}/siqaco-env/bin/celery flower -A siqaco --address=127.0.0.1 --port=5555
user = ${USER}
directory = ${dir}
logfile = /var/log/supervisor/morumotto_flower.log
logfile_maxbytes = 50MB
logfile_backups=10
loglevel = info
autostart = true
autorestart = true

[program:morumotto_runserver]
command = ${dir}/siqaco-env/bin/python manage.py runserver 0.0.0.0:8000
user = ${USER}
directory = ${dir}
logfile = /var/log/supervisor/morumotto_runserver.log
logfile_maxbytes = 50MB
logfile_backups=10
loglevel = info
autostart = true
autorestart = true

[group:morumotto]
programs=morumotto_celery,morumotto_flower,morumotto_runserver
priority=999

EOT
sudo mv morumotto.conf /etc/supervisor/conf.d/ || exit
sudo supervisorctl reread
sudo supervisorctl update
```

</details>

  + Manually :

Create a file named morumotto.conf, and paste the following text inside :

<details>
<summary>Click to display text</summary>

**WARNING : you must change ${dir} with your siqaco directory and ${USER} with your user name**

_morumotto.conf_
```sh
[program:morumotto_celery]
command = ${dir}/siqaco-env/bin/celery -A siqaco worker -l info
user = ${USER}
directory = ${dir}
logfile = /var/log/supervisor/morumotto_celery.log
logfile_maxbytes = 50MB
logfile_backups=10
loglevel = info
autostart = true
autorestart = true

[program:morumotto_flower]
command = ${dir}/siqaco-env/bin/celery flower -A siqaco --address=127.0.0.1 --port=5555
user = ${USER}
directory = ${dir}
logfile = /var/log/supervisor/morumotto_flower.log
logfile_maxbytes = 50MB
logfile_backups=10
loglevel = info
autostart = true
autorestart = true

[program:morumotto_runserver]
command = ${dir}/siqaco-env/bin/python manage.py runserver 0.0.0.0:8000
user = ${USER}
directory = ${dir}
logfile = /var/log/supervisor/morumotto_runserver.log
logfile_maxbytes = 50MB
logfile_backups=10
loglevel = info
autostart = true
autorestart = true

[group:morumotto]
programs=morumotto_celery,morumotto_flower,morumotto_runserver
priority=999
```

</details>

Then move it to the /etc/supervisor/conf.d, and update supervisor

```sh
sudo cp morumotto.conf /etc/supervisor/conf.d/morumotto.conf
sudo supervisorctl reread
sudo supervisorctl update
```
## Getting Started

##### Check that the deamon is running :

```sh
sudo supervisorctl status
```

This should display something like :

```
morumotto:morumotto_celery       RUNNING   pid 3939, uptime 0:38:21
morumotto:morumotto_flower       RUNNING   pid 3937, uptime 0:38:22
morumotto:morumotto_runserver    RUNNING   pid 3938, uptime 0:38:21
```

##### Open you browser and enter the following URL:

+ [http://127.0.0.1:8000/home](http://127.0.0.1:8000/home)

##### Visit the admin page:

+ [http://127.0.0.1:8000/admin](http://127.0.0.1:8000/admin)


##### Access from another computer in the same network :

+ add the server's IP (1.2.3.4 for example) where SIQACO is located to the ALLOWED_HOST in siqaco/siqaco/custom_settings.py

Then in a terminal, enter

```sh
source siqaco-env/bin/activate  
python manage.py runserver 0.0.0.0:8080
```

Note that you have changed the port used by SIQACO from 8000 (default) to 8080

Then from any computer in the network, visit the following URL:

+ [http://1.2.3.4:8080/home](http://1.2.3.4:8080/home)



##### To run the software without the daemon :
_Note : this is only for development_

In a terminal enter :  

```sh
source siqaco-env/bin/activate  
python manage.py runserver
```


## Read the docs

[user_guide.pdf](https://k2.ipgp.fr/geber/siqaco/raw/master/docs/user_guide.pdf?inline=false)