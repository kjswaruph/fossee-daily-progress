# Task

Date: 06/11/25

- 15:00 Logged in, navigated to `online_test/docker`, and ran `docker compose build` directly, bypassing the broken `invoke` script.
- 15:30 Fixed first build error (`requirements-py3.txt` not found) by copying all `requirements/*.txt` files into the `docker/Files/` directory  
- 15:15 Edited `Dockerfile_django` to correctly `ADD` and `pip3 install` the requirements from the `docker/Files/` directory.
- 16:45 Fixed second build error (`SyntaxError` on `pip install psutil`) by upgrading both Dockerfiles from `ubuntu:16.04` to `ubuntu:18.04`.
- 17:45 Fixed third build error (stuck on interactive timezone prompt) by adding `ENV DEBIAN_FRONTEND=noninteractive` to both Dockerfiles.
- 17:30 Fixed fourth build error (`RequiredDependencyException: jpeg`) by adding `libjpeg-dev`, `zlib1g-dev`, and `libfreetype6-dev` to the `apt-get` commands in both Dockerfiles.
- 18:15 The final `docker compose build` command completed successfully.
- 18:30 Started all services with `docker compose up -d`.
- 18:45 Ran `docker compose ps` and confirmed all three services (`yaksh_django`, `yak`