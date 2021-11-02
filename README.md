Netbox Docker
=============
Ansible role to configure Netbox as a docker-compose project with a Caddy Reverse-Proxy in front of it, for TLS termination.


Installation
------------
Install this role with Ansible Galaxy:

`ansible-galaxy install git+https://github.com/baburciu/ansible-role-netbox-docker.git --roles-path=~/netbox-project`


Requirements
------------
- [x] `docker` and `docker-compose` must be available and installed on the target system.

- [x] Caddy server certificate and server key are in a directory defined by `netbox_base_dir`:`/opt/netbox` role variable.

```shell
files
└── /opt/netbox
    ├── caddy-server.crt
    └── caddy-server.key
```

- [x] The X509v3 Subject Alternative Name of the `caddy-server.crt` must be defined in `caddy_cert_san` and `caddy_default_sni` role variables.

<br/>
To generate a server key:

```shell
openssl genrsa -out caddy-server.key 4096
```

Generate certificate request (`CSR`) with a configuration file like this:

```shell
# caddy-server.conf
[ req ]
prompt = no
distinguished_name = dn
req_extensions = req_ext

[ dn ]
CN = netbox.boburciu.privatecloud.com
emailAddress = bogdan.burciu@boburciu.privatecloud.com
O = BOBURCIU.PRIVATECLOUD
OU = SEC
C = RO

[ req_ext ]
subjectAltName =  DNS:netbox.boburciu.privatecloud.com, DNS:localhost, IP:192.168.200.222
```

And the following `openssl` command:

```shell
openssl req -new -config caddy-server.conf -key caddy-server.key -out caddy-server.csr
```

Then request the CA to sign the CSR and fetch the certificate, e.g. `caddy-server.crt`.

In case you don't have a CA, we can be our own certificate authority (CA) by creating a self-signed root CA certificate, then install it as a trusted certificate in the local browser.
Create a private key (ca.key) and a self-signed root CA certificate (ca.crt) from the command line:

```shell
openssl req -x509 -sha256 -days 1825 -newkey rsa:2048 -keyout ca.key -out ca.crt
```

Even though CSR can contain SANs, the signer does not have to include all SANs requested (and can itself add SANs it signs), that is why we need to use `-extfile caddy-server.conf -extensions req_ext` so that the `req_ext` section with the SANs is enforced during signing.

```shell
openssl x509 -req -CA ca.crt -CAkey ca.key -extensions req_ext -in caddy-server.csr -out caddy-server.crt -days 365 -CAcreateserial -extfile caddy-server.conf 
```

We can check the server cert used by Caddy for HTTPS clients and notice the SANs (`openssl x509 -text -noout -in caddy-server.crt`).
```shell
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:netbox.boburciu.privatecloud.com, DNS:localhost, IP Address:192.168.200.222
```

Role Variables
--------------
Configuration and installation options are made available as variables. Some of these (`secret_key`, passwords) _must_ be overridden before use!

