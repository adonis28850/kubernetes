# K3S Notes

## Install with Secrets encryption using "secretbox"

    # curl -sfL https://get.k3s.io | sh -s - server --secrets-encryption --secrets-encryption-provider secretbox --disable=traefik --node-name k3s-cluster  --tls-san openmediavault --tls-san 192.168.0.227

    # sudo k3s secrets-encrypt status

## Change permissions not to need "sudo" all the time:

    # sudo chmod 644 /etc/rancher/k3s/k3s.yaml

## Set up the environment variable for kubectl and other add-ons to use the correct configuration file:

    # echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bashrc
    # source ~/.bashrc

## Enable Kubectl autocomplete

    # echo 'source <(kubectl completion bash)' >>~/.bashrc
    # echo 'alias k=kubectl' >>~/.bashrc
    # echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

## Delete packed Traefik and install main one

    Delete manifests from the server node:

        # rm -f /var/lib/rancher/k3s/server/manifests/traefik*
        # rm -f /var/lib/rancher/k3s/server/static/charts/traefik*

    Add traefik repository

        # helm repo add traefik https://helm.traefik.io/traefik
        # helm repo update

    Download and install latest version of Gateway API with support for Experimental features (TCPRoutes and UDPRoutes)

        # kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/experimental-install.yaml

    Install traefik

        # helm install traefik traefik/traefik --namespace traefik --create-namespace

    Verify the installation

        # kubectl get pods -n kube-system | grep traefik

## Enable and configure Traefik listeners to use Gateway API and dashboard:

    We need to create a self-signed certificate to terminate TLS on the websecure listener:

        # openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=*.k3s.local"

    Then create a new Secret with this certificate:

        # kubectl create secret tls local-selfsigned-tls --cert=tls.crt --key=tls.key --namespace traefik

    First create another Secret for Traefik's dashboard:

        # USER_PASS=$(htpasswd -nb admin password)
        # kubectl create secret generic dashboard-auth --from-literal=users="$USER_PASS" --namespace traefik

    Apply the custom values:

    # helm upgrade traefik traefik/traefik --namespace traefik -f values_basic.yaml -f values_extra_services --reuse-values

## Install Syncthing via Helm:

    Ensure the defined folders for the local PV exist and have the right permissions as per the Helm Chart.

    Create the directories
    # mkdir -p /home/antonio/data/syncthing/config/
    # mkdir -p /home/antonio/data/syncthing/data/

    Set ownership to UID 1000 and GID 1000 (matching the securityContext)
    # chown -R 1000:1000 /home/antonio/data/syncthing/

    Set appropriate permissions (Owner: read/write/execute, Group: read/execute)
    # chmod -R 755 /home/antonio/data/syncthing/

    # helm install syncthing syncthing-chart/ --namespace syncthing --create-namespace

## Install Radicale via Helm:

    Ensure the defined folders for the local PV exist and have the right permissions as per the Helm Chart.

    Create the directories
    # mkdir -p /home/antonio/data/radicale/config/
    # mkdir -p /home/antonio/data/radicale/data/

    Create config file:
    # cp radicale-chart/config /home/antonio/data/radicale/config

    Create a secret with the htpasswd credentials:
    # htpasswd -c auth username
    # kubectl create namespace radicale
    # kubectl create secret generic radicale-auth --from-file=auth -n radicale

    Set ownership to UID 1000 and GID 1000 (matching the securityContext)
    # chown -R 1000:1000 /home/antonio/data/radicale/

    Set appropriate permissions (Owner: read/write/execute, Group: read/execute)
    # chmod -R 755 /home/antonio/data/radicale/

    # helm install radicale radicale-chart --namespace radicale

## Install Immich via Helm:

    Ensure the defined folders for the local PVs exist and have the right permissions as per the Helm Chart.

    Create the directories
    # mkdir -p /home/antonio/data/immich/library
    # mkdir -p /home/antonio/data/immich/database
    # mkdir -p /home/antonio/data/immich/valkey

    Apply proper ownership (UID 1000, GID 1000)
    # chown -R 1000:1000 /home/antonio/data/immich

    Set permissions to allow the containers to read and write
    # chmod -R 775 /home/antonio/data/immich

    # helm install immich immich-chart --namespace immich --create-namespace
    
