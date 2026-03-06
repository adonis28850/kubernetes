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

## Install Jellyfin + Gluetun VPN + AIOStreams via Helm:

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

    Generate ProtonVPN WireGuard configuration for Gluetun:
    1. Login to ProtonVPN web console
    2. Go to Downloads → WireGuard configuration
    3. Fill out the form:
       - Platform: GNU/Linux
       - Protocol: WireGuard
       - Features: Select "Streaming" or leave default
       - Country: Choose your preferred country
    4. IMPORTANT: Note the Private Key - it's only shown once!

    Add Jellyfin Helm repository
    # helm repo add jellyfin https://jellyfin.github.io/jellyfin-helm
    # helm repo update

    Deploy AIOStreams
    # helm install aiostreams ./aiostreams-chart --namespace aiostreams --create-namespace

    Deploy Jellyfin with Gluetun VPN sidecar:
    # kubectl create namespace jellyfin
    # kubectl apply -f ./jellyfin-chart/pv.yaml
    # kubectl apply -f ./jellyfin-chart/pvc.yaml
    # kubectl create secret generic gluetun-vpn-credentials --from-literal=WIREGUARD_PRIVATE_KEY="your_private_key_from_protonvpn" --namespace jellyfin
    # helm install jellyfin jellyfin/jellyfin --namespace jellyfin -f ./jellyfin-chart/values.yaml
    # kubectl apply -f ./jellyfin-chart/httproute.yaml

    **NOTE:** This implementation uses extraContainers as a temporary workaround. The official
    Jellyfin Helm chart v2.7.0 doesn't support extraInitContainers with restartPolicy: Always
    for native Kubernetes sidecars. When chart v3.0.0 is released, this should be updated to use
    extraInitContainers for proper sidecar behavior that blocks Jellyfin startup until VPN is ready.

    Verify Gluetun VPN is working:
    # kubectl logs -n jellyfin deployment/jellyfin -c gluetun | grep -i connected
    # kubectl exec -n jellyfin deployment/jellyfin -- curl -s https://ipinfo.io/ip

    Verify Jellyfin web UI is accessible:
    # kubectl exec -n traefik deployment/traefik -- curl -s http://jellyfin.jellyfin.svc.cluster.local:8096/health

    Apply Flux image repositories
    # kubectl apply -f flux/image-repositories/jellyfin.yaml
    # kubectl apply -f flux/image-repositories/aiostreams.yaml
    # kubectl apply -f flux/image-repositories/gluetun.yaml

    Apply Flux image policies
    # kubectl apply -f flux/image-policies/jellyfin-policy.yaml
    # kubectl apply -f flux/image-policies/aiostreams-policy.yaml
    # kubectl apply -f flux/image-policies/gluetun-policy.yaml

    # Then verify Flux is monitoring them:

    Check image repositories
    # kubectl get imagerepository -n flux-system

    Check image policies
    # kubectl get imagepolicy -n flux-system

