# Drupal and Keycloak on seperate VMs

## 1. Setup VM for both Drupal and Keycloak

- Install Rocky Linux ISO image from Rocky linux official site
- Open VirtualBox and create two new machines for openplc and keycloak
- After creating machines, Navigate to File > Tools > Networks > Host-only Networks > Create 
- Check Configure Adapter Manually and make sure DHCP server is disabled
- Now open created machine's setting. Under Network, set
  - Adapter 1
    - Attached to: NAT
    - Name: Select the NAT network we created
      Leave rest as it is
  - Adapter 2
    - Turn on Enable Network Adapter
    - Attached to: Host-only adapter
      Leave rest as it is
  - Save
- Start both the VMs 
- Run these following commands 
```sh
nmcli con show
nmcli connection modify enp0s8 ipv4.addresses 192.168.56.10/24
nmcli connection modify enp0s8 ipv4.method manual
nmcli connection modify enp0s8 ipv4.gateway ""
nmcli connection modify enp0s8 ipv4.dns "8.8.8.8"
nmcli connection up enp0s8
nmcli connection show
```

## 2. Drupal VM Setup

### A. Install and configure packages

SSH into Drupal VM

```sh
ssh your_username@your_drupal_vm_ip
```

Update and install required packages

```sh
sudo dnf update -y
sudo dnf install mysql8.4-server wget git podman jq slirp4netns -y
```

Allow port 8081

```sh
sudo firewall-cmd --permanent --add-port=8081/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

### B. Configure MySQL

Enable and start MySQL service

```sh
sudo systemctl enable mysqld
sudo systemctl start mysqld
```

Set MySQL password

```sh
sudo mysql_secure_installation
Press y|Y for Yes, any other key for No: n
Please set the password for root here.

New password:

Re-enter new password:

Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y

Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y

All done!
```

Restart mysqld

```sh
sudo systemctl restart mysqld
sudo systemctl status mysqld
# Try logging into mysql
mysql -u root -p
```

### C. Creating podman container

Setup project directory

```sh
mkdir openplc
cd openplc
# Download these 4 files init_scripts, Dockerfile, sites.json and openplc.sql
# Docker file
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/openplc/Dockerfile
# init_sites script
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/openplc/init_sites
# sites.json
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/openplc/sites.json
# Add dump file
```

Run init_sites script

```sh
chmod +x init_sites
./init_sites
```

Fill the fields

```sh
Enter MySQL root password: {your_sql_root_password}
!! Kindly use lowercase for naming and don't include white spaces !!
Enter name of the site: openplc
Enter name of site database: openplcdb
Add date tag to database name? [default: _2025_10_10]:
Enter database username you wish to use: openplcuser
Enter path to SQL dumpfile for site: ./openplc.sql
Choose Drupal version for site (10, 11): 10
Enter branch name to clone: main

Is this information correct? (y/n): y

Do you want to create and use a new volume? [y/N]: y

Enter custom volume name [default: openplc_volume]:

Proceed with image build? (y/N): y
```

Check if container is running

## 3. Keycloak VM setup

### A. Install and configure packages

SSH into Drupal VM

```sh
ssh your_username@your_drupal_vm_ip
```

Update and install required packages

```sh
sudo dnf update -y
# Install mysql, wget and java
sudo dnf install mysql8.4-server wget java-21-openjdk-devel -y
```

Allow port 8080

```sh
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

### B. Install keycloak

```sh
cd /opt
# Download keycloak release
sudo wget https://github.com/keycloak/keycloak/releases/download/26.4.0/keycloak-26.4.0.zip
sudo unzip keycloak-26.4.0.zip
sudo mv keycloak-26.4.0 keycloak

# Create a dedicated user for Keycloak
sudo groupadd keycloak
sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
sudo chown -R keycloak:keycloak /opt/keycloak

# Adjust SELinux context so that keycloak files can be executed without being blocked
sudo chcon -R -u system_u -t usr_t /opt/keycloak
```

### C. Create database and user for keycloak

```sh
# Login to MySQl
sudo mysql -u root -p

CREATE DATABASE keycloakdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'keycloakuser'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON keycloakdb.* TO 'keycloakuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Download MySQL JDBC Driver and add it into keycloak

```sh
cd /opt/keycloak/providers
sudo wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar
sudo chown keycloak:keycloak mysql-connector-j-8.4.0.jar
# SELinux file context for provider JAR
sudo chcon -u system_u -t usr_t /opt/keycloak/providers/mysql-connector-j-8.4.0.jar
```

Edit `/opt/keycloak/conf/keycloak.conf`:

```shell
sudo vi /opt/keycloak/conf/keycloak.conf

# Update/add these lines

#Configure keycloak to use MySQL instead of H2
db=mysql
db-username=keycloakuser
db-password=your_password
db-url=jdbc:mysql://localhost:3306/keycloakdb

# Proxy & Hostname settings
proxy=edge
proxy-headers=xforwarded

hostname=your_ip_addr
http-enabled=true
https-enabled=false
http-port=8080
```

Create a temporary admin:

```sh
cd /opt/keycloak
sudo -u keycloak ./bin/kc.sh build
export P=your_password
sudo --preserve-env=P ./bin/kc.sh bootstrap-admin user --username admin --password:env P
```

> **Note**: This admin is temporary. After logging into the Admin Console, create a permanent admin user with the `admin` role, then remove the bootstrap user.

Run and verify

```shell
cd /opt/keycloak
sudo -u keycloak ./bin/kc.sh start --optimized
```

Now login to keycloak console with temporary credentials and create new admin user and delete the bootstrap user

Create the service file `/etc/systemd/system/keycloak.service`:

```shell
[Unit]
Description=Keycloak Authorization Server
After=network.target

[Service]
Type=idle
User=keycloak
Group=keycloak
WorkingDirectory=/opt/keycloak
ExecStart=/opt/keycloak/bin/kc.sh start --optimized
LimitNOFILE=102400
LimitNPROC=102400
TimeoutStartSec=600
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

Set correct owner and selinux

```shell
sudo chcon -u system_u -t systemd_unit_file_t /etc/systemd/system/keycloak.service
sudo chown -R keycloak:keycloak /opt/keycloak
sudo chmod -R 755 /opt/keycloak

# Enable keycloak daemon service.
sudo systemctl daemon-reload
sudo systemctl enable --now keycloak
sudo systemctl status keycloak
```

## 4. Keycloak script

Create dir and set permissions

```
mkdir ~/keycloak
sudo chown -R keycloak:$USER keycloak/
cd keycloak
```

Create the following files with given contents

config.json
```json
[
  {
    "keycloak_realm": "",
    "keycloak_base": "",
    "keycloak_user": "",
    "keycloak_password": "",
    "drupal_base": "",
    "keycloak_client_id": ""
  }
]
```

client.json
```json
{
  "clientId": "",
  "enabled": true,
  "publicClient": false,
  "protocol": "openid-connect",
  "clientAuthenticatorType": "client-secret",
  "standardFlowEnabled": true,
  "redirectUris": [""]
}
```

Create a script file and add the following 