| Option                         | Default                          | Description
|--------------------------------|----------------------------------|------------
| `netbox_base_dir`              | `/opt/netbox`                    | Root path for netbox's docker-compose file and data store
| `netbox_port`                  | `8080`                           | Host port to expose netbox on. If blank, netbox's port is not exposed
| `netbox_netbox_image`          | `netboxcommunity/netbox:v2.10.4` | Netbox docker image tag
| `netbox_redis_image`           | `redis:6-alpine`                 | Redis docker image tag
| `netbox_postgres_image`        | `postgres:12-alpine`             | Postgres docker image tag
| `netbox_admin_api_token`       | -                                | API token for the default admin user, created on first run
| `netbox_admin_email`           | `admin@example.com`              | Email of the default admin user, created on first run
| `netbox_admin_password`        | `admin`                          | Password for the default admin user, created on first run
| `netbox_admin_username`        | `admin`                          | Username of the default admin user, created on first run
| `netbox_allowed_hosts`         | `*`                              | If set, connections will be restricted to this host header
| `netbox_cors_origin_allow_all` | `true`                           | If set to true, all CORS origins will be allowed
| `netbox_cors_origin_whitelist` | `http://localhost`               | Whitelist of acceptable CORS origin headers
| `netbox_default_page_size`     | `250`                            | Default number of objects for paginated views
| `netbox_email_certfile`        | -                                | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_from`            | `netbox@bar.com`                 | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_keyfile`         | -                                | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_password`        | -                                | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_port`            | `25`                             | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_server`          | `localhost`                      | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_timeout`         | `5`                              | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_use_ssl`         | `false`                          | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_use_tls`         | `false`                          | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_email_username`        | `netbox`                         | [Email server settings documentation](https://docs.djangoproject.com/en/3.1/ref/settings/#std:setting-EMAIL_HOST)
| `netbox_enforce_global_unique` | `false`                          | Enforcement of unique IP space can be toggled on a per-VRF basis. To enforce unique IP space within the global table, set this to true
| `netbox_container_labels`      | `[]`                             | Optional extra container labels to apply to the netbox container. See [Traefiknginx-proxy Support](#traefiknginx-proxy-support)
| `netbox_container_env`         | `[]`                             | Optional extra container environment variables to apply to the netbox container. See [Traefik/nginx-proxy Support](#traefiknginx-proxy-support)
| `netbox_login_required`        | `false`                          | Whether or not a user must be authenticated to view DCIM details in Netbox
| `netbox_max_page_size`         | `1000`                           | Maximum number of objects for paginated API requests
| `netbox_metrics_enabled`       | `false`                          | Expose Prometheus monitoring metrics at the HTTP endpoint '/metrics'
| `netbox_napalm_password`       | -                                | Credentials that NetBox will use to authenticate to devices when connecting via NAPALM
| `netbox_napalm_timeout`        | `10`                             | NetBox session timeout for network devices when connecting via NAPALM
| `netbox_napalm_username`       | -                                | Credentials that NetBox will use to authenticate to devices when connecting via NAPALM
| `netbox_pg_db`                 | `netbox`                         | Postgres database name
| `netbox_pg_host`               | `postgres`                       | Postgres database host. This should not be changed if using the default docker-compose setup.
| `netbox_pg_password`           | `J5brHrAXFLQSif0K`               | Postgres password
| `netbox_pg_user`               | `netbox`                         | Postgres user
| `netbox_proxy_network_name`    | -                                | Extra external network to attach to the netbox container. See [Traefiknginx-proxy Support](#traefiknginx-proxy-support)
| `netbox_redis_cache_host`      | `redis-cache`                    | Redis cache instance host. This should not be changed if using the default docker-compose setup.
| `netbox_redis_cache_password`  | `H733Kdjndks81`                  | Redis cache instance password
| `netbox_redis_host`            | `redis`                          | Redis instance host. This should not be changed if using the default docker-compose setup.
| `netbox_redis_password`        | `H733Kdjndks81`                  | Redis instance password
| `netbox_secret_key`            | -                                | [Netbox secret key](https://docs.djangoproject.com/en/stable/ref/settings/#std:setting-SECRET_KEY). Should be at least 50 characters long
| `netbox_skip_startup_scripts`  | `false`                          | If true, do not run startup scripts on container start
| `netbox_skip_superuser`        | `false`                          | If true, do not create superuser on container start
| `netbox_webhooks_enabled`      | `true`                           | If true, enable netbox webhooks functionality
| `caddy_cert_san`               | `netbox.boburciu.privatecloud`   | List of Subject Alternative Name fields (DNS/IP) for the Caddy server cert
| `caddy_default_sni`            | `192.168.200.222`                | The IP address of the VM on which NetBox/Caddy is deployed, will use this IP address as TLS SNI for all ClientHello messages that come without one (when IP address is used in URL instead of hostname) - mandatory for Caddy


Example Playbook
----------------
Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```shell
---
- name: Deploy NetBox docker with Ansible
  hosts: localhost
  become: yes
  become_method: sudo
  roles:
    - ansible-role-netbox-docker
```

Traefik/nginx-proxy Support
---------------------------
This playbook can be used with [traefik](https://doc.traefik.io/traefik/v2.1/routing/routers/#rule) or jwilder's [nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy) by adding labels with `netbox_container_labels`, or environment variables with `netbox_container_env`, respectively. Additionally, `netbox_proxy_network_name` will attach the netbox service to an additional network, as traefik/nginx-proxy usually reside in a network other than that created by docker-compose projects. 

#### traefik
```yaml
# ./defaults/main.yml:
netbox_container_labels:
  - traefik.enable:'true'
  - traefik.http.services.netbox.loadbalancer.server.port:"8080"
  - traefik.http.routers.netbox.tls:'true'
  - traefik.http.routers.netbox.rule:"Host(`netbox.boburciu.privatecloud.com`)"
netbox_proxy_network_name: 'default'

traefik_container_labels:
  - traefik.enable:"true"
  - traefik.http.routers.traefik.entrypoints:"http"
  - traefik.http.routers.traefik.rule:"Host(`netbox.boburciu.privatecloud.com`)"
  - traefik.http.middlewares.traefik-auth.basicauth.users:"USER:PASSWORD"
  - traefik.http.middlewares.traefik-https-redirect.redirectscheme.scheme:"https"
  - traefik.http.routers.traefik.middlewares:"traefik-https-redirect"
  - traefik.http.routers.traefik-secure.entrypoints:"https"
  - traefik.http.routers.traefik-secure.rule:"Host(`netbox.boburciu.privatecloud.com`)"
  - traefik.http.routers.traefik-secure.middlewares:"traefik-auth"
  - traefik.http.routers.traefik-secure.tls:"true"
  - traefik.http.routers.traefik-secure.tls.certresolver:"http"
  - traefik.http.routers.traefik-secure.service:"api@internal"
traefik_proxy_network_name: 'default'



# templates/docker-compose.yml.j2:
services:
  netbox: &netbox
    image: {{ netbox_netbox_image }}
    depends_on:
    - postgres
    - redis
    - redis-cache
    - netbox-worker
    env_file: env/netbox.env
    user: '101'
    volumes:
    - ./reports:/etc/netbox/reports:z,ro
    - ./scripts:/etc/netbox/scripts:z,ro
    - netbox-media-files:/opt/netbox/netbox/media:z
    {% if netbox_port %}
    ports:
    - "{{ netbox_port }}:8080"
    {% else %}
    expose:
    - "8080"
    {% endif %}
    restart: unless-stopped
    {% if netbox_container_labels %}
    labels:
      {{ netbox_container_labels | to_nice_yaml | indent(6) }}
    {% endif %}
    networks:
    - default
    {% if netbox_proxy_network_name %}
    - proxy
    {% endif %}
  traefik:
    image: "traefik:v2.4"
    container_name: "traefik"
    ports:
      # HTTPS / SSL port
      - "443:443"
      # The Traefik Web UI port (enabled by api:insecure: true in traefik.yml)
      - "8080:8080"
    volumes:
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik/config.yml:/etc/traefik/config.yml:ro
      - ./certs:/etc/certs:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    {% if traefik_container_labels %}
    labels:
      {{ traefik_container_labels | to_nice_yaml | indent(6) }}
    {% endif %}
    networks:
    - default
   {% if traefik_proxy_network_name %}
   - proxy
   {% endif %}
```

License
-------
This project itself is licensed under the terms of the [MIT License](LICENSE). View [license information](https://github.com/netbox-community/netbox/blob/develop/LICENSE.txt) for software contained within.

Usage notes
-------

`yum install python-virtualenv` <br/>
`./ansible-pyenv` <br/>
```shell
[root@gitlab-runner-and-netbox projects]# ansible-galaxy role remove ansible-role-netbox-docker --roles-path=./netbox-project
- successfully removed ansible-role-netbox-docker
[root@gitlab-runner-and-netbox projects]# ansible-galaxy install git+https://github.com/baburciu/ansible-role-netbox-docker.git --roles-path=./netbox-project
Starting galaxy role install process
- extracting ansible-role-netbox-docker to /root/projects/netbox-project/ansible-role-netbox-docker
- ansible-role-netbox-docker was installed successfully
[root@gitlab-runner-and-netbox projects]#
[root@gitlab-runner-and-netbox projects]# ls -lt netbox-project/
total 0
drwxr-xr-x. 8 root root 185 Nov  2 18:30 ansible-role-netbox-docker
[root@gitlab-runner-and-netbox projects]#
[root@gitlab-runner-and-netbox projects]# cd prepare-netbox-docker-ansible/
[root@gitlab-runner-and-netbox prepare-netbox-docker-ansible]# cat deploy-netbox-docker.yml
---
- name: Deploy NetBox docker with Ansible
  hosts: localhost
  become: yes
  become_method: sudo
  roles:
    - ../netbox-project/ansible-role-netbox-docker