## Install Weather Station via Helm:

     The Weather Station consists of three components:
     - Core: REST API and dashboard (accessible at weather.casa.local)
     - Ingestor: Captures RF signals from RTL-SDR dongle
     - PostgreSQL: Database for weather data

     Prerequisites:
     1. Install Ko for building Go containers
        https://ko.build/install/#install-from-github-releases

     Build images (they'll be available to k3s as docker.io/k3s/core and docker.io/k3s/ingestor):
     # cd /home/antonio/weather_station/core
     # KO_DOCKER_REPO=k3s ko build --tarball=/tmp/core.tar --sbom=none ./
     # sudo k3s ctr images import /tmp/core.tar

     # cd /home/antonio/weather_station/ingestor
     # KO_DEFAULTBASEIMAGE=hertzg/rtl_433:latest KO_DOCKER_REPO=k3s ko build --tarball=/tmp/ingestor.tar --sbom=none --disable-optimizations ./
     # sudo k3s ctr images import /tmp/ingestor.tar

     Note: The Core application embeds static files (dashboard) using Go's embed package, so no special configuration is needed.

     Note: The namespace is created by the PostgreSQL chart. If you encounter namespace ownership errors
     when deploying additional charts, clear the Helm ownership metadata:
     # kubectl annotate namespace weather-station meta.helm.sh/release-name- meta.helm.sh/release-namespace- --overwrite
     # kubectl label namespace weather-station app.kubernetes.io/managed-by- --overwrite

     Ensure the defined folders for the local PV exist and have the right permissions as per the Helm Chart.

     Create the directories
     # mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/WeatherStation/postgres/
     # mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/WeatherStation/postgres-backups/

     Set ownership to UID 1001 and GID 1001 (matching PostgreSQL securityContext)
     # chown -R 1001:1001 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/WeatherStation/

     Set appropriate permissions (Owner: read/write/execute, Group: read/execute)
     # chmod -R 755 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/WeatherStation/

     Create Kubernetes Secret for PostgreSQL credentials (replace with strong passwords):
     # kubectl create namespace weather-station
     # kubectl create secret generic weather-station-postgres-auth \
       --from-literal=postgres-password=<strong-postgres-admin-password> \
       --from-literal=password=<strong-user-password> \
       --from-literal=username=weather_user \
       --namespace weather-station

     Deploy PostgreSQL (dependencies must be built first):
     # cd /home/antonio/kubernetes/weather_station/weather_station-postgres-chart
     # helm dependency build
     # helm install weather-station-postgres ./weather_station-postgres-chart --namespace weather-station

     Wait for PostgreSQL to be ready:
     # kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=postgresql -n weather-station --timeout=300s

     Deploy Core service:
     # helm install weather-station-core ./weather_station-core-chart --namespace weather-station

     Deploy Ingestor service:
     # helm install weather-station-ingestor ./weather_station-ingestor-chart --namespace weather-station

     Verify the installation:
     # kubectl get pods -n weather-station
     # kubectl get svc -n weather-station
     # kubectl get gateway -n weather-station
     # kubectl get httproute -n weather-station

     Access the dashboard:
     Open https://weather.casa.local in your browser

     Check logs:
     # kubectl logs -n weather-station deployment/weather-station-core
     # kubectl logs -n weather-station deployment/weather-station-ingestor

     Check RTL-SDR device is accessible:
     # kubectl exec -n weather-station deployment/weather-station-ingestor -- ls -la /dev/bus/usb/

     Troubleshooting:
     - If ingestor can't access USB device, check device permissions:
       # ls -la /dev/bus/usb/
       # kubectl describe pod -n weather-station -l app.kubernetes.io/name=weather-station-ingestor

     - If dashboard is not accessible, check Gateway and HTTPRoute:
       # kubectl get gateway -n weather-station
       # kubectl get httproute -n weather-station
       # kubectl logs -n traefik deployment/traefik | grep weather

     - If database connection fails, check PostgreSQL is running:
       # kubectl get pods -n weather-station -l app.kubernetes.io/name=postgresql
       # kubectl logs -n weather-station -l app.kubernetes.io/name=postgresql

     Uninstall (if needed):
     # helm uninstall weather-station-ingestor -n weather-station
     # helm uninstall weather-station-core -n weather-station
     # helm uninstall weather-station-postgres -n weather-station
     # kubectl delete namespace weather-station

     Note: If you need to redeploy after uninstalling, the namespace will be recreated by the PostgreSQL chart.
     If you encounter namespace ownership errors when deploying additional charts, clear the Helm ownership metadata:
     # kubectl annotate namespace weather-station meta.helm.sh/release-name- meta.helm.sh/release-namespace- --overwrite
     # kubectl label namespace weather-station app.kubernetes.io/managed-by- --overwrite

     Access the dashboard:
     The weather station dashboard is available at https://weather.casa.local/
     API endpoints:
     - /health - Health check
     - /api/ingest - Ingest weather data
     - /api/weather/current - Current weather
     - /api/weather/recent - Recent readings
     - /api/weather/history - Historical data
     - /api/weather/years - Available years