keycloak
```sh
#!/bin/bash

checkInput() {
	if [[ "$KEYCLOAK_REALM" != "${KEYCLOAK_REALM,,}" || "$KEYCLOAK_REALM" =~ [[:space:]] ]]; then
		echo "Error: Keycloak Realm must be lowercase only with no spaces." >&2
		exit 1
	fi

	if [[ "$KEYCLOAK_CLIENT_ID" != "${KEYCLOAK_CLIENT_ID,,}" || "$KEYCLOAK_CLIENT_ID" =~ [[:space:]] ]]; then
		echo "Error: Keycloak Client ID must be lowercase only with no spaces." >&2
		exit 1
	fi

	if ! [[ "$KEYCLOAK_BASE" =~ ^https?:// ]]; then
		echo "Error: Keycloak Base URL must start with http:// or https://." >&2
		exit 1
	fi

}

getInput() {
	local tmp_keycloak_realm tmp_keycloak_base tmp_keycloak_user tmp_keycloak_password tmp_keycloak_client_id tmp_drupal_base
	local confirm

	while true; do
		echo "!! Kindly use lowercase for naming and don't include white spaces !!"
		read -r -p "Enter Keycloak realm: " tmp_keycloak_realm
		read -r -p "Enter Keycloak base url : " tmp_keycloak_base
		read -r -p "Enter Keycloak admin user: " tmp_keycloak_user
		read -s -p "Enter Keycloak admin password: " tmp_keycloak_password
		echo
		read -r -p "Enter Keycloak client ID you want to create: " tmp_keycloak_client_id
		read -r -p "Enter Drupal base url : " tmp_drupal_base
		echo

		echo "Summary of entered information:"
		echo "Keycloak Realm        : $tmp_keycloak_realm"
		echo "Keycloak Base URL     : $tmp_keycloak_base"
		echo "Keycloak Admin User   : $tmp_keycloak_user"
		echo "Keycloak Client ID    : $tmp_keycloak_client_id"
		echo "Drupal Base URL       : $tmp_drupal_base"
		echo

		read -r -p "Is this information correct? (y/n): " confirm
		if [[ "$confirm" =~ ^[Yy]$ ]]; then
			KEYCLOAK_REALM="$tmp_keycloak_realm"
			KEYCLOAK_BASE="$tmp_keycloak_base"
			KEYCLOAK_USER="$tmp_keycloak_user"
			KEYCLOAK_PASSWORD="$tmp_keycloak_password"
			KEYCLOAK_CLIENT_ID="$tmp_keycloak_client_id"
			KEYCLOAK_DRUPAL_BASE="$tmp_drupal_base"
			echo "Confirmed. Continuing..."
			break
		else
			echo "Trying again..."
			echo
		fi
	done

	checkInput && echo "Config passed input validation"

	return 0
}

updateClientJSON() {
	jq --arg client_id "$KEYCLOAK_CLIENT_ID" \
		--arg redirect_uri "$DRUPAL_BASE/openid-connect/keycloak" \
		'.clientId = $client_id | .redirectUris = [$redirect_uri]' "$CLIENT_JSON_PATH" >tmp && mv tmp "$CLIENT_JSON_PATH"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to update $CLIENT_JSON_PATH." >&2
		return 1
	fi
	echo "$CLIENT_JSON_PATH updated with Client ID and Redirect URI."
	return 0
}

updateConfigJSON() {

	jq --arg realm "$KEYCLOAK_REALM" \
		--arg base "$KEYCLOAK_BASE" \
		--arg user "$KEYCLOAK_USER" \
		--arg password "$KEYCLOAK_PASSWORD" \
		--arg client_id "$KEYCLOAK_CLIENT_ID" \
		--arg drupal_base "$KEYCLOAK_DRUPAL_BASE" \
		'map(
           if .keycloak_realm == "" or .keycloak_realm == null then
               . + {
                   "keycloak_realm": $realm,
                   "keycloak_base": $base,
                   "keycloak_user": $user,
                   "keycloak_password": $password,
                   "keycloak_client_id": $client_id,
                   "keycloak_drupal_base": $drupal_base
               }
           else .
           end
       )' "$CONFIG_JSON_PATH" >tmp && mv tmp "$CONFIG_JSON_PATH"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to update $CONFIG_JSON_PATH." >&2
		return 1
	fi
	echo "Config saved to $CONFIG_JSON_PATH"

	return 0
}

createClient() {
	echo "Keycloak login"
	sudo -u keycloak /opt/keycloak/bin/kcadm.sh config credentials \
		--server "$KEYCLOAK_BASE" \
		--realm "$KEYCLOAK_REALM" \
		--user "$KEYCLOAK_USER" \
		--password "$KEYCLOAK_PASSWORD"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to login. Check server URL, realm, user, or password." >&2
		return 1
	fi
	echo "Logged in successfully."

	echo "Creating Keycloak client '$KEYCLOAK_CLIENT_ID'..."
	sudo -u keycloak /opt/keycloak/bin/kcadm.sh create clients -r "$KEYCLOAK_REALM" -f "$CLIENT_JSON_PATH"

	if [ $? -ne 0 ]; then
		echo "Error: Failed to create Keycloak client." >&2
		return 1
	fi
	echo "Keycloak client '$KEYCLOAK_CLIENT_ID' created."

	echo "Retrieving client details and saving to secret.json..."
	sudo -u keycloak /opt/keycloak/bin/kcadm.sh get clients -r "$KEYCLOAK_REALM" -q clientId="$KEYCLOAK_CLIENT_ID" --fields clientId,secret >secret.json

	if [ $? -ne 0 ]; then
		echo "Warning: Could not retrieve client details" >&2
	else
		echo "Client details saved to secret.json"
	fi

	return 0
}

KEYCLOAK_REALM=""
KEYCLOAK_BASE=""
KEYCLOAK_USER=""
KEYCLOAK_PASSWORD=""
KEYCLOAK_CLIENT_ID=""
CLIENT_JSON_PATH="client.json"
CONFIG_JSON_PATH="config.json"

main() {
	if ! command -v jq &>/dev/null; then
		echo "Error: 'jq' command not found. Please install 'jq'" >&2
		exit 1
	fi

	if [[ ! -f "$CLIENT_JSON_PATH" ]]; then
		echo "Error: Client configuration file '$CLIENT_JSON_PATH' not found." >&2
		exit 1
	fi

	if [[ ! -f "$CONFIG_JSON_PATH" ]]; then
		echo "Error: Main configuration file '$CONFIG_JSON_PATH' not found." >&2
		exit 1
	fi

	{
		getInput &&
			updateClientJSON &&
			updateConfigJSON &&
			createClient &&
			echo "::: Keycloak Client Setup Complete :::"
	} || {
		echo "!!! Keycloak Client Setup Failed !!!"
	}
}

time main "$@"
```