[root@gitlab-runner-and-netbox prepare-netbox-docker-ansible]# source ../pyenv-ansible-2.9/bin/activate
(pyenv-ansible-2.9) [root@gitlab-runner-and-netbox prepare-netbox-docker-ansible]# yum install libselinux-python3
(pyenv-ansible-2.9) [root@gitlab-runner-and-netbox prepare-netbox-docker-ansible]# pip install docker-compose
(pyenv-ansible-2.9) [root@gitlab-runner-and-netbox prepare-netbox-docker-ansible]# ansible-playbook deploy-netbox-docker.yml -e 'netbox_base_dir=/opt/netboxnew'

PLAY [Deploy NetBox docker with Ansible] *********************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************
ok: [localhost]

TASK [../netbox-project/ansible-role-netbox-docker : update netbox directories] ******************************************************
ok: [localhost] => (item=env)

TASK [../netbox-project/ansible-role-netbox-docker : update environment files] *******************************************************
ok: [localhost] => (item=netbox.env)
ok: [localhost] => (item=postgres.env)
ok: [localhost] => (item=redis-cache.env)
ok: [localhost] => (item=redis.env)

TASK [../netbox-project/ansible-role-netbox-docker : update configuration files] *****************************************************
ok: [localhost] => (item=docker-compose.yml)
changed: [localhost] => (item=Caddyfile)

TASK [../netbox-project/ansible-role-netbox-docker : remove outdated files] **********************************************************
ok: [localhost] => (item=netbox.env)
ok: [localhost] => (item=configuration/configuration.py)
ok: [localhost] => (item=configuration/gunicorn_config.py)

RUNNING HANDLER [../netbox-project/ansible-role-netbox-docker : restart netbox] ******************************************************
changed: [localhost]

PLAY RECAP ***************************************************************************************************************************
localhost                  : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

(pyenv-ansible-2.9) [root@gitlab-runner-and-netbox prepare-netbox-docker-ansible]# ls -lt /opt/netboxnew                              
total 16
drwxr-xr-x. 2 root root    6 Nov  2 18:44 reports
drwxr-xr-x. 2 root root    6 Nov  2 18:44 scripts
-rw-r--r--. 1 root root  512 Nov  2 18:44 Caddyfile
-rw-r--r--. 1 root root 1826 Nov  2 18:36 caddy-server.crt
-rw-r--r--. 1 root root 3243 Nov  2 18:36 caddy-server.key
-rw-r--r--. 1 root root 2204 Nov  2 18:30 docker-compose.yml
drwxr-xr-x. 2 root root   84 Nov  2 18:26 env
(pyenv-ansible-2.9) [root@gitlab-runner-and-netbox prepare-netbox-docker-ansible]# cd /opt/netboxnew; docker-compose ps
          Name                         Command               State                         Ports
-----------------------------------------------------------------------------------------------------------------------
netboxnew_caddy-rp_1        caddy run --config /etc/ca ...   Up      2019/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
netboxnew_netbox-worker_1   /opt/netbox/venv/bin/pytho ...   Up
netboxnew_netbox_1          /sbin/tini -- /opt/netbox/ ...   Up      8080/tcp
netboxnew_postgres_1        docker-entrypoint.sh postgres    Up      5432/tcp
netboxnew_redis-cache_1     docker-entrypoint.sh sh -c ...   Up      6379/tcp
netboxnew_redis_1           docker-entrypoint.sh sh -c ...   Up      6379/tcp
[root@gitlab-runner-and-netbox netboxnew]#
```
