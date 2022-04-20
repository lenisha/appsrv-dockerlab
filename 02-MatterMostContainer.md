# Deploying and Running MatterMost Container on App Service 

## 1. Mattermost on Docker

Instructions for installing mattermost on docker are at their docs site: [Install Mattermost via Docker](https://docs.mattermost.com/install/install-docker.html)

To run the server it uses docker compose to manage multiple containers - postgres, nginx and mattermost.

To run it in Azure we will externalize Postgres and use Azure Postgres server (fully managed by MSFT) and we do not need Nginx reverse proxy, since Azure App Service provides the required functionality such as TLS termination.

Mattermost server also uses storage volumes to store indexes and configuration etc. On Azure we could use Azure Storage Account Files and map them to App Service.


## 2. Setup Azure Prereqs

- Create `Azure Database for PostgreSQL flexible server` 
![app](./docs/postgres.jpg)

- Create `Azure Storage Account` and file share to map to container

![app](./docs/fileshare.jpg)

- Create ACR (if not yet created previously)


## 3. Import mattermot container to ACR

- Login to azure using `az login`

- Import Mattermost container to ACR using cli. More details on the command in [ACR Import Images](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-import-images?tabs=azure-cli, where `clcontainers` is name of ACR
)

```sh
az acr import -n clcontainers --source  docker.io/mattermost/mattermost-enterprise-edition:6.3 --image mattermost/mattermost-enterprise-edition:6.3
```


## 4. Create Azure App Service

- Create WebApp with Setting to Linux and Docker and upload `docker-appservice.yml` from this repo

![app](./docs/appsvcdocker.jpg)

![app](./docs/compose.jpg)

- Edit ``docker-appservice.yml`  to replace values for `image:` to your ACR server and `MM_SERVICESETTINGS_SITEURL` to point to your appservice domain

```yaml
version: "2.4"
services:
  mattermost:

    container_name: mattermost
    ## REPLACE your image tag
    image: clcontainers.azurecr.io/mattermost/mattermost-enterprise-edition:6.3 
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    pids_limit: 200
    read_only: false
    tmpfs:
      - /tmp
    volumes:
      - /mattermost/config:/mattermost/config:rw
      - /mattermost/data:/mattermost/data:rw
      - /mattermost/logs:/mattermost/logs:rw
      - /mattermost/plugins:/mattermost/plugins:rw
      - /mattermost/client/plugins:/mattermost/client/plugins:rw
      - /mattermost/bleve-indexes:/mattermost/bleve-indexes:rw

    environment:
      # timezone inside container
      - TZ=UTC

      # necessary Mattermost options/variables (see env.example)
      - MM_SQLSETTINGS_DRIVERNAME=postgres
      - MM_SQLSETTINGS_DATASOURCE=${MM_SQLSETTINGS_DATASOURCE}
      # necessary for bleve
      - MM_BLEVESETTINGS_INDEXDIR=/mattermost/bleve-indexes
      # REPLACE with Domain name for App Service
      - MM_SERVICESETTINGS_SITEURL=https://mattermosten.azurewebsites.net

```

- Update AppService Compose YAML in Deployment Center

## 5. Update settings and create File Mappings

- As shown in Docker Compose YAML we need to set Connection string variable  `MM_SQLSETTINGS_DATASOURCE` to Azure Postgres connection string and `WEBSITES_PORT` to mattermost container exposed port. 

![app](./docs/appsettings.jpg)

See `appsettings.json` for example configuration in this repo.

- Create file mappings in App Service settings pointing to Azure Files in Storage Account and giving same names in share as in Compose manifest `/mattermost/config|data|logs`

![app](./docs/appsvcfileshare.jpg)

- Restart the app service and navigate to mattermost url

- For troubleshooting use `Log Stream`

## References
[Create a multi-container (preview) app in Web App for Containers](https://docs.microsoft.com/en-us/azure/app-service/tutorial-multi-container-app)

[Mounting Volumes on Azure Web App for Containers](https://www.codit.eu/blog/mounting-volumes-on-azure-web-app-for-containers/?country_sel=be)