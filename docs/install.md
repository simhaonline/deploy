# LXD manager deployment

This is the repository which helps with the deployment of the lxd-manager. It is up to you to run all the required services on your own, but as it is rather complex, we help you with the following instructions.

## Requirements

The services are all dockerised and require a linux host with the following software

- [docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
- [docker-compose](https://docs.docker.com/compose/install/)

## Step-by-step guide

### Repo

Clone this repository:

    git clone https://github.com/lxd-manager/deploy.git

We suggest you create a branch with your actual configuration values.

### Web Proxy

This docker-compose enables a reverse proxy for handling TLS. If you do NOT want to use this or provide you own ingress service, please remove the `reverse-proxy` section from the services defined in `docker-compose.yml` and provide a mean to access the `nginx` service on port 80.

Otherwise create an external network with:

    docker network create web

Decide on the FQDN where you will be hosting the webservice and replace `service-fqdn` at the docker-compose.yml file `nginx` service in the traefik labels.

The proxy automatically generates Let's Encrypt TLS vertificates for the domain. For this, your email is required in the  `[certificatesResolvers.le.acme]` section of the `traefik.toml` file.

### LXD connection

The lxd api uses TLS client certificates as authentication mechanism. Therefore place a TLS key and (optionally self signed) certificate to `certs/lxd.key` and `certs/lxd.crt`.
You add new hosts through the api, where you supply a trust password which is used to establish the certificate at the new host.

    openssl req -x509 -newkey rsa:4096 -nodes -keyout certs/lxd.key -out certs/lxd.crt -days 3650

There is an option to provide the CA certificate the server is using via the LXD_CA_CERT environment variable. If the lxd server uses the default self signed certificate, you have to set the variable to "False"

### DNS

There is a DNS server which generates responses based on the containers available in the database. This requires you to set up a NS delegation of the subdomain `ct-subdomain.d.tld.` under which the containers will be named to a nameserver `ns-fqdn.d.tld.` which points to your deployment.
Replace `server-ip` in the ports section with the external IP where the DNS server should listen. (If it is an IPv6 address omit \[ and \])

#### Example

In your DNS zone set:

    ct.example.org.     1800	IN	NS	lxd-ui.example.org.
    lxd-ui.example.org. 1800	IN	A	1.2.3.4

and in `docker-compose.yml` set

- `ns-fqdn.d.tld.` = `lxd-ui.example.org.`
- `ct-subdomain.d.tld.` = `ct.example.org.`

Your containers get FQDNs of the form `name.ct.example.net`

### Secrets

The services require credentials to communicate. For this purpose, please create the following files unter `secrets/`:

- `secrets/ct_postgres` with 50 random alphanumeric characters (used as Database password of the `ctapi` user)
- `secrets/db_encrypt` with 50 random alphanumeric characters (used to symmetrically encrypt the ssh host keys of the containers)
- `secrets/gitlab_id` the id of the oauth application from gitlab
- `secrets/gitlab_secret` the secret ot the oauth application

The random passwords may be created by

    head /dev/urandom | tr -dc A-Za-z0-9 | head -c 50 > secrets/ct_postgres
    head /dev/urandom | tr -dc A-Za-z0-9 | head -c 50 > secrets/db_encrypt

#### Gitlab OAuth

To get the gitlab_id and _secret, create an gitlab application with the following parameters:

- callback url:  https://ui-fqdn/social-auth/complete/gitlab/
- not trusted
- confidential (_id and _secret remains at the server)
- scopes: read_user

set the gitlab url for the services beat, api, celery environments:
 
    SOCIAL_AUTH_GITLAB_API_URL: "https://gitlab.yourdomain.com"

### Run

    docker-compose pull
    docker-compose up -d
    
to create an admin user, execute

    docker-compose exec api python3 manage.py createsuperuser

### Backup

To get clean backups, run the `backup.sh` before a system snapshot to dump the database into the backup folder.
