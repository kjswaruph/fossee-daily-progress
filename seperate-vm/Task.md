# Drupal and Keycloak on seperate VMs

## 1. Setup VM for both Drupal and Keycloak

- Install Rocky Linux ISO image from Rocky linux official site
- Open VirtualBox and create two new machines for openplc and keycloak
- After creating machines, Navigate to File > Tools > Networks > Host-only Networks > Create
- Check Configure Adapter Manually and make sure DHCP server is disabled
- Now open created machine's setting. Under Network, set
  - Adapter 1
    - Attached to: NAT
  - Adapter 2
    - Turn on Enable Network Adapter
    - Attached to: Host-only adapter
      Leave rest as it is
  - Save
- Start both the VMs
- Run these following commands

```sh
nmcli con show
# For keycloak vm 192.168.56.11/24 and for drupal vm 192.168.56.10/24
nmcli connection modify enp0s8 ipv4.addresses 192.168.56.10/24
nmcli connection modify enp0s8 ipv4.method manual
nmcli connection modify enp0s8 ipv4.gateway ""
nmcli connection modify enp0s8 ipv4.dns "8.8.8.8"
nmcli connection up enp0s8
nmcli connection show
```

## 2. Setup drupal and keycloak VM

Install keycloak and drupal on respective VMs

## 3. Keycloak script

Create dir and set permissions

```
mkdir ~/keycloak
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

Add the script

```sh
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/seperate-vm/keycloak
```

Fix permissions and run the script

```sh
sudo chown -R keycloak:$USER keycloak/
chmod +x keycloak
./keycloak
```

This will generate secret.json file copy the contents of the file

## 4. Drupal script

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

Add script file

```sh
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/seperate-vm/drupal
```

Run the script file

```sh
chmod +x drupal
./drupal
```

## 5. Test SSO

Open your drupal site and try to login
