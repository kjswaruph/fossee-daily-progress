# Run openplc locally

### 1. Dir Setup

- Create a dir for openplc
- Add init_sites script
- Create sites.json with content

```json
[
  {
    "SITE_NAME": "openplc",
    "PORT": 8081,
    "REPO": "https://github.com/FOSSEE/openplc_docker_image"
  }
]
```
- Add openplc.sql file

### 2. Install necessary packages

Install mysql, podman, slirp4netns, jq 

### 3. Run init_sites script

```sh
chmod +x init_sites
./init_sites
```

### 4. Enter site configuration

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

Summary of entered information:
Site Name        : openplc
Database Name    : openplcdb_2025_10_10
Database Username: openplcuser
SQL Dump File    : ./openplc.sql
Image Name       : openplc_image
Container Name   : openplc_container
Service Name     : openplc_persist
Drupal Version   : 10

Is this information correct? (y/n): y
Confirmed. Continuing...
[sudo] password for root: 

```

### 5. Drop in podman container

```sh
podman exec -it openplc_container bash
chmod +x vendor/bin/drush
vendor/bin/drush cr # Reload cache
vendor/bin/drush user-create {your_username} --mail="{your_email}" --password="{your_password}"
vendor/bin/drush user-add-role "administrator" {your_username}
```

### 6. Open openplc site

- Open http://localhost:8081/ 
- Login into the site http://localhost:8081/user/login with your credentials
