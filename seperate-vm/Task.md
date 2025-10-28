# Drupal and Keycloak Integration on Separate VMs

This guide walks you through setting up Drupal and Keycloak on separate Virtual Machines with Single Sign-On (SSO) integration.

## Prerequisites

- VirtualBox installed on your host machine
- Rocky Linux ISO image 
## 1. VM Setup and Network Configuration

### Create Virtual Machines

1. Download Rocky Linux ISO from the official website
2. Open VirtualBox and create two new virtual machines:
   - `drupal-vm` (for Drupal)
   - `keycloak-vm` (for Keycloak)

### Configure Host-Only Network

1. In VirtualBox, navigate to: File > Tools > Network Manager > Host-only Networks
2. Click Create to add a new host-only network
3. Select the network and click Properties
4. Under Adapter tab:
   - Check Configure Adapter Manually
5. Under DHCP Server tab:
   - Uncheck "Enable Server"
6. Click Apply

### Configure Network Adapters for Both VMs

For each VM, open Settings > Network:

- Adapter 1:
  - Enable Network Adapter: 
  - Attached to: NAT
  
- Adapter 2:
  - Enable Network Adapter: 
  - Attached to: Host-only Adapter
  - Name: Select the host-only network created earlier

Save and exit

### Assign Static IP Addresses

Start both VMs and configure static IPs on the host-only network interface.

For Keycloak VM (assign `192.168.56.11`):
```sh
# List network connections
nmcli con show

# Configure static IP for host-only adapter (usually enp0s8)
nmcli connection modify enp0s8 ipv4.addresses 192.168.56.11/24
nmcli connection modify enp0s8 ipv4.method manual
nmcli connection modify enp0s8 ipv4.gateway ""
nmcli connection modify enp0s8 ipv4.dns "8.8.8.8"
nmcli connection up enp0s8

# Verify configuration
ip addr show enp0s8
```

For Drupal VM (assign `192.168.56.10`):
```sh
nmcli con show
nmcli connection modify enp0s8 ipv4.addresses 192.168.56.10/24
nmcli connection modify enp0s8 ipv4.method manual
nmcli connection modify enp0s8 ipv4.gateway ""
nmcli connection modify enp0s8 ipv4.dns "8.8.8.8"
nmcli connection up enp0s8
ip addr show enp0s8
```

Test Connectivity:
```sh
# From Keycloak VM
ping -c 3 192.168.56.10

# From Drupal VM
ping -c 3 192.168.56.11
```

## 2. Setup drupal and keycloak VM

Install keycloak and drupal by following this guide: https://github.com/kjswaruph/fossee-daily-progress/blob/main/drupal-site-on-vm/Task.md

Note: Do not create keycloak and drupal clients we will use scripts to create in next section.

## 3. Keycloak script

This script automates the creation of Keycloak clients for Drupal integration.

### Setup directory

On Keycloak VM

```
mkdir ~/keycloak
cd keycloak
```

### Create config files

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

Download and run the script

```sh
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/seperate-vm/keycloak
# Set proper permissions
cd ..
sudo chown -R keycloak:$USER keycloak/
sudo chmod -R g+w keycloak/

# Run the script
chmod +x keycloak
./keycloak
```

Fix permissions and run the script

```sh
cd ..
sudo chown -R keycloak:$USER keycloak/
sudo chmod -R g+w keycloak/
chmod +x keycloak
./keycloak
```

This will add your new client in client.json.

```json
[
  {
    "clientId": "openplc",
    "secret": "<secret>",
    "enabled": true,
    "publicClient": false,
    "protocol": "openid-connect",
    "clientAuthenticatorType": "client-secret",
    "standardFlowEnabled": true,
    "redirectUris": [
      "<drupal_redirect>"
    ]
  }, ...
]
```

## 4. Drupal script

This script configures Drupal's OpenID Connect settings to integrate with Keycloak.

### Directory setup

Create a dir and copy required files

```sh
mkdir drupal
cd drupal
```

Download the script file

```sh
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/seperate-vm/drupal
```

Run the script file

```sh
chmod +x drupal
./drupal -c <your_client_id> -s <your_client_secret>
```

### Drush Configuration 

This part of documentation is a explanation to all the configurations that are being made in **createClient()** function inside the **drupal** script.

The drush commands updates the openid_connect.settings.kecloak.yaml or database in Drupal configuration

| Command                                                                     | Description                                                                                                                                        |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `vendor/bin/drush config:set openid_connect.settings.keycloak enabled true` | Enables the Keycloak OpenID Connect provider in Drupal.                                                                                            |
| `settings.client_id '$CLIENT_ID'`                                           | Sets the Keycloak client ID that Drupal will use for authentication.                                                                               |
| `settings.client_secret '$CLIENT_SECRET'`                                   | Sets the secret key used to authenticate Drupal with Keycloak.                                                                                     |
| `settings.keycloak_base '$KEYCLOAK_BASE'`                                   | Defines the base URL of your Keycloak server, you may have provided when asked.                                                                    |
| `settings.keycloak_realm '$KEYCLOAK_REALM'`                                 | Specifies the Keycloak realm Drupal should connect to. (*eg. master*)                                                                              |
| `settings.userinfo_update_email false`                                      | Disables automatic updating of user email from Keycloak profile info.                                                                              |
| `settings.keycloak_groups.enabled false`                                    | Disables Keycloak group synchronization.                                                                                                           |
| `settings.keycloak_groups.claim_name groups`                                | (If enabled) would specify which claim contains group data.                                                                                        |
| `settings.keycloak_groups.split_groups false`                               | Prevents splitting nested group names.                                                                                                             |
| `settings.keycloak_groups.split_groups_limit '0'`                           | Sets no limit for group splitting (not active since disabled).                                                                                     |
| `settings.keycloak_groups.rules []`                                         | No custom group mapping rules are defined.                                                                                                         |
| `settings.keycloak_sso true`                                                | Enables single sign-on (SSO) If a user is already logged into Keycloak, they will be automatically logged into Drupal without seeing a login form. |
| `settings.keycloak_sign_out true`                                           | Enables Keycloak-based single sign-out (log out from both systems).                                                                                |
| `settings.check_session.enabled false`                                      | Disables session checking via Keycloak iframe (reduces background polling).                                                                        |
| `settings.check_session.interval null`                                      | No periodic session check interval set.                                                                                                            |
| `settings.redirect_url ''`                                                  | No specific redirect URL after login/logout (default Drupal behavior).                                                                             |
| `settings.keycloak_i18n_enabled false`                                      | Disables Keycloak internationalization (language-specific logins).                                                                                 |
| `vendor/bin/drush cr`                                                       | Clears Drupal’s cache to apply all configuration changes.                                                                                          |
| `vendor/bin/drush config:get openid_connect.settings.keycloak`              | Prints the final Keycloak OpenID Connect configuration for verification.                                                                           |

## 5. Test SSO

### Verify Configuration

1. **Check Keycloak:**
   - Login to Keycloak admin console: `http://192.168.56.11:8080`
   - Navigate to your realm > Clients
   - Verify the Drupal client exists and is enabled

2. **Check Drupal:**
   - Navigate to: Configuration > Web > OpenID Connect
   - Verify Keycloak client is enabled and configured

### Test Login Flow

Open `http://192.168.56.10:8081/user/login` and check if it redirects to keycloak admin console 


