# Task

Date: 10/10/25

- 2:30 Logged in
- 2:40 Download VirtualBox and RockyLinux image
- 3:00 Got error while installing VirtualBox
- 4:00 Fixed the error by installing all necessary packages
- 4:20 Successfully installed rockylinux machine on VB
- 4:30 Updated the packages of rockylinux
- 4:40 Install required dependencies
- 5:00 Setup project dir with files init_sites, sites.json, Dockerfile and openplc.sql
- 5:10 Run init_sites
- 5:30 Stuck at error while running init_sites due to connection issues
  At Docker build step 7:

```sh
A connection timeout was encountered. If you intend to run Composer without connecting to the internet, run the command again prefixed with COMPOSER_DISABLE_NETWORK=1 to make Composer run in offline mode.
The following exception probably indicates you have misconfigured DNS resolver(s)
In CurlDownloader.php line 372:
curl error 28 while downloading https://packages.drupal.org/8/packages.json
: Resolving timed out after 10000 milliseconds

Error: building at STEP "RUN composer update && curl -s https://raw.githubusercontent.com/Metadrop/drupal-fix-permissions-script/refs/heads/main/drupal_fix_
permissions.sh | bash -s -- -u=${ENU_USR}": while running runtime: exit status 1
Error: Failed to build Podman image openplc_image for site 'openplc'
!!! Operation Failed !!!
Error occurred. Check input and try again.
```

- 6:00 Logged off
