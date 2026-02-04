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

    Apply Flux image repositories
    # kubectl apply -f flux/image-repositories/syncthing.yaml

    Apply Flux image policies
    # kubectl apply -f flux/image-policies/syncthing-policy.yaml

    # Then verify Flux is monitoring them:

    Check image repositories
    # kubectl get imagerepository -n flux-system

    Check image policies
    # kubectl get imagepolicy -n flux-system

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


    Apply Flux image repositories
    # kubectl apply -f flux/image-repositories/radicale.yaml

    Apply Flux image policies
    # kubectl apply -f flux/image-policies/radicale-policy.yaml

    # Then verify Flux is monitoring them:

    Check image repositories
    # kubectl get imagerepository -n flux-system

    Check image policies
    # kubectl get imagepolicy -n flux-system

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

    Apply Flux image repositories
    # kubectl apply -f flux/image-repositories/immich.yaml

    Apply Flux image policies
    # kubectl apply -f flux/image-policies/immich-policy.yaml

    # Then verify Flux is monitoring them:

    Check image repositories
    # kubectl get imagerepository -n flux-system

    Check image policies
    # kubectl get imagepolicy -n flux-system

## Install Jellyfin + AIOStreams via Helm:

    Ensure the defined folders for the local PVs exist and have the right permissions as per the Helm Charts.

    For Jellyfin
    # mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin/config
    # mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin/cache
    # mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin/media
    # chown -R 1000:1000 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin
    # chmod -R 755 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin

    For AIOStreams
    # mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/AIOStreams/data
    # chown -R 1000:1000 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/AIOStreams
    # chmod -R 755 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/AIOStreams

    You need to generate a 64-character hex key for AIOStreams. Run this command on your server:

    # openssl rand -hex 32

    Then update ./kubernetes/aiostreams-chart/values.yaml replacing the placeholder with your generated key.

    Add Jellyfin Helm repository
    # helm repo add jellyfin https://jellyfin.github.io/jellyfin-helm
    # helm repo update

    Deploy AIOStreams
    # helm install aiostreams ./aiostreams-chart --namespace aiostreams --create-namespace

    Deploy Jellyfin
    # kubectl apply -f ./jellyfin-chart/pv.yaml
    # kubectl apply -f ./jellyfin-chart/pvc.yaml
    # helm install jellyfin jellyfin/jellyfin --namespace jellyfin --create-namespace -f ./jellyfin-chart/values.yaml
    # kubectl apply -f ./jellyfin-chart/httproute.yaml


    Apply Flux image repositories
    # kubectl apply -f flux/image-repositories/jellyfin.yaml
    # kubectl apply -f flux/image-repositories/aiostreams.yaml

    Apply Flux image policies
    # kubectl apply -f flux/image-policies/jellyfin-policy.yaml
    # kubectl apply -f flux/image-policies/aiostreams-policy.yaml

    # Then verify Flux is monitoring them:

    Check image repositories
    # kubectl get imagerepository -n flux-system

    Check image policies
    # kubectl get imagepolicy -n flux-system

    NOTE
    Jellyfin/Gelato cannot reach the Manifest file from AIOStreams with the provided URL, his is because aiostreams.casa.local is an external domain that resolves to the internal IP (192.168.0.227), but Jellyfin can't resolve it through the internal Kubernetes DNS.

    The solution is to use the internal Kubernetes service name instead of the external domain. The internal service name is aiostreams.aiostreams.svc.cluster.local or simply aiostreams.aiostreams.

    Update the Gelato configuration in Jellyfin to use:

     http://10.43.52.146:3000/stremio/...



