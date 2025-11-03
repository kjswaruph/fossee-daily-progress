# SSO Automation Scripts

This guide provides documentation for automation scripts that streamline Keycloak and Drupal SSO integration.

## Overview

The automation scripts consist of:
1. keycloak_client_setup - Automates Keycloak client creation and configuration
2. drupal_keycloak_sso - Automates Drupal OpenID Connect module installation and SSO setup

These scripts work together to establish Single Sign-On between Keycloak and Drupal installations.

## 1. Keycloak Client Setup Script

### Overview

The `keycloak` script automates the process of creating a Keycloak client for Drupal SSO integration using kcadm.sh script file.

### Setup Directory

```sh
mkdir ~/keycloak-automation
cd ~/keycloak-automation
```

### Create Configuration Files

Create two JSON configuration files:

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
[]
```

### Download the Script

```sh
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/sso-automation/keycloak_client_setup
chmod +x keycloak_client_setup
```

### Run the script

```sh
./keycloak_client_setup
```

## 2. Drupal SSO Setup Script

### Overview

The `drupal_keycloak_sso` script automates Drupal configuration for Keycloak SSO integration. It handles:
- OpenID Connect and Keycloak module installation
- Drupal user creation with administrator role
- OpenID Connect configuration via Drush

### Download the Script

```sh
cd /var/www/html
wget https://raw.githubusercontent.com/kjswaruph/fossee-daily-progress/refs/heads/main/sso-automation/drupal_keycloak_sso
chmod +x drupal_keycloak_sso
```
### Run the Script

```sh
./drupal_keycloak_sso \
  --client-id <your-client-id> \
  --client-secret <your-client-secret> \
  --base <keycloak-base-url> \
  --realm <keycloak-realm> \
  --email <user-email> \  # should be same as keycloak user email
  --drupal-user <drupal-username> \
  --drupal-password <drupal-password> \
  --user <system-user> # use user which has perms for drupal folder
```

### Verify Configuration

After successful execution, verify the configuration:

```sh
vendor/bin/drush config:get openid_connect.settings.keycloak
```

Expected output:
```yaml
enabled: true
settings:
  client_id: <your-client-id>
  client_secret: <your-client-secret>
  keycloak_base: <keycloak-url>
  keycloak_realm: <realm-name>
  keycloak_sso: true
```


## 3. Complete Integration Workflow

### Step 1: Setup Keycloak Client

On the Keycloak server:

```sh
cd ~/keycloak-automation
./keycloak
```

Enter the required information when prompted. Note the generated client secret from `client.json`.

### Step 2: Configure Drupal SSO

Inside your Drupal site folder:

```sh
cd /var/www/html
./drupal_keycloak_sso \
  --client-id <client-id-from-step-1> \
  --client-secret <secret-from-step-1> \
  --base <keycloak-url> \
  --realm <realm-name> \
  --email <admin-email> \
  --drupal-user <username> \
  --drupal-password <password> \
  --user <system-user>
```

### Step 3: Test SSO Integration

1. Navigate to Drupal Login: 
   ```
   http://<drupal-url>/user/login
   ```

2. Test SSO Flow:
   - You should be redirected to Keycloak login page
   - After successful login, redirected back to Drupal

3. Verify User Account:
   - Check that user is logged into Drupal
   - Verify user has administrator role 

4. Test Single Logout:
   - Log out from Drupal
   - Verify logged out from Keycloak as well
