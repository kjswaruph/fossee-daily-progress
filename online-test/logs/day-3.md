# Task

Date: 07/11/25

- 14:00 Logged in and fixed `DisallowedHost` error by hardcoding the IP in `online_test/settings.py`.
- 14:30 Restarted container and verified site access.
- 15:15 Began Keycloak SSO integration using the built-in `social-auth-app-django` library.
- 15:45 Added Keycloak button to `login.html` pointing to `social:begin 'keycloak'`.
- 16:00 Rebuilt container and tested login. Repeatedly failed with a persistent `InvalidAlgorithmError` even after adding `python-jose` and `cryptography`.
- 16:30 Decided to abandon `social-auth` as it was not working with this project version.
- 16:45 Switched to `mozilla-django-oidc`:
    - Removed all `social-auth` Keycloak settings from `settings.py`.
    - Added `mozilla-django-oidc` to `docker/Files/requirements-common.txt`.
    - Rebuilt the `yaksh-django` container.
- 17:30 Updated Keycloak client's "Valid Redirect URIs" to `http://192.168.56.12:8000/oidc/callback/`.
- 17:45 Restarted the `yaksh-django` container.
- 18:00 Tested the new `mozilla-django-oidc` login flow. Keycloak SSO login was successful.
- 18:15 Logged off.