Production Deployment
=====================

Example how to deploy Pomerium for single-instance production environment.

Clone repositories
------------------

.. code-block:: bash

    mkdir ~/extra2000
    cd ~/extra2000
    git clone https://github.com/extra2000/pomerium-podman.git
    git clone https://github.com/extra2000/pomerium.git pomerium-podman/src/pomerium

Then, ``cd`` into project root directory:

.. code-block:: bash

    cd pomerium-podman

Build Pomerium Image
--------------------

From the project root directory, ``cd`` into ``src/pomerium/`` and then build:

.. code-block:: bash

    cd src/pomerium/
    podman build -t extra2000/pomerium/pomerium .

Deploy Pomerium
---------------

From the project root directory, ``cd`` into ``deployment/production/pomerium``:

.. code-block:: bash

    cd deployment/production/pomerium

Create config files:

.. code-block:: bash

    cp -v configmaps/pomerium.yaml{.example,}
    cp -v configs/config.yaml{.example,}

Create ``signing_key`` in ``configs/config.yaml``:

.. code-block:: bash

    openssl ecparam -genkey -name prime256v1 -noout -out ec_private.pem
    openssl req -x509 -new -key ec_private.pem -days 1000000 -out ec_public.pem -subj "/CN=unused"
    cat ec_private.pem | base64

.. note::

    Copy the generated ``base64`` output of the ``ec_private.pem`` into ``signing_key``. The ``ec_private.pem`` and ``ec_public.pem`` are no longer needed.

Create pod file:

.. code-block:: bash

    cp -v pomerium-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Create SELinux security policy:

.. code-block:: bash

    cp -v selinux/pomerium_podman.cil{.example,}

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/pomerium_podman.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "pomerium_podman"

Deploy Pomerium:

.. code-block:: bash

    podman play kube --configmap configmaps/pomerium.yaml --seccomp-profile-root ./seccomp pomerium-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name pomerium-pod
    systemctl --user enable pod-pomerium-pod.service container-pomerium-pod-srv01.service

Pomerium Behind NGINX
---------------------

NGINX ``conf.d/pomerium.conf``:

.. code-block:: text

    server {
        server_name authenticate.mydomain.io forwardauth.mydomain.io;
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        ssl_certificate /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;
        include /etc/nginx/ssl-params.conf;
        access_log /var/log/nginx/ssl-access.log;
        error_log /var/log/nginx/ssl-error.log;

        client_max_body_size 1000M;

        location / {
            proxy_set_header        Host $host;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header        X-Forwarded-Proto $scheme;
            proxy_pass              http://127.0.0.1:8443;
            proxy_http_version      1.1;
            proxy_read_timeout      90;
        }
    }

NGINX ``conf.d/verifypomerium.conf``:

.. code-block:: text

    server {
        server_name verifypomerium.mydomain.io;
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        ssl_certificate /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;
        include /etc/nginx/ssl-params.conf;
        access_log /var/log/nginx/ssl-access.log;
        error_log /var/log/nginx/ssl-error.log;

        client_max_body_size 1000M;

        location / {
            proxy_set_header        Host $host;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header        X-Forwarded-Proto $scheme;
            proxy_pass              http://127.0.0.1:8443;
            proxy_http_version      1.1;
            proxy_read_timeout      90;
        }
    }
