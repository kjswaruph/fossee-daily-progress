# Run openplc locally

## 1. Dir Setup

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
