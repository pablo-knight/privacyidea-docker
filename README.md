[![Docker](https://github.com/gpappsoft/privacyidea-docker/actions/workflows/docker-publish.yml/badge.svg)](https://github.com/gpappsoft/privacyidea-docker/actions/workflows/docker-publish.yml)

# privacyIDEA-docker

Simply deploy and run a privacyIDEA instance in a container environment. 

## Overview 
[privacyIDEA](https://github.com/privacyidea/privacyidea) is an open solution for strong two-factor authentication like OTP tokens, SMS, smartphones or SSH keys. 

This project is a complete build environment under linux to build and run privacyIDEA in a container environment. It uses the official [python](https://pypi.org/project/privacyIDEA/) image and the official [privacyIDEA-Project](https://pypi.org/project/privacyIDEA/)  from PyPi. The image uses [gunicorn](https://gunicorn.org/) to run the app. 

**The main goals of this project are to:**
- Give an idea how to run privacyIDEA in a container.
- Build and run the container image, simple and fast.
- Deploy different versions and/or stages (e.g. production, staging, devel ...) with the same or different configuration on the same host.
- Easy deploy a "full-stack" (e.g. privacyIDEA, radius, database and reverse proxy) with docker compose.
- Keep the container's image simple and slim.
- Build images with no changes to the original privacyIDEA code and few additional scripts as possible inside the image to run the container. 

**What this project is not:**
- A fully tested and "production-ready" installation of privacyIDEA for *your* container environment.
- A possible way to ignore the [privacyIDEA Documentation](https://privacyidea.readthedocs.io/en/latest/)
- A guide on how to use docker
- The "one and only" or "best" method to run privacyIDEA in a container.
- Finished ;)

> [!Note] 
> The image does **not include** a reverse proxy or a database backend. Running the default image as a standalone container uses gunicorn and a sqlite database. This is not suitable for a production environment.
>
> A more 'real-world' scenario, which is often used, is described in the [Compose a privacyIDEA stack](#compose-a-privacyidea-stack) section.
>
> Also check the [Security considerations](#security-considerations) before running the image or stack in a production environment.

While decoupling the privacyIDEA image from dependencies like Nginx, Apache or database vendors, it is possible to run privacyIDEA with your favorite components.

If you prefer another approach or would like to test another solution, take a look at the [Khalibre / privacyidea-docker](https://github.com/Khalibre/privacyidea-docker) project. This project might be a more suitable solution for your needs.

### Repository 

| Directory | Description |
|-----------|-------------|
| *conf* | contains *pi.cfg* and *logging.cfg* files which is included in the image build process.|
| *secrets* | contains the secrets for docker compose - this project tries to avoid using env-vars for sensitive data (passwords)|
| *scripts* | contains custom scripts for the script-handler. Will be mounted into the container when compose a stack. Scripts must be executable (chmod +x)|
| *examples* | contains different example-environment files for a whole stack via docker compose|
|*ssl* | contains ssl certificates for the reverse-proxy. Replace it with your own certificate and key file. Use PEM-Format without a passphrase. \*.pfx is not supported. Name must be *pi.pem* and *pi.key* |
|*templates*| contains files used for different services (nginx, radius ...)|

### Images
Sample images from this project can be found here: 
| registry | repository |
|----------|------------|
| [docker.io](https://hub.docker.com/r/gpappsoft/privacyidea-docker)|```docker pull docker.io/gpappsoft/privacyidea-docker:latest```
| [ghcr.io](https://github.com/gpappsoft/privacyidea-docker/pkgs/container/privacyidea-docker)| ```docker pull ghcr.io/gpappsoft/privacyidea-docker:latest```|

> [!Note] 
> ```latest``` tagged image is maybe a pre- or development-release. Please use always a release number (like ```3.9.1```) 

### Prerequisites and requirements

- Installed a container runtime engine (docker / podman).
- Installed [BuildKit](https://docs.docker.com/build/buildkit/), [buildx](https://github.com/docker/buildx) and [Compose V2](https://docs.docker.com/compose/install/linux/) (docker-compose-v2) components
- The repository is tested with versions listed in [COMPAT.md](COMPAT.md)
- [Podman](https://podman.io) is partially supported. **Please refer to [PODMAN.md](PODMAN.md) for more details.**

## Quickstart

#### Quick & Dirty

```
docker pull docker.io/gpappsoft/privacyidea-docker:3.9.1
docker run -d --name privacyidea-docker\
            -e PI_REGISTRY_CLASS=null\
            -e PI_BOOTSTRAP=true\
            -e PI_PEPPER=superSecret\
            -e PI_SECRET=superSecret\
			-v pi-pilog:/var/log/privacyidea:rw,Z\
			-v pi-piconfig:/etc/privacyidea:rw,Z\
			-p 8080:8080\
			gpappsoft/privacyidea-docker:3.9.1
```
Web-UI: http://localhost:8080

user/password: **admin** / **admin**

#### Preferred way

To build and run a simple local privacyIDEA container (which can run standalone with sqlite):

```
git clone https://github.com/gpappsoft/privacyidea-docker.git
cd privacyidea-docker
make cert secret build run
....
```

Answer the following question with a ```y```:
```
Warning! Overwrite ALL SECRETS  in ./secrets directory: Are you sure? [y/N] y
```

##### Accessing the Web-UI:
Use https://localhost:8080

Default admin username: **admin** 

Default admin password is stored in *./secrets/pi_admin_pass*


## Build images

You can use *Makefile* targets to build different images with different privacyIDEA versions.

#### Build a specific privacyIDEA version
```
make build PI_VERSION=3.8.1
```

#### Push to a registry
Use ```make push [REGISTRY=<registry>]```to tag and push the image[^1]
##### Example 
Push image to local registry on port 5000[^2]

```
make push REGISTRY=localhost:5000
``` 

#### Remove the container:
```
make clean
```
You can start the container with the same database (sqlite) and configuration and use ```make run``` again without bootstrapping the instance.
#### Remove the container including volumes:
```
make distclean
```
&#9432; This will wipe the whole container including the volumes!

### Overview targets

| target | optional ARGS | description | example
---------|----------|---|---------
| ```build ``` | ```PI_VERSION```<br> ```IMAGE_NAME```|Build an image. Optional: specify the version and image name| ```make build PI_VERSION=3.9.1```|
| ```push``` | ```REGISTRY```|Tag and push the image to the registry. Optional: specify the registry URI. Defaults to *localhost:5000*| ```make push REGISTRY=github.com/gpappsoft/privacyidea-docker/pkgs/container/privacyidea```|
| ```run``` |  ```PORT``` <br> ```TAG```  |Run a standalone container with gunicorn and sqlite. Optional: specify the prefix tag of the container name and listen port. Defaults to *pi* and port *8080*| ```make run TAG=prod PORT=8888```|
| ```secret``` | |Generate and **overwrite** secrets in *./secrets* | ```make secret```|
| ```cert``` | |Generate a self-signed certificate for the reverse proxy container in *./ssl*. and **overwrite** the existing one | ```make secret```|
| ```stack``` |```TAG```| Run a whole stack with the environment file *examples/application-prod.env*  | ```make stack TAG=prod```|
| ```clean``` |```TAG```| Remove the container and network without removing the named volumes. Optional: change prefix tag of the container name. Defaults to *pi* | ```make clean TAG=prod```|
| ```distclean``` |```TAG```| Remove the container, network **and named volumes**. Optional: change prefix tag of the container name. Defaults to *pi* | ```make distclean TAG=prod```|

> [!Important] 
> Using the image as a standalone container is not production ready. For a more like 'production ready' instance, please refer to the next section.

## Compose a privacyIDEA stack

By using docker compose you can easily deploy a customized privacyIDEA instance, including Nginx as a reverse-proxy and MariaDB as a database backend.

With the use of different environment files for different full-stacks,  you can deploy and run multiple stacks at the same time on different ports. 

```mermaid
graph TD;
  a2("--env-file=examples/application-dev.env");
  w3(https://localhost:8444);
  w4(RADIUS  1813);
  subgraph s3 [stack 2];
       r2(RADIUS);
       n4(NGINX);
       n5(privacyIDEA);
       n6(MariaDB);
  end;
  a1("--env-file=examples/application-prod.env");
  w1(https://localhost:8443);
  w2(RADIUS 1812);
  subgraph s1 [stack 1];
       r1(RADIUS);
       n1(NGINX);
       n2(privacyIDEA);
       n3(MariaDB);
  end;

  a1~~~w1;
  a1~~~w2;
  a2~~~w3;
  a2~~~w4;

  w3<-->n4<-- reverse proxy -->n5(privacyIDEA\nwith gunicorn on port 8080)<-->n6
  w4<-->r2<-- privacyIDEA Radius -->n5(privacyIDEA\nwith gunicorn on port 8080)       

  w1<-->n1<-- reverse proxy -->n2(privacyIDEA\nwith gunicorn on port 8080)<-->n3
  w2<-->r1<-- privacyIDEA Radius -->n2(privacyIDEA\nwith gunicorn on port 8080)
  
  classDef plain font-size:12px,fill:#ddd,stroke:#fff,stroke-width:4px,color:#000;
  classDef heading font-size:12px,fill:#9db668,stroke:#fff,stroke-width:4px,color:#fff;
  classDef title font-size:16px,color:#000,background-color:#9db668,stroke:#ffff;
  classDef docker fill:#265a88,stroke:#fff,stroke-width:4px,color:#fff;
  classDef cluster fill:#fff,stroke:#888,stroke-width:2px,color:#265a88;
  class w1,w2,w3,w4 plain;
  class n1,n2,n3,n4,n5,n6,r1,r2,t1 docker;
  class s1,s2,s3,s4 title;
  class a1,a2 heading;
```

Find example .env files in the *examples* directory.

> [!Note]
> The RADIUS container is not included in this repository at the moment. The freeradius container image, including the [privacyIDEA RADIUS-plugin](https://github.com/privacyidea/FreeRADIUS), will be released soon.

---
### Examples:
To use this example, you have to run a local registry[^2] and already pushed an image into it with ```make push``` command.

Run a stack with project the name *prod* and environment variables files from *examples/application-prod.env*

```
  $ make cert secret  #run only once to generate certificate and secrets
  $ PI_BOOTSTRAP=true docker compose --env-file=examples/application-prod.env -p prod up
```
Alternative you can run the ```make```target:

```
make cert secret stack
```

Shutdown the stack with the project name *prod* and **remove** all resources (container,networks, etc.) except the volumes.

```
docker compose -p prod down 
```

You can start the stack in the background with console detached using the **-d** parameter.

```
  $ PI_BOOTSTRAP=true docker compose --env-file=examples/application-prod.env -p prod up -d
```

Full example including build with  ```make```targets:
```
make cert secret build push stack PI_VERSION=3.9.1 TAG=pidev
```
---
Now you can deploy additional containers like OpenLDAP for user realms or Owncloud as a client to test 2FA authentication. 

Have fun!

> [!IMPORTANT] 
>- Volumes will not be deleted. 
>- Currently same secrets from *./secrets/* are used for every stack. Future releases will change this behavior.
>- Delete the */etc/privacyidea/BOOTSTRAP* file **inside* the privacyIDEA container if you want to bootstrap again. This will not delete an existing database!
>- Compose takes some additional time (~1 minute) because of health-checks.


## Environment Variables

### privacyIDEA
| Variable | Default | Description
|-----|---------|-------------
```ENVIRONMENT``` | examples/application-prod.env | Used to set the correct environment file (env_file) in the docker compose, which is used by the container. Use a relative filename here.
```PI_VERSION```|3.9.1| Set the used image version
```PI_BOOTSTRAP``` | false | Set to ```true``` to create database tables on the first start of the container. If you need to re-run, then you have to delete the */etc/privacyidea/BOOTSTRAP* file inside the container. 
```PI_UPDATE```| false | Set to ```true``` to run the database schema upgrade script in case of a new privacyIDEA version. 
```PI_PASSWORD```|admin| Don't use this in productive environments. Use secrets with docker compose / docker swarm instead. See [Security considerations](#security-considerations) for more information.
```PI_ADMIN```|admin| login name of the initial administrator
```PI_PORT```|8080| Port used by gunicorn. Don't use this directly in productive environments. Use a reverse proxy..
```PI_LOGLEVEL```|INFO| Log level in uppercase (DEBUG, INFO, WARNING, ect.). ```docker log``` is always ```INFO``` level because of security. See *conf/logging.cfg* for more details
```SUPERUSER_REALM```|"admin,helpdesk"| Admin realms, which can be used for policies in privacyIDEA. Comma separated list. See the privacyIDEA documentation for more information.
```PI_SQLALCHEMY_ENGINE_OPTIONS```| False | Set pool_pre_ping option. Set to ```True``` for DB clusters (like Galera).
```PI_PEPPER``` |superSecret | Used for ```PI_PEPPER``` in pi.cfg. Use `make secrets` to generate new secrets. See [Security considerations](#security-considerations) for more information.
```SECRET_KEY``` | superSecret | Used for ```SECRET_KEY``` in pi.cfg. Use `make secrets` to generate new secrets. See [Security considerations](#security-considerations) for more information.

### DB connection parameters
| Variable | Description
|-----|-------------
```DB_HOST```| Database host
```DB_PORT```| Database port
```DB_NAME```| Database name
```DB_USER```| Database user
```DB_API```| Database driver (e.g. ```mysql+pymysql```)
```DB_EXTRA_PARAM```| Extra parameter (e.g. ```"?charset=utf8"```). Will be appended to the SQLAlchemy URI (see pi.cfg)

### Reverse proxy parameters
| Variable | Default | Description
|-----|---------|-------------
```PROXY_PORT```| 8443 | Exposed HTTPS port
```PROXY_SERVERNAME```| localhost | Set the reverse-proxy server name. Should be the common name used in the certificate.

### Secrets used by docker compose located in *secrets/*
| Filename | Default | Description
|-----|---------|-------------
| *db_password*| superSecret | The database password 
| *pi_admin_pass*| admin | The password for the initial admin
| *pi_pepper*| superSecret | The PI_PEPPER secret for pi.cfg
| *pi_secret*| superSecret | The SECRET_KEY secret for pi.cfg

## Security considerations

#### Secrets 
The current concept of using secrets with files in *docker-compose.yaml* is a first approach to reducing the risk of using secrets with environment variables. This may change in future versions.

Different stacks always use the **same** secrets. 

## Known Bugs

- unkown

## Frequently Asked Questions

#### Why are not all pi.cfg parameters available as environment variables?
- I only included the most essential and often-used parameters. You can add more variables to the *conf/pi.conf* file and build your own image.

#### Why not include the scripts for the script-handler in the image?
- The image should only contain a bare privacyIDEA without custom data. You can add them to the *Dockerfile* file and build your own image.

#### How can I rotate the audit log?

- Simply use a cron job on the host system with docker exec and the pi-manage command: 
```
docker exec -it pi-privacyidea-1 pi-manage audit rotate_audit --age 90
```
#### How can i access the logs?

- Use docker log or use the logfile:  
```
docker logs pi-privacyidea-1 
```
```
docker exec -it pi-privacyidea-1 cat /var/log/privacyidea/privacyidea.log
```
#### How can I update the container to a new privacyIDEA version?
- Build a new image, make a push and pull. Re-create the container with ```PI_UPDATE=true```. This will run the schema update script to update the database.

#### Can I import a privacyIDEA database dump into the database container from the stack?
- Yes, by providing the sql dump to the db container. Please refer to the *"Initializing the database contents"* section from the official [MariaDB docker documentation](https://hub.docker.com/_/mariadb).

#### Help! ```make build``` does not work, how can i fix it?

- Check the [Prerequisites and requirements](#prerequisites-and-requirements). Often there is a missing plugin (buildx, compose) - install the plugins and try again:
```
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
curl -SL https://github.com/docker/buildx/releases/download/v0.12.0/buildx-v0.12.0.linux-amd64 -o $DOCKER_CONFIG/cli-plugins/docker-buildx
chmod +x $DOCKER_CONFIG/cli-plugins/docker-{buildx,compose}
```

#### Help! Stack is not starting because of an error like ```permission denied```. How can I fix it?

Check selinux and change the permissions like:
```
chcon -R -t container_file_t PATHTOHOSTDIR
```
```PATHTOHOSTDIR``` should point to the privacyidea-docker folder.

#### Help! ```Error response from daemon: invalid mount config for type "bind": bind source path does not exist: ...```, how can I fix it?

Check if the required certificates (*pi.key* / *pi.pem*) exists in ssl/

#### Help! ```make push```does not work with my local registry, how can I fix it?

- Maybe you try to use ssl: Use the insecure option in your */etc/containers/registries.conf*: 
   ```
   [[registry]]
   prefix="localhost"
   location="localhost:5000"
   insecure=true
   ```
#### How can I create a backup of my data?

- The pi-manage backup command is not working. You have to dump the database manually. For the example stack, use the db container: 

```
docker exec -it pi-db-1 mariadb-dump -u pi -psuperSecret pi
```
- enckey/config, certificates and logs stored in the named volumes *_piconfig *_pilog

## Roadmap

#### Customization and scripts

Support for [customization](https://privacyidea.readthedocs.io/en/latest/faq/customization.html) comming soon.

#### Radius

A first release of a Freeradius image with the privacyIDEA-RADIUS plugin is in progress.

Any feedback are welcome! 

# Disclaimer

This project is my private project doing in my free time. This project is not from the NetKnights company. The project uses the open-source version of privacyIDEA. There is no official support from NetKnights for this project.

[^1]: If you push to external registries, you may have to login first.
[^2]: You can run your own local registry with:\
   ``` docker  run -d -p 5000:5000 --name registry registry:2.7 ``` 
   
   