# Integrate SSO for online-test application

## Install Dependencies

```sh
sudo dnf install git python3 python3-pip wget -y
```

```sh
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run hello-world
sudo usermod -aG docker $USER
```

```sh
git clone https://github.com/FOSSEE/online_test.git
cd online_test
cp requirements/*.txt docker/Files/
```

Edit `docker/Files/requirements-codeserver.txt` and find psutil and change it to `psutil==5.4.3` 
Edit `docker/Files/requirements-common.txt` and find psutil and change it to `psutil==5.4.3` 

Edit `docker/Dockerfile_django`


%% ```sh
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
source ~/miniconda3/bin/activate
conda init --all
```

```sh
# Enter the MySQL shell:

sudo mysql -u root

#Then create the database and user:
CREATE DATABASE yaksh; 
CREATE USER 'yaksh_user'@'localhost' IDENTIFIED BY 'yourpassword'; 
GRANT ALL PRIVILEGES ON yaksh.* TO 'yaksh_user'@'localhost'; 
FLUSH PRIVILEGES; 
EXIT;
```

```sh
conda create --name yaksh_env python=3.8
conda activate yaksh_env
git clone https://github.com/FOSSEE/online_test.git
cd online_test
pip3 install --upgrade pip==24.0
pip3 install -r requirements/requirements-common.txt
pip3 install -r requirements/requirements-production.txt
sudo pip3 install -r requirements/requirements-codeserver.txt
```

```sh
mv .sampleenv .env
vi/nano .env
DB_ENGINE=mysql # Or psycopg (postgresql), sqlite3 (SQLite)
DB_NAME=yaksh
DB_USER=root
DB_PASSWORD=mypassword # Or the password used while creating a Database
DB_HOST=localhost
DB_PORT=3306
```
Edit `docker/Files/requirements-codeserver.txt` and find psutil and change it to `psutil==5.4.3` 
 %%





