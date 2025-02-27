# K3s Cluster Setup with Calico, WireGuard, Loki and Grafana

## Clean Existing Installations
```bash
# Uninstall K3s from server node
/usr/local/bin/k3s-uninstall.sh

# Uninstall K3s from agent nodes
/usr/local/bin/k3s-agent-uninstall.sh

# Remove any existing WireGuard interfaces
sudo ip link del wireguard.cali
```

## Install K3s on Server Node with Calico
```bash
# Install K3s with Calico (no Flannel)
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --flannel-backend=none --disable-network-policy" sh -
```

## Install Calico with WireGuard Encryption
```bash
# Apply Calico operator
kubectl apply -f https://docs.projectcalico.org/manifests/tigera-operator.yaml

# Create custom resources for Calico
cat > custom-resources.yaml << EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: true
  cni:
    type: Calico
EOF

# Apply custom resources
kubectl create -f custom-resources.yaml

# Enable WireGuard encryption
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true,"wireguardEnabledV6":true}}'
```

## Verify Calico and WireGuard Status
```bash
# Check Calico status
kubectl get tigerastatus

# Check WireGuard status
sudo wg show
```

## Add Worker Nodes to K3s Cluster
```bash
# Get the join token on server node
NODE_TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)
SERVER_IP=40.160.10.50  # Replace with your server IP

# On each worker node, run:
curl -sfL https://get.k3s.io | K3S_URL=https://${SERVER_IP}:6443 K3S_TOKEN=${NODE_TOKEN} sh -
```

## Install Helm
```bash
# Download and install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Set KUBECONFIG for Helm
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

## Install Loki with S3 Storage
```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Create Loki values file with S3 storage
cat > loki-values.yaml << EOF
loki:
  auth_enabled: false
  storage:
    type: s3
    s3:
      endpoint: s3.ap-south-1.amazonaws.com
      region: ap-south-1
      access_key_id: YOUR_ACCESS_KEY
      secret_access_key: YOUR_SECRET_KEY
      bucketnames: mintairlogs
      s3ForcePathStyle: false
  schemaConfig:
    configs:
      - from: 2020-10-24
        store: boltdb-shipper
        object_store: s3
        schema: v11
        index:
          prefix: index_
          period: 24h

write:
  replicas: 2
  persistence:
    size: 10Gi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 60

read:
  replicas: 2
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 60

backend:
  replicas: 2
  persistence:
    size: 10Gi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 60
EOF

# Install Loki with Helm
helm install loki grafana/loki --namespace monitoring -f loki-values.yaml
```

## Install Grafana Agent for Docker Log Collection
```bash
# Add Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Get Loki service URL
LOKI_URL=$(kubectl get svc -n monitoring loki-gateway -o jsonpath='{.spec.clusterIP}'):3100

# Create Grafana Agent values file
cat > grafana-agent-values.yaml << EOF
agent:
  mode: flow
  enableReporting: false
  clusterName: k3s-cluster
  configMap:
    create: true
    content: |
      logging {
        level  = "info"
        format = "logfmt"
      }

      loki.source.docker "containers" {
        host = "unix:///var/run/docker.sock"
        targets = discovery.docker.containers.targets
        forward_to = [loki.write.loki.receiver]
      }

      discovery.docker "containers" {
        host = "unix:///var/run/docker.sock"
      }

      loki.write "loki" {
        endpoint {
          url = "http://${LOKI_URL}/loki/api/v1/push"
        }
      }

  volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock

controller:
  type: daemonset
EOF

# Install Grafana Agent
helm install grafana-agent grafana/grafana-agent --namespace monitoring -f grafana-agent-values.yaml
```

## Install Grafana on Control Plane Node
```bash
# Create Grafana values file
cat > grafana-values.yaml << EOF
service:
  type: NodePort
persistence:
  enabled: true
  size: 10Gi
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      url: http://loki-gateway.monitoring.svc.cluster.local:3100
      access: proxy
      isDefault: true
nodeSelector:
  node-role.kubernetes.io/control-plane: "true"
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule
EOF

# Install Grafana using Helm
helm install grafana grafana/grafana --namespace monitoring -f grafana-values.yaml

# Get Grafana admin password
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Get Grafana service NodePort
kubectl get svc -n monitoring grafana
```

## Verify Installation
```bash
# Check all deployments
kubectl get pods -n monitoring

# Test Grafana Agent logs
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana-agent --tail=20
```

## Access Grafana
- Access Grafana UI at http://CONTROL_PLANE_IP:NODEPORT
- Username: admin
- Password: (retrieved from secret)
