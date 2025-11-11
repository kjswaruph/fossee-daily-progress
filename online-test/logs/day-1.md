# Task

Date: 05/11/25

- 13:00 Logged in to Rocky Linux
- 13:10 Installed Git, and cloned the `online_test` repository.   
- 13:30 Decided to attempt a direct installation using Miniconda, as recommended by the documentation.
- 13:45 Created a Python 3.8 environment, but the `pip install` failed repeatedly due to package compilation errors and dependency conflicts.
- 15:30 Abandoned the direct installation, concluding the old dependencies were incompatible with the modern OS.
- 15:45 Pivoted to the Docker-based deployment.
- 16:00 Installed Docker Engine and Docker Compose, started the service, and added the user to the `docker` group (requiring a re-login).
- 16:45 Attempted to build from the `docker/` directory using `invoke build`.
- 17:00 The build script failed with host Python errors (`ModuleNotFoundError: No module named 'lexicon'`).
- 17:30 Tried to fix the host script in a venv, but this led to deeper Python 3.12 incompatibility errors (`ModuleNotFoundError: No module named 'imp'`).
- 18:00 Concluded the `invoke` script was unusable and decided to use `docker compose` commands directly tomorrow. 
- 18:10 Logged off.