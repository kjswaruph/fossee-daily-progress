# Integrate Keycloak SSO for online-test application

This guide walks you through integrating Keycloak Single Sign-On (SSO) with the FOSSEE Online Test application using Docker and Mozilla Django OIDC on Rocky Linux 10.

## 1. Install Docker and Docker Compose

```sh
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

## 2. Clone and Setup Online Test Application

```sh
cd ~
git clone https://github.com/FOSSEE/online_test.git
cd online_test
```

### Prepare Docker Environment

Copy requirement files to Docker build context:

```sh
cp requirements/*.txt docker/Files/
```

### Configure Environment Variables

Create environment configuration from sample:

```sh
cp .sampleenv .env
```

Edit `.env` file and uncomment all variables:

## 3. Setup dockerfile

Add the Mozilla Django OIDC library to the requirements file:

```sh
echo "mozilla_django_oidc" >> docker/Files/requirements-common.txt
```

Edit `Dockerfile_django`, update ubuntu image to 20.04 and add ENV DEBIAN_FRONTEND variable

```Dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive
```

Replace `RUN cd /Sites/online_test && pip3 install -r /tmp/requirements-py3.txt ` with:

```
RUN cd /Sites/online_test && \
    pip3 install -r /tmp/requirements-common.txt && \
    pip3 install -r /tmp/requirements-production.txt
```

Edit `Dockerfile_codeserver`, update ubuntu image to 20.04 and add env DEBIAN_FRONTEND

```Dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive
```

## 4. Keycloak Client Configuration

Create django client

- Open keycloak admin console and go to Manage realms and switch to your required realm
- Manage > Clients > Create client
- Client ID: online-test
- Client Authentication: on
- Valid redirect URIs: https://your_django_domain/oidc/callback/
- Save and copy the Client Secret from the credentials tab

## 5. Configure Django Application

### Update Django Settings

Edit `online_test/settings.py` file:

```python
INSTALLED_APPS = [
    # ... existing apps ...
    'mozilla_django_oidc',  # Add this line
]

AUTHENTICATION_BACKENDS = (
    'mozilla_django_oidc.auth.OIDCAuthenticationBackend', # Add this
    'django.contrib.auth.backends.ModelBackend',
)


# Add these lines and replace your_client_secret, your_keycloak_realm, your_keycloak_url
OIDC_RP_CLIENT_ID = "online-test"
OIDC_RP_CLIENT_SECRET = "your_client_secret"

OIDC_OP_AUTHORIZATION_ENDPOINT = "http://your_keycloak_url/realms/your_keycloak_realm/protocol/openid-connect/auth"
OIDC_OP_TOKEN_ENDPOINT = "http://your_keycloak_url/realms/your_keycloak_realm/protocol/openid-connect/token"
OIDC_OP_USER_ENDPOINT = "http://your_keycloak_url/realms/your_keycloak_realm/protocol/openid-connect/userinfo"
OIDC_RP_SIGN_ALGO = "RS256"
OIDC_OP_JWKS_ENDPOINT = "http://your_keycloak_url/realms/your_keycloak_realm/protocol/openid-connect/certs"
OIDC_OP_LOGOUT_ENDPOINT = "http://your_keycloak_url/realms/your_keycloak_realm/protocol/openid-connect/logout"
OIDC_STORE_ID_TOKEN = True

```

Edit `online_test/urls.py`

```python
from django.conf.urls import url, include
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^', include('yaksh.urls')),
    url(r'^oidc/', include('mozilla_django_oidc.urls')),  # Add this line
]
```

**OIDC URL Endpoints:**

- `/oidc/authenticate/` - Initiates login flow
- `/oidc/callback/` - Handles callback from Keycloak
- `/oidc/logout/` - Logs out user

Add Keycloak login button to the login page:

Find the login form section and add the Keycloak button:

```html
<a href="{% url 'oidc_authentication_init' %}" class="btn btn-neutral btn-icon">
  <span class="btn-inner--icon">
    <i class="fa fa-key"></i>
  </span>
  <span class="btn-inner--text">Keycloak</span>
</a>
```

## 6. Build and run with docker

### Build Docker Images

Navigate to the docker directory and build:

```sh
cd docker
docker compose build
docker compose up -d
docker compose ps
```

### Initialize Database

Run Django migrations and load demo data:

```sh
# Create migration for notifications plugin
docker compose exec yaksh-django python3 manage.py makemigrations notifications_plugin

# Apply all migrations
docker compose exec yaksh-django python3 manage.py migrate

# Load demo fixtures (sample data)
docker compose exec yaksh-django python3 manage.py loaddata demo_fixtures.json

# Restart Django service to apply changes
docker compose restart yaksh-django
```

## 7. Configure Firewall

Allow external access to the application:

```sh
# Add port 8000 to firewall
sudo firewall-cmd --add-port=8000/tcp --permanent
sudo firewall-cmd --reload
```

Verify firewall rules:

```sh
sudo firewall-cmd --list-ports
```

You should see `8000/tcp` in the output.

## 8. Test SSO Integration

### Access the Application

Open your browser and navigate to:

```
http://<server-ip>:8000
```

### Test Login Flow

1. Navigate to Login Page `http://<your_server_url:8000`
2. Click "Login with Keycloak" button
3. Redirect to Keycloak:
   - You should be redirected to Keycloak login page
   - Enter Keycloak credentials
4. Successful Authentication:
   - After login, redirected back to Online Test application
   - You should be logged in automatically
   - User account created in Django if first login

### Verify User Account

Check that user was created in Django:

```sh
docker compose exec yaksh-django python3 manage.py shell
```

In the Django shell:

```python
from django.contrib.auth.models import User
users = User.objects.all()
print(users)
exit()
```

### Test Logout

1. Click logout in the application
2. Verify you're logged out from both Django and Keycloak
3. Test automatic re-login if Keycloak session is still active