Run the script 
```sh
chmod +x keycloak
./keycloak
```

This will generate secret.json file copy the contents of the file

## 6. Drupal script

Create a dir and copy required files
```sh 
mkdir drupal
cd drupal
```

Add the file which you got from keycloak script

```json
[ {
  "clientId" : <your_clientId>,
  "secret" : <your_clientSecret>
} ]
```

Create a script file

drupal
```sh
#!/bin/bash

checkPrereqs() {
	if ! command -v podman &>/dev/null; then
		echo "Error: 'podman' command not found. Please install 'podman'." >&2
		exit 1
	fi

	if ! command -v jq &>/dev/null; then
		echo "Error: 'jq' command not found. Please install 'jq'." >&2
		exit 1
	fi

	if [[ ! -f "$SECRET_FILE" ]]; then
		echo "Error: Client secret file '$SECRET_FILE' not found." >&2
		echo "Please run the keycloak.sh script first to generate it." >&2
		exit 1
	fi

	if ! podman ps --format "{{.Names}}" | grep -q "^$CONTAINER_NAME$"; then
		echo "Error: Podman container '$CONTAINER_NAME' is not running." >&2
		exit 1
	fi

	echo "Prerequisites check passed."
}

getInput() {
	local tmp_keycloak_base tmp_keycloak_realm
	local confirm

	while true; do
		read -r -p "Enter Keycloak base url: " tmp_keycloak_base
		read -r -p "Enter Keycloak realm: " tmp_keycloak_realm
		echo

		echo "Summary of entered information:"
		echo "Keycloak Base URL     : $tmp_keycloak_base"
		echo "Keycloak Realm        : $tmp_keycloak_realm"
		echo

		read -r -p "Is this information correct? (y/n): " confirm
		if [[ "$confirm" =~ ^[Yy]$ ]]; then
			KEYCLOAK_BASE="$tmp_keycloak_base"
			KEYCLOAK_REALM="$tmp_keycloak_realm"
			echo "Confirmed. Continuing..."
			break
		else
			echo "Trying again..."
			echo
		fi
	done
	return 0
}

readSecrets() {
	echo "Reading client details from $SECRET_FILE..."

	CLIENT_ID=$(jq -r '.[0].clientId' "$SECRET_FILE")
	CLIENT_SECRET=$(jq -r '.[0].secret' "$SECRET_FILE")

	if [ -z "$CLIENT_ID" ] || [ "$CLIENT_ID" == "null" ]; then
		echo "Error: Could not read 'clientId' from $SECRET_FILE." >&2
		return 1
	fi

	if [ -z "$CLIENT_SECRET" ] || [ "$CLIENT_SECRET" == "null" ]; then
		echo "Error: Could not read 'secret' from $SECRET_FILE." >&2
		return 1
	fi

	echo "Successfully read Client ID: $CLIENT_ID"
	return 0
}

createClient() {
	echo "Configuring Drupal OpenID Connect..."

	KEYCLOAK_BASE="${KEYCLOAK_BASE%/}"

	podman exec -it "$CONTAINER_NAME" bash -c "
        vendor/bin/drush config:set openid_connect.settings.keycloak enabled true -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.client_id '$CLIENT_ID' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.client_secret '$CLIENT_SECRET' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_base '$KEYCLOAK_BASE' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_realm '$KEYCLOAK_REALM' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.userinfo_update_email false -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.enabled false -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.claim_name groups -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.split_groups false -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.split_groups_limit '0' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_groups.rules [] -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_sso true -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_sign_out true -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.check_session.enabled false -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.check_session.interval null -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.redirect_url '' -y &&
        vendor/bin/drush config:set openid_connect.settings.keycloak settings.keycloak_i18n_enabled false -y &&

        echo 'Rebuilding Drupal cache...' &&
        vendor/bin/drush cr &&

        echo 'Final configuration:' &&
        vendor/bin/drush config:get openid_connect.settings.keycloak
    "

	if [ $? -ne 0 ]; then
		echo "Error: Failed during Drupal configuration." >&2
		return 1
	fi

	return 0
}

KEYCLOAK_BASE=""
KEYCLOAK_REALM=""
CLIENT_ID=""
CLIENT_SECRET=""
CONTAINER_NAME="openplc_container"
SECRET_FILE="secret.json"

main() {
	{
		checkPrereqs &&
			getInput &&
			readSecrets &&
			createClient &&
			echo
		echo "::: Drupal OpenID Connect Setup Complete :::"
	} || {
		echo
		echo "!!! Drupal OpenID Connect Setup Failed !!!"
	}
}

time main "$@"
```

Run the script file

```sh
chmod +x drupal
./drupal
```

## 7. Test SSO

Open your drupal site and try to login