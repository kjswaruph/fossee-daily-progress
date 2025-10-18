# Task: Keycloak automation

Date: 12/10/25

- 2:30 Logged in
- 2:40 Stop openplc_container and observe it gets removed due to --rm tag
- 2:50 Manually run container without --rm tag to keep it after stopping
- 3:00 Install openid_connect and keycloak modules using composer inside container
- 3:10 Enable modules using drush
- 3:20 Stop container and verify it persists (not removed)
- 3:30 Commit container changes using podman commit to save module installations
- 3:40 Verify image is updated with new changes
- 3:50 Stop openplc_persist service
- 4:00 Test persistence by running new container from updated image
- 4:10 Verify openid_connect and keycloak modules are available in new container
- 4:20 Successfully implemented persistent container changes
- 4:30 Logged off
