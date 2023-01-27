# automation
automation scripts 
#!/bin/bash
#
# Install Docker,set up registry and install kind k8s cluster 

set -oe errexit
#Proxy
http_proxy='http://10.7.0.140:8081'

# desired cluster name; default is "kind"
KIND_CLUSTER_NAME="cnoc-cdd-cluster"

# default registry name and port
reg_name='internal.docker.registry'
reg_port='443'
host_ip='10.241.9.84'

#Install docker 
docker_setup()
{
  sudo apt update -y && sudo apt upgrade -y
  sudo apt-get install -y ca-certificates curl gnupg lsb-release
  curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

  sudo apt-get install docker.io -y
  systemctl enable docker.service
  systemctl start docker.service

  echo 1 > /proc/sys/net/ipv4/ip_forward
  lsmod | grep br_netfilter
  sudo modprobe br_netfilter

  cat << EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
  sudo sysctl --system
  cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
  systemctl enable docker
  systemctl daemon-reload
  systemctl restart docker
}

#Setting up containerd
containerd_setup()
{
  cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
  sudo modprobe overlay
  sudo modprobe br_netfilter
  # Setup required sysctl params, these persist across reboots.
  cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

  # Apply sysctl params without reboot
  sudo sysctl --system

  #Install and configure containerd
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  sudo apt update -y
  sudo apt install -y containerd.io
  sudo mkdir -p /etc/containerd/
  sudo touch /etc/containerd/config.toml
  #containerd config default | sudo tee /etc/containerd/config.toml
  cp "$(pwd)"/containerd-config.toml /etc/containerd/config.toml
  mkdir -p /etc/systemd/system/containerd.service.d
  cat <<EOF | sudo tee /etc/systemd/system/containerd.service.d/http-proxy.conf
[Service]
Environment="HTTPS_PROXY=$http_proxy"
Environment="HTTP_PROXY=$http_proxy"
Environment="http_proxy=$http_proxy"
Environment="https_proxy=$http_proxy"
Environment="no_proxy=10.241.9.41,.internal,10.96.0.1,10.244.0.0/16,.svc.cluster.local,internal.docker.registry,.bete.ericy.com, .svc"
Environment="NO_PROXY=10.241.9.41,.internal,10.96.0.1,10.244.0.0/16,.svc.cluster.local,internal.docker.registry,.bete.ericy.com, .svc"
EOF

  #Start containerd
  sudo systemctl restart containerd
  sudo systemctl enable --now containerd
}

#Install crictl
crictl_setup()
{
  VERSION="v1.26.0" # check latest version in /releases page
  wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
  sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
  rm -f crictl-$VERSION-linux-amd64.tar.gz
}

#Create docker registry
registry_setup()
{
  mkdir -p /mnt/registry
  mkdir -p "$(pwd)"/certs
  openssl genrsa -out "$(pwd)"/certs/domain.key 2048
  openssl req -new -key "$(pwd)"/certs/domain.key -out "$(pwd)"/certs/domain.csr -subj "/C=US/ST=Texas/L=Dallas/O=Ericsson Inc./CN=internal.docker.registry"
  openssl x509 -req -days 365 -in  "$(pwd)"/certs/domain.csr -signkey  "$(pwd)"/certs/domain.key -out "$(pwd)"/certs/registry.crt
  cp "$(pwd)"/certs/registry.crt /usr/local/share/ca-certificates/
  update-ca-certificates
  mkdir -p /etc/systemd/system/docker.service.d
  cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTPS_PROXY=$http_proxy"
Environment="HTTP_PROXY=$http_proxy"
Environment="http_proxy=$http_proxy"
Environment="https_proxy=$http_proxy"
Environment="no_proxy=10.241.9.41,.internal,10.96.0.1,10.244.0.0/16,.svc.cluster.local,internal.docker.registry,.bete.ericy.com, .svc"
Environment="NO_PROXY=10.241.9.41,.internal,10.96.0.1,10.244.0.0/16,.svc.cluster.local,internal.docker.registry,.bete.ericy.com, .svc"
EOF
  systemctl daemon-reload
  systemctl restart docker
  docker pull registry
  docker run -d \
    --restart=always \
    --name internal.docker.registry \
    -v "$(pwd)"/certs:/certs \
    -v "$(pwd)"/auth:/auth \
    --mount type=bind,src=/mnt/registry,dst=/var/lib/registry \
    -e "REGISTRY_HTTP_ADDR=0.0.0.0:443" \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
    -e "REGISTRY_AUTH=htpasswd" \
    -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
    -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
    registry:2
}

#Install KIND Cluster
KIND_setup()
{

  #Add the registry in the /etc/hosts
  echo "$host_ip cnoc-automation $reg_name registry" | sudo tee -a /etc/hosts

  echo "> initializing Docker registry"

  # create registry container unless it already exists
  running="$(docker inspect -f '{{.State.Running}}' "${reg_name}" 2>/dev/null || true)"
  if [ "${running}" != 'true' ]; then
    docker run \
      -d --restart=always -p "${reg_port}:5000" --name "${reg_name}" \
      registry:2
  fi

  echo "> initializing Kind cluster: ${KIND_CLUSTER_NAME} with registry ${reg_name}"

  # create a cluster with the local registry enabled in containerd
  cat <<EOF | kind create cluster --name "${KIND_CLUSTER_NAME}" --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:${reg_port}"]
    endpoint = ["http://${reg_name}:${reg_port}"]
EOF

  docker network connect kind "${reg_name}"

  echo "> port-forwarding k8s API server"
  /usr/local/bin/start-portforward-service.sh start

  APISERVER_PORT=$(kubectl config view -o jsonpath='{.clusters[].cluster.server}' | cut -d: -f 3 -)
  /usr/local/bin/portforward.sh $APISERVER_PORT
  kubectl get nodes # make sure it worked

  echo "> port-forwarding local registry"
  /usr/local/bin/portforward.sh $reg_port

  echo "> applying local-registry docs"

  cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "localhost:${reg_port}"
    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
EOF

  echo "> waiting for kubernetes node(s) become ready"
  kubectl wait --for=condition=ready node --all --timeout=60s

  echo "> with-kind-cluster.sh setup complete! Running user script: $@"
  exec "$@"
}

#docker_setup
#containerd_setup
#crictl_setup
registry_setup
#KIND_setup
