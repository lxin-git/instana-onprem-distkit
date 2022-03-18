# v215 cstan/cdata deployment


<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [v215 cstan/cdata deployment](#v215-cstancdata-deployment)
	- [Environment](#environment)
	- [Kubenertes Cluster Deployment via kubespray](#kubenertes-cluster-deployment-via-kubespray)
		- [Example to rerun playbook to change settings for ingress add-on](#example-to-rerun-playbook-to-change-settings-for-ingress-add-on)
	- [Instana DB Server Setup](#instana-db-server-setup)
	- [Instana Core & Unit setup in cstan (v215)](#instana-core-unit-setup-in-cstan-v215)
		- [Miscellaneous](#miscellaneous)
		- [Preparation](#preparation)
		- [setup instana core/unit](#setup-instana-coreunit)
		- [Configure nginx ingress controller to expose gateway and acceptor](#configure-nginx-ingress-controller-to-expose-gateway-and-acceptor)
	- [ArgoCD setup](#argocd-setup)

<!-- /TOC -->


## Environment

| host fqdn             | ip            | cpu | mem | disk | role   |
|-----------------------|---------------|-----|-----|------|--------|
| cstan-k1.fyre.ibm.com | 9.112.252.140 | 16  | 32  | 500  | control plane, etcd,worker |
| cstan-k2.fyre.ibm.com | 9.112.252.164 | 16  | 32  | 500  | worker |
| cstan-k3.fyre.ibm.com | 9.112.252.178 | 16  | 32  | 500  | worker |


## Kubenertes Cluster Deployment via kubespray

Reference URL: https://kubespray.io/

- Login to a vm which can be access to `cstan` cluster and has docker daemon installed.
```
ssh root@xtool1.fyre.ibm.com
```

- Clone kubespray repo for k8s deployment
```
mkdir -p /root/deploy/k8s/cstan && cd /root/deploy/k8s/cstan
git clone https://github.com/kubernetes-sigs/kubespray
```

- Prepare ssh key for ansible playbook execution

```
ssh-keygen
ssh-copy-id root@cstan-k1.fyre.ibm.com
ssh-copy-id root@cstan-k2.fyre.ibm.com
ssh-copy-id root@cstan-k3.fyre.ibm.com
```

- Create inventory for k8s cluster

`vim kubespray/inventory/sample/inventory.ini` as following (change value to meet your environment):
```
[all]
cstan-k1.fyre.ibm.com ansible_host=cstan-k1.fyre.ibm.com ip=9.112.252.140
cstan-k2.fyre.ibm.com ansible_host=cstan-k2.fyre.ibm.com ip=9.112.252.164
cstan-k3.fyre.ibm.com ansible_host=cstan-k3.fyre.ibm.com ip=9.112.252.178

[kube_control_plane]
cstan-k1.fyre.ibm.com

[etcd]
cstan-k1.fyre.ibm.com

[kube_node]
cstan-k1.fyre.ibm.com
cstan-k2.fyre.ibm.com
cstan-k3.fyre.ibm.com

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```
- Create extra k8s deploy setting
> Now kubespray uses `containerd` as the container engine by default, and the image command tool uses `nerdctl` by default, but `nerdctl` cannot use the registry auth configured in `/etc/containerd/config.toml`, which causes the playbook failed to pull image from docker.io during execution(the docker pull rate limitation), so you need to set the image command tool to `crictl` and add docker.io  creds in `containerd_registry_auth` configuration (kubespray uses `registry-1.docker.io` as the image by default ), if the network of the cluster nodes is not good, consider adding proxy settings (as follows):

> For kubernetes version, avoid using the latest version, try the version < 1.22, as api version changed since v1.22, which may cause unexcepted error for instana core.
```
cat << EOF > kubespray/extra-settings.yml
kube_version: v1.21.10
image_command_tool: crictl
containerd_registry_auth:
  - registry: registry-1.docker.io
    auth: "bGxpeGlubjpYaW4xQHhpbmw="
upstream_dns_servers:
  - 9.0.146.50
  - 8.8.8.8   		
http_proxy: "http://xcoc-proxy.fyre.ibm.com:3128"
https_proxy: "http://xcoc-proxy.fyre.ibm.com:3128"
EOF
```

> configure to enable addon for argocd & ingress controller, all the options can be find in `kubespray/inventory/sample/group_vars/k8s_cluster/addons.yml`.
```
cat << EOF > kubespray/extra-addons.yml
helm_enabled: true
ingress_nginx_enabled: true
ingress_nginx_host_network: true
cert_manager_enabled: true
argocd_enabled: true
EOF
```

- Launch inception container for playbook kick-off
> Mount the cloned and customized `kubespray` directory to replace the one inside container, mount the private key for ansible remove access. Add proxy setting if required.
```
docker run --rm -it \
           --mount type=bind,source="$(pwd)"/kubespray,dst=/kubespray \
           --mount type=bind,source="${HOME}"/.ssh/id_rsa,dst=/root/.ssh/id_rsa \
           --env HTTP_PROXY="http://xcoc-proxy.fyre.ibm.com:3128" \
           --env HTTPS_PROXY="http://xcoc-proxy.fyre.ibm.com:3128" \
           quay.io/kubespray/kubespray:v2.18.0 bash
 ```          

- Kick-off the playbook INSIDE container for cluster deployment
```           
ansible-playbook -i inventory/sample/inventory.ini \
                 --private-key /root/.ssh/id_rsa cluster.yml \
                 -e @extra-settings.yml \
                 -e @extra-addons.yml
```


### Example to rerun playbook to change settings for ingress add-on

```
ansible-playbook -i inventory/sample/inventory.ini \
                 --private-key /root/.ssh/id_rsa cluster.yml \
                 -e @extra-settings.yml \
                 -e @extra-addons.yml \
                 --tags apps,ingress-nginx,ingress-controller
```


## Instana DB Server Setup

check the instruction here about ansible automation for  [Automatic Deploy Instana Distributed Data Store Components](./README.md).


## Instana Core & Unit setup in cstan (v215)

### Miscellaneous
To check with aval package: https://self-hosted.instana.io/   
RPM packages location: https://self-hosted.instana.io/rpm/release/product/rpm/generic/x86_64/Packages   
Download URL example for specific version of kubectl plugin & instana console:   
https://self-hosted.instana.io/rpm/release/product/rpm/generic/x86_64/Packages/instana-console-215-1.x86_64.rpm   
https://self-hosted.instana.io/rpm/release/product/rpm/generic/x86_64/Packages/instana-kubectl-215-5.x86_64.rpm   

### Preparation

- setup nfs

> cdata-d3.fyre.ibm.com
```
mkdir -p /mnt/nfs/share
chown nobody:nobody /mnt/nfs/share
chmod 777 /mnt/nfs/share
echo "/mnt/nfs/share *(rw,sync,no_root_squash,no_all_squash)" > /etc/exports.d/instana-share.exports
exportfs -rv
systemctl start nfs-server
```

> cstan-k1.fyre.ibm.com
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=${NFS_SERVER} --set nfs.path=${NFS_MOUNT}
```

### setup instana core/unit

makesure k8s is ready.

> cstan-k1.fyre.ibm.com
```
export INSTANA_VERSION="215-5"
export INSTANA_DOWNLOAD_KEY="<Your Download Key>"
export INSTANA_SALES_KEY="<Your Sales Key>"
export INSTANA_FQDN=${INSTANA_FQDN:-$(hostname)}
export ROOT_DIR=/opt/instana/k8s/
export WORK_DIR=${ROOT_DIR}/.work
export DEPLOY_LOCAL_WORKDIR=${WORK_DIR}/local/localdev

export NFS_SERVER="cdata-d3.fyre.ibm.com"
export NFS_MOUNT="/mnt/nfs/share"

export INSTANA_LICENSE=${DEPLOY_LOCAL_WORKDIR}/license
mkdir -p ${DEPLOY_LOCAL_WORKDIR}
mkdir -p ${ROOT_DIR}/conf
cd ${ROOT_DIR}
```

- get license

```
curl https://instana.io/onprem/license/download?salesId=${INSTANA_SALES_KEY} -o ${DEPLOY_LOCAL_WORKDIR}/license
```

- install kubectl instana plugin
```
wget https://self-hosted.instana.io/rpm/release/product/rpm/generic/x86_64/Packages/instana-kubectl-${INSTANA_VERSION}.x86_64.rpm
yum install instana-kubectl-${INSTANA_VERSION}.x86_64.rpm
```

- prepare settings.hcl

```
openssl req -x509 -newkey rsa:2048 -keyout ${DEPLOY_LOCAL_WORKDIR}/tls.key -out ${DEPLOY_LOCAL_WORKDIR}/tls.crt -days 365 -nodes -subj "/CN=*.${INSTANA_FQDN}"
openssl dhparam -out ${DEPLOY_LOCAL_WORKDIR}/dhparams.pem 1024
```
```
export CASSANDRA_NODES=\"cdata-d1.fyre.ibm.com\",\"cdata-d2.fyre.ibm.com\"
export COCKROACHDB_NODES=\"cdata-d2.fyre.ibm.com\",\"cdata-d3.fyre.ibm.com\"
export CLICKHOUSE_NODES=\"cdata-d1.fyre.ibm.com\",\"cdata-d2.fyre.ibm.com\"
export ELASTICSEARCH_NODES=\"cdata-d1.fyre.ibm.com\",\"cdata-d2.fyre.ibm.com\",\"cdata-d3.fyre.ibm.com\"
export KAFKA_NODES=\"cdata-d1.fyre.ibm.com\",\"cdata-d3.fyre.ibm.com\"
export ZOOKEEPER_NODES=\"cdata-d1.fyre.ibm.com\",\"cdata-d2.fyre.ibm.com\",\"cdata-d3.fyre.ibm.com\"
```

```
cat << EOF > conf/settings.hcl.tpl
admin_password = "passw0rd"                        # The initial password the administrator will receive
download_key = "@@INSTANA_DOWNLOAD_KEY"            # Provided by instana
sales_key = "@@INSTANA_SALES_KEY"                  # This identifies you as our customer and is required to activate your license
base_domain = "@@INSTANA_FQDN"                     # The domain under which instana will be reachable
core_name = "instana-core"                         # A name identifiying the Core CRD created by the operator. This needs to be unique if you have more than one instana installation running
tls_crt_path = "@@DEPLOY_LOCAL_WORKDIR/tls.crt"    # Path to the certificate to be used for the HTTPS endpoints of instana
tls_key_path = "@@DEPLOY_LOCAL_WORKDIR/tls.key"    # Path to the key to be used for the HTTPS endpoints of instana
license = "@@INSTANA_LICENSE"                      # License file
dhparams = "@@DEPLOY_LOCAL_WORKDIR/dhparams.pem"   # Diffie–Hellman params for improved connection security
email {                                            # configure this so instana can send alerts and invites
  user = "<user_name>>"
  password = "<user_password>>"
  host = "<smtp_host_name>"
}
token_secret = "randomstring"                      # Secret for generating the tokens used to communicate with instana
databases "cassandra"{                             # Database definitions, see below the code block for a detailed explanation.
    nodes = [@@CASSANDRA_NODES]
}
databases "cockroachdb"{
    nodes = [@@COCKROACHDB_NODES]
}
databases "clickhouse"{
    nodes = [@@CLICKHOUSE_NODES]
}
databases "elasticsearch"{
    nodes = [@@ELASTICSEARCH_NODES]
}
databases "kafka"{
    nodes = [@@KAFKA_NODES]
}
databases "zookeeper"{
    nodes = [@@ZOOKEEPER_NODES]
}
profile = "small"                                  # Specify the memory/cpu-profile to be used for components
spans_location {                                   # Spans can be stored in either s3 or on disk, this is an s3 example
    persistent_volume {                            # Use a persistent volume for raw-spans persistence
        volume_name=""                             # Name of the persistent volume to be used
        storage_class = "nfs-client"               # Storage class to be used
    }
}
ingress "agent-ingress" {                          # This block defines the public reachable name where the agents will connect
    hostname = "@@INSTANA_FQDN"
    port     = 30950
}
units "prod" {                                     # This block defines a tenant unit named prod associated with the tenant instana
    tenant_name       = "instana"
    initial_agent_key = "@@INSTANA_DOWNLOAD_KEY"
    profile           = "small"
}
enable_network_policies = false                    # If set to true network policies will be installed (optional)

EOF
```

```
cat conf/settings.hcl.tpl | \
      sed -e "s|@@INSTANA_DOWNLOAD_KEY|${INSTANA_DOWNLOAD_KEY}|g; \
        s|@@INSTANA_SALES_KEY|${INSTANA_SALES_KEY}|g; \
        s|@@INSTANA_LICENSE|${INSTANA_LICENSE}|g; \
        s|@@INSTANA_FQDN|${INSTANA_FQDN}|g; \
        s|@@CASSANDRA_NODES|${CASSANDRA_NODES}|g; \
        s|@@COCKROACHDB_NODES|${COCKROACHDB_NODES}|g; \
        s|@@CLICKHOUSE_NODES|${CLICKHOUSE_NODES}|g; \
        s|@@ELASTICSEARCH_NODES|${ELASTICSEARCH_NODES}|g; \
        s|@@KAFKA_NODES|${KAFKA_NODES}|g; \
        s|@@ZOOKEEPER_NODES|${ZOOKEEPER_NODES}|g; \
        s|@@DEPLOY_LOCAL_WORKDIR|${DEPLOY_LOCAL_WORKDIR}|g;" > ${DEPLOY_LOCAL_WORKDIR}/settings.hcl
```


- prepare operator & crs yaml
```
cat << EOF > conf/values.yaml
imagePullSecrets:
  - name: instana-registry
EOF
```

```
kubectl instana operator template --namespace=instana-operator --cluster-domain=cluster.local --values conf/values.yaml --output-dir ${DEPLOY_LOCAL_WORKDIR}/out
```

> After generating the above manifest, check followings in the output directory:
> - If the image version in `deployment_instana-operator_instana-operator.yaml` correct.
> - If the `imagePullSecrets` (instana-registry) correct.
>
> If you don't have cert-manager deployed in k8s, try `kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml` to deploy cert-manager (In this tutorial cert-manager already enabled during kubespray k8s setup).


```
kubectl instana migrate --settings-file ${DEPLOY_LOCAL_WORKDIR}/settings.hcl --output-dir ${DEPLOY_LOCAL_WORKDIR}/out
```
> After generating the above manifest, check whether the agent key is imported in `${DEPLOY_LOCAL_WORKDIR}/out/secrets/secret_instana-operator_instana-registry.yaml`

- run installation
```
kubectl create namespace instana-operator
kubectl create namespace instana-core
kubectl create namespace instana-units

```
```
kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/secrets/
kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/
```
> After apply, check and wait for instana-operator ready: `kubectl get po -n instana-operator`


```
kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/crs/
```
> After apply.
> check and wait for everything in instana-core namespace ready: `kubectl get po -n instana-core`
> check and wait for tu-instana-prod-ui-backend deployment ready: `kubectl get deploy tu-instana-prod-ui-backend -n instana-units`


### Configure nginx ingress controller to expose gateway and acceptor

- Edit ingress-nginx daemonset`kubectl edit ds ingress-nginx-controller -n ingress-nginx` , add args `--enable-ssl-passthrough`
```
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/ingress-nginx
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
        - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
        - --annotations-prefix=nginx.ingress.kubernetes.io
        - --watch-ingress-without-class=true
        - --report-node-internal-ip-address
        - --enable-ssl-passthrough
```
> To expose Instana UI service, we need enable ssl passthrough for ingress contoller, this was disabled by default during ingress controller deployment.

- Create ingress for gateway service:
```
cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/gateway-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: instata-core-ingress-https
  namespace: instana-core
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"    
spec:
  rules:
    - host: "cstan-k1.fyre.ibm.com"
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: gateway
              port:
                number: 8443
    - host: "prod-instana.cstan-k1.fyre.ibm.com"
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: gateway
              port:
                number: 8443
```
```
kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/gateway-ingress.yaml
```

- Create nodeport service to expose acceptor (deprecated, will revise to use ingress)

```
cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: acceptor-nodeport
  namespace: instana-core
spec:
  ports:
    - name: http-service
      port: 8600
      nodePort: 30950
      protocol: TCP
  selector:
	  app.kubernetes.io/component: acceptor
	  app.kubernetes.io/name: instana
	  app.kubernetes.io/part-of: core
	  instana.io/group: service
  type: NodePort
EOF
```
```
kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/nodeport.yaml
```
- Access backend UI

make sure your dns service can resolve fqdn `${INSTANA_FQDN}` and `prod-instana.${INSTANA_FQDN}` to any of your cluster node ip
> In this example, you can use any of these 3 ip: 9.112.252.140, 9.112.252.164, 9.112.252.178

https://${INSTANA_FQDN}   
admin@instana.local/passw0rd   


## ArgoCD setup

After kubespray deployed the cluster, argocd automatic installed via addon setting.

- Expose service
You need expose the `argocd-server` service first:
```
kubectl edit svc -n argocd argocd-server
```
change `spec.type` to `NodePort`

> You can also expose the service via ingress rather than nodeport. up 2 you.

- Get admin password
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

- Login to argocd UI
```
https://cstan-k1.fyre.ibm.com:32280
admin/aW2m9X6KekmHwGxb
```
