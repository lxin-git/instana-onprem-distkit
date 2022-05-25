# Deploy Instana Backend in Kubernetes with Distributed Datastores
This is the deployment instruction for Self-Hosted K8S Instana Backend, which include the automation for VM based datastore components required by Instana Backend.
We will take the Instana V221 as an example, Instana operator start to support `v1beta2` CR in this version, and `v1beta1` will be deprecated from V225, so we will use `v1beta2` in this turorial.


  - [Environment](#environment)
  - [Kubernetes Cluster Deployment via kubespray](#kubernetes-cluster-deployment-via-kubespray)
  - [Distributed Datastores Deployment via Playbooks](#distributed-datastores-deployment-via-playbooks)
  - [Instana Core & Unit Setup (v221)](#instana-core--unit-setup-v221)
    - [Miscellaneous](#miscellaneous)
    - [Preparation](#preparation)
    - [Deploy Instana Operator](#deploy-instana-operator)
    - [Deploy Instana Core & Unit](#deploy-instana-core--unit)
    - [Configure ingress to expose gateway & acceptor](#configure-ingress-to-expose-gateway--acceptor)
    - [Access backend UI](#access-backend-ui)
  - [Known Limitation](#known-limitation)


## Environment

| host fqdn             | ip            | cpu | mem | disk | role   |
|-----------------------|---------------|-----|-----|------|--------|
| bstan-1.fyre.ibm.com | 9.112.255.202 | 16  | 32  | 500  | control plane, etcd,worker |
| bstan-2.fyre.ibm.com | 9.123.116.165 | 16  | 32  | 500  | worker |
| bstan-3.fyre.ibm.com | 9.123.116.167 | 16  | 32  | 500  | worker |


## Kubernetes Cluster Deployment via kubespray

In this section, we will demostrate how to use `kubespray` to deploy a production kubernetes cluster. For detailed instruction please refer to https://kubespray.io/ 
You can choose your own approach to create the kubernetes cluster to run Instana Backend.   

The following add-on should be installed in your kubernetes cluster:   
    - Cert manager
    - Nginx ingress controller

Steps to build your k8s cluster via kubespray:
1. Login to a vm which can be access to `bstan` cluster and has docker daemon installed. 
eg:
    ```
    ssh root@xtool1.fyre.ibm.com
    ```

1. Clone kubespray repo for k8s deployment
    ```
    mkdir -p /root/deploy/k8s/bstan && cd /root/deploy/k8s/bstan
    git clone https://github.com/kubernetes-sigs/kubespray
    ```

1. Prepare ssh key for ansible playbook execution
    ```
    ssh-keygen
    ssh-copy-id root@bstan-1.fyre.ibm.com
    ssh-copy-id root@bstan-2.fyre.ibm.com
    ssh-copy-id root@bstan-3.fyre.ibm.com
    ``` 

1. Create inventory for k8s cluster
    `vim kubespray/inventory/sample/inventory.ini` as following (change value to meet your environment):
    ```
    [all]
    bstan-1.fyre.ibm.com ansible_host=bstan-1.fyre.ibm.com ip=9.112.255.202
    bstan-2.fyre.ibm.com ansible_host=bstan-2.fyre.ibm.com ip=9.123.116.165
    bstan-3.fyre.ibm.com ansible_host=bstan-3.fyre.ibm.com ip=9.123.116.167

    [kube_control_plane]
    bstan-1.fyre.ibm.com

    [etcd]
    bstan-1.fyre.ibm.com

    [kube_node]
    bstan-1.fyre.ibm.com
    bstan-2.fyre.ibm.com
    bstan-3.fyre.ibm.com

    [calico_rr]

    [k8s_cluster:children]
    kube_control_plane
    kube_node
    calico_rr
    ```

1. Create extra k8s deploy setting

    Now kubespray uses `containerd` as the container engine by default, and the image command tool uses `nerdctl` by default, but `nerdctl` cannot use the registry auth configured in `/etc/containerd/config.toml`, which causes the playbook failed to pull image from docker.io during execution(the docker pull rate limitation), so you need to set the image command tool to `crictl` and add docker.io  creds in `containerd_registry_auth` configuration (kubespray uses `registry-1.docker.io` as the image by default ), if the network of the cluster nodes is not good, consider adding proxy settings (as following):
    &nbsp; 
    ```
    cat << EOF > kubespray/extra-settings.yml
    image_command_tool: crictl
    containerd_registry_auth:
    - registry: registry-1.docker.io
        auth: "<your docker.io creds (b64 encoded string for username:passwd)>"
    upstream_dns_servers:
    - 9.0.146.50   
    http_proxy: "http://<proxy_ip>:<port>"
    https_proxy: "http://<proxy_ip>:<port>"
    EOF
    ```

    &nbsp;
    Configure to enable addon for cert-manager & ingress controller, all the options can be find in `kubespray/inventory/sample/group_vars/k8s_cluster/addons.yml`.
    ```
    cat << EOF > kubespray/extra-addons.yml
    helm_enabled: true
    ingress_nginx_enabled: true
    ingress_nginx_host_network: true
    cert_manager_enabled: true
    argocd_enabled: true
    EOF
    ```
    > `ingress_nginx` will be enabled for Instana UI endpoint setup.
    > `cert_manager` will be enabled to automatically provision the secret of `instana-operator-webhook-certs`.

1. Launch inception container for playbook kick-off
    Mount the cloned and customized `kubespray` directory to replace the one inside container, mount the private key for ansible remote access. After following command, you will be in the inception container:
    ```
    docker run --rm -it \
            --mount type=bind,source="$(pwd)"/kubespray,dst=/kubespray \
            --mount type=bind,source="${HOME}"/.ssh/id_rsa,dst=/root/.ssh/id_rsa \
            --env HTTP_PROXY="http://<proxy_ip>:<port>" \
            --env HTTPS_PROXY="http://<proxy_ip>:<port>" \
            quay.io/kubespray/kubespray:v2.18.0 bash
    ```          
    > Add proxy setting if required. Noted that if you're reinstalling via this playbook, make sure using the ip address as proxy host rather than fqdn. eg. `http://9.112.252.254:3128`, this is because the dns setting has been changed in cluster nodes during first installation. and may not able to resolve your proxy hostname.
    
    &nbsp;
    Kick-off the playbook INSIDE container for cluster deployment
    ```           
    ansible-playbook -i inventory/sample/inventory.ini \
                    --private-key /root/.ssh/id_rsa cluster.yml \
                    -e @extra-settings.yml \
                    -e @extra-addons.yml
    ```

    After playbook finished execution, you will get your the k8s cluster up & running. 
    &nbsp;

    To rerun playbook to change settings for ingress add-on

    ```
    ansible-playbook -i inventory/sample/inventory.ini \
                    --private-key /root/.ssh/id_rsa cluster.yml \
                    -e @extra-settings.yml \
                    -e @extra-addons.yml \
                    --tags apps,ingress-nginx,ingress-controller
    ```




## Distributed Datastores Deployment via Playbooks
This repo covered the ansible playbooks to help you automaticly deploy all the necessary datastores for Instana.  Check the instruction here about how to use the ansible automation: [Automatic Deploy Instana Distributed Data Store Components](./auto-deploy-dbsvrs.md)


## Instana Core & Unit Setup (v221)

After kubernetes cluster and datastores ready, you can start to deploy Instana backend into kubernetes cluster and configure the datastore connections. 
> Playbooks and gitops assets are under develop in this repo to enable the automation for all the steps in this section.
 
### Miscellaneous
To check with aval package: https://self-hosted.instana.io/
RPM packages location: https://self-hosted.instana.io/rpm/release/product/rpm/generic/x86_64/Packages
Download URL example for specific version of kubectl plugin & instana console:
https://self-hosted.instana.io/rpm/release/product/rpm/generic/x86_64/Packages/instana-console-215-1.x86_64.rpm
https://self-hosted.instana.io/rpm/release/product/rpm/generic/x86_64/Packages/instana-kubectl-215-5.x86_64.rpm


### Preparation

This is the datastores allocation in my sample environment.   

| Host                  | zookeeper | kafka | clickhouse | cockroachdb | cassandra | elasticsearch | nfs |
|-----------------------|:---------:|:-----:|:----------:|:-----------:|:---------:|:-------------:|:---:|
| bstore-1.fyre.ibm.com | X         | X     | X          |             | X         | X (master)    |     |
| bstore-2.fyre.ibm.com | X         |       | X          | X           | X         | X(data)       |     |
| bstore-3.fyre.ibm.com | X         | X     |            | X           |           | X(data)       | X   |

> In my casse, most kubernetes related command will be executed in `bstan-1.fyre.ibm.com` as this is the k8s master node.


1. Setup NFS Provisioner
    In this example, we use nfs provisioner to create the necessary persistent volumes for Instana backend, which are required by Instana rawSpansStorage.  
    &nbsp;
    > bstore-3.fyre.ibm.com
    ```
    mkdir -p /mnt/nfs/share
    chown nobody:nobody /mnt/nfs/share
    chmod 777 /mnt/nfs/share
    echo "/mnt/nfs/share *(rw,sync,no_root_squash,no_all_squash)" > /etc/exports.d/instana-share.exports
    exportfs -rv
    systemctl enable nfs-server
    systemctl start nfs-server
    ```
    &nbsp;
    > bstan-1.fyre.ibm.com
    ```
    export NFS_SERVER="bstore-3.fyre.ibm.com"
    export NFS_MOUNT="/mnt/nfs/share"
    helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
    helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=${NFS_SERVER} --set nfs.path=${NFS_MOUNT}
    ```

1. Variables setting

    > bstan-1.fyre.ibm.com
    ```
    export INSTANA_VERSION="221-3"
    export INSTANA_DOWNLOAD_KEY="<your download key>"
    export INSTANA_SALES_KEY="<your sales key>"
    export INSTANA_FQDN=${INSTANA_FQDN:-$(hostname)}
    export INSTANA_ADMINPASSWORD="passw0rd"
    export ROOT_DIR=/opt/instana/k8s/
    export WORK_DIR=${ROOT_DIR}/.work
    export DEPLOY_LOCAL_WORKDIR=${WORK_DIR}/local/localdev
    export NFS_SERVER="bstore-3.fyre.ibm.com"
    export NFS_MOUNT="/mnt/nfs/share"
    export INSTANA_LICENSE=${DEPLOY_LOCAL_WORKDIR}/license

    export INSTANA_TENANT="tenant0"
    export INSTANA_UNIT="unit0"
    mkdir -p ${DEPLOY_LOCAL_WORKDIR}/out
    mkdir -p ${ROOT_DIR}/conf/out
    cd ${ROOT_DIR}
    ```

2. install kubectl instana plugin

    > bstan-1.fyre.ibm.com
    ```
    wget https://self-hosted.instana.io/rpm/release/product/rpm/generic/x86_64/Packages/instana-kubectl-${INSTANA_VERSION}.x86_64.rpm
    yum install -y instana-kubectl-${INSTANA_VERSION}.x86_64.rpm
    ```

3. get license

    > bstan-1.fyre.ibm.com
    ```
    curl https://instana.io/onprem/license/download?salesId=${INSTANA_SALES_KEY} -o ${INSTANA_LICENSE}
    ```

### Deploy Instana Operator

1. Render manifest for Instana Registry pull secret
    ```
    cat << EOF > $ROOT_DIR/conf/out/values.yaml
    imagePullSecrets:
      - name: instana-registry
    EOF
    ```
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/secret_instana-operator_instana-registry.yaml
    apiVersion: v1
    kind: Secret
    stringData: 
      .dockerconfigjson: >-
        {
          "auths": {
            "https://containers.instana.io": {
              "username": "_",
              "password":"${INSTANA_DOWNLOAD_KEY}",
              "email": ""
            }
          }
        }
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/name: instana
      name: instana-registry
      namespace: instana-operator
    type: kubernetes.io/dockerconfigjson
    EOF
    ```

1. Render other operator manifest
    ```
    kubectl instana operator template --namespace=instana-operator --cluster-domain=cluster.local --values $ROOT_DIR/conf/out/values.yaml --output-dir ${DEPLOY_LOCAL_WORKDIR}/out
    ```

1. Deploy the Instana Operator
    ```
    kubectl create namespace instana-operator
    kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/
    ```
    sample output:
    ```console
    [root@bstan-1 k8s]# kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/
    certificate.cert-manager.io/instana-operator-webhook-certs created
    clusterrole.rbac.authorization.k8s.io/instana-operator created
    clusterrolebinding.rbac.authorization.k8s.io/instana-operator created
    customresourcedefinition.apiextensions.k8s.io/cores.instana.io created
    customresourcedefinition.apiextensions.k8s.io/datastores.instana.io created
    customresourcedefinition.apiextensions.k8s.io/units.instana.io created
    deployment.apps/instana-operator created
    issuer.cert-manager.io/instana-operator created
    mutatingwebhookconfiguration.admissionregistration.k8s.io/instana-operator-mutating created
    role.rbac.authorization.k8s.io/instana-operator-leader-election created
    rolebinding.rbac.authorization.k8s.io/instana-operator-leader-election created
    secret/instana-registry created
    service/instana-operator created
    serviceaccount/instana-operator created
    validatingwebhookconfiguration.admissionregistration.k8s.io/instana-operator-validating created
    ```

1. Check Operator Status
    ```console
    [root@bstan-1 k8s]# kubectl get po -n instana-operator
    NAME                               READY   STATUS    RESTARTS   AGE
    instana-operator-d6c49f9f7-gwnqr   1/1     Running   0          14m
    ```

### Deploy Instana Core & Unit

1. Create self-signed certificate for instana-tls secret
    ```
    openssl req -x509 -newkey rsa:2048 -keyout ${DEPLOY_LOCAL_WORKDIR}/tls.key -out ${DEPLOY_LOCAL_WORKDIR}/tls.crt -days 365 -nodes -subj "/CN=*.${INSTANA_FQDN}"
    openssl dhparam -out ${DEPLOY_LOCAL_WORKDIR}/dhparams.pem 1024
    ```

    awk 黑科技(for ansible instana-tls.yaml.j2 use )：
    ```
    awk 'NR>2 { sub(/\r/, ""); printf "%s",last} { last=$0 }' tls.key
    awk 'NR>2 { sub(/\r/, ""); printf "%s",last} { last=$0 }' tls.crt
    ```


1. Render all secrets manifest
    There are 5 key secrets need to be created start from v221:
    - The core instana ingress cert: `secret_instana-core_instana-tls.yaml`
    - The instana-registry pull secret for both core & unit namespace:  `secret_instana-core_instana-registry.yaml` & `secret_instana-units_instana-registry.yaml`
    - The core & unit configure secret: `instana-core.yaml` & `${INSTANA_TENANT}-${INSTANA_UNIT}.yaml`
    
    Here are the steps to create the necessary secrets:

    ```
    mkdir -p ${DEPLOY_LOCAL_WORKDIR}/out/secrets/
    ```
        
    > (1) `secret_instana-core_instana-tls.yaml`
    ```
    kubectl create secret tls instana-tls --namespace instana-core \
        --cert=${DEPLOY_LOCAL_WORKDIR}/tls.crt \
        --key=${DEPLOY_LOCAL_WORKDIR}/tls.key \
        --dry-run=client \
        --output=yaml | \
        sed  '/^metadata:/a\ \ labels: {"app.kubernetes.io/name":"instana"}' \
        > ${DEPLOY_LOCAL_WORKDIR}/out/secrets/secret_instana-core_instana-tls.yaml
    ```

    > (2) `secret_instana-core_instana-registry.yaml`
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/secrets/secret_instana-core_instana-registry.yaml
    apiVersion: v1
    stringData: 
      .dockerconfigjson: >-
        {
          "auths": {
            "https://containers.instana.io": {
              "username": "_",
              "password":"${INSTANA_DOWNLOAD_KEY}",
              "email": ""
            }
          }
        }
    kind: Secret
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/name: instana
      name: instana-registry
      namespace: instana-core
    type: kubernetes.io/dockerconfigjson
    EOF
    ```

    > (3) `secret_instana-units_instana-registry.yaml`
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/secrets/secret_instana-units_instana-registry.yaml
    apiVersion: v1
    stringData: 
      .dockerconfigjson: >-
        {
          "auths": {
            "https://containers.instana.io": {
              "username": "_",
              "password":"${INSTANA_DOWNLOAD_KEY}",
              "email": ""
            }
          }
        }
    kind: Secret
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/name: instana
      name: instana-registry
      namespace: instana-units
    type: kubernetes.io/dockerconfigjson
    EOF
    ```

    > (4) `instana-core.yaml`
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/secrets/instana-core.yaml
    apiVersion: v1
    kind: Secret
    type: Generic
    metadata:
      labels:
        app.kubernetes.io/name: instana
      name: instana-core
      namespace: instana-core
    stringData:
      config.yaml: |
        # The initial password for the admin user
        adminPassword: ${INSTANA_ADMINPASSWORD}
        # Diffie-Hellman parameters to use
        dhParams: |
    $(cat ${DEPLOY_LOCAL_WORKDIR}/dhparams.pem | sed 's/^/      /g')
        # The download key you received from us
        downloadKey: ${INSTANA_DOWNLOAD_KEY}
        # The sales key you received from us
        salesKey: ${INSTANA_SALES_KEY}
        # Seed for creating crypto tokens. Pick a random 12 char string
        tokenSecret: mytokensecret
        # Configuration for raw spans storage
        #rawSpansStorageConfig:
        #  pvcConfig:
        #    resources:
        #      requests:
        #        storage: 2Gi
        #    storageClassName: nfs-client
        # SAML/OIDC configuration
    
        # ------ Optional, input serviceProviderConfig when you configure SAML/OIDC -------
        #serviceProviderConfig:
        #  # Password for the key/cert file
        #  keyPassword: mykeypass
        #  # The combined key/cert file
        #  pem: |
        #    -----BEGIN RSA PRIVATE KEY-----
        #    <snip/>
        #    -----END RSA PRIVATE KEY-----
        #    -----BEGIN CERTIFICATE-----
        #    <snip/>
        #    -----END CERTIFICATE-----
        # ----------------------------------------------------------------------------------
    EOF
    ```

    > (5) `${INSTANA_TENANT}-${INSTANA_UNIT}.yaml`
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/secrets/${INSTANA_TENANT}-${INSTANA_UNIT}.yaml
    apiVersion: v1
    kind: Secret
    type: Opaque
    metadata:
      labels:
        app.kubernetes.io/name: instana
      name: ${INSTANA_TENANT}-${INSTANA_UNIT}
      namespace: instana-units
    stringData:
      config.yaml: |
        # The Instana license. an be a plain text string or a JSON array.
        license: '${INSTANA_LICENSE}'
        # A list of agent keys. Currently, only the first key ist used.
        # Future versions will allow multiple keys to be specified for better key management.
        agentKeys:
          - ${INSTANA_DOWNLOAD_KEY}
    EOF
    ```

1. Render CRs manifest
    ```
    mkdir -p ${DEPLOY_LOCAL_WORKDIR}/out/crs/
    ```
    > (1) `core_instana-core_instana-core.yaml`
    
    Export the distributed datastore host set variables before you create the manifest for instana-core CR:
    ```
    export CASSANDRA_NODES=\"bstore-1.fyre.ibm.com\",\"bstore-2.fyre.ibm.com\"
    export COCKROACHDB_NODES=\"bstore-2.fyre.ibm.com\",\"bstore-3.fyre.ibm.com\"
    export CLICKHOUSE_NODES=\"bstore-1.fyre.ibm.com\",\"bstore-2.fyre.ibm.com\"
    export ELASTICSEARCH_NODES=\"bstore-1.fyre.ibm.com\",\"bstore-2.fyre.ibm.com\",\"bstore-3.fyre.ibm.com\"
    export KAFKA_NODES=\"bstore-1.fyre.ibm.com\",\"bstore-3.fyre.ibm.com\"
    export ZOOKEEPER_NODES=\"bstore-1.fyre.ibm.com\",\"bstore-2.fyre.ibm.com\",\"bstore-3.fyre.ibm.com\"
    ```
    
    You can choose either `v1beta1` or `v1beta2` spec for the CR. Be aware that the `v1beta1` will be deprecated after `v225`. 
    
    v1beta1:
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/crs/core_instana-core_instana-core.yaml
    apiVersion: instana.io/v1beta1
    kind: Core
    metadata:
      creationTimestamp: null
      name: instana-core
      namespace: instana-core
    spec:
      agentAcceptorConfig:
        host: ${INSTANA_FQDN}
        port: 30950
      baseDomain: ${INSTANA_FQDN}
      datastoreConfigs:
        - addresses: [ ${CASSANDRA_NODES} ]
          ports:
            - name: tcp
              port: 9042
          schemas:
            - profiles
            - spans
            - metrics
          type: cassandra
        - addresses: [ ${COCKROACHDB_NODES} ]
          ports:
            - name: tcp
              port: 26257
          schemas:
            - butlerdb
            - tenantdb
            - sales
          type: cockroachdb
        - addresses: [ ${CLICKHOUSE_NODES} ]
          clusterName: local
          ports:
            - name: tcp
              port: 9000
            - name: http
              port: 8123
          schemas:
            - application
            - logs
          type: clickhouse
        - addresses: [ ${ELASTICSEARCH_NODES} ]
          clusterName: onprem_onprem
          ports:
            - name: tcp
              port: 9300
            - name: http
              port: 9200
          schemas:
            - metadata_ng
          type: elasticsearch
        - addresses: [ ${KAFKA_NODES} ]
          ports:
            - name: tcp
              port: 9092
          schemas:
            - global-ingress
            - ingress
          type: kafka
      emailConfig:
        smtpConfig:
          from: <user_name>>
          host: <smtp_host_name>
          port: 465
          useSSL: true
      imageConfig:
        registry: containers.instana.io
      rawSpansStorageConfig:
        pvcConfig:
          resources:
            requests:
              storage: 2Gi
          storageClassName: nfs-client
      resourceProfile: small
      imagePullSecrets:
        - name: instana-registry
      featureFlags:
        - name: feature.logging.enabled
          enabled: false
    status:
      lastUpdate: null
    EOF
    ```
    
    v1beta2:
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/crs/core_instana-core_instana-core.yaml
    apiVersion: instana.io/v1beta2
    kind: Core
    metadata:
      creationTimestamp: null
      name: instana-core
      namespace: instana-core
    spec:
      agentAcceptorConfig:
        host: ${INSTANA_FQDN}
        port: 30950
      baseDomain: ${INSTANA_FQDN}
      datastoreConfigs:
    
        clickhouseConfigs:
          - hosts: [ ${CLICKHOUSE_NODES} ]
            clusterName: local
            ports:
              - name: tcp
                port: 9000
              - name: http
                port: 8123
            schemas:
              - application
              - logs
    
        cassandraConfigs:
          - hosts: [ ${CASSANDRA_NODES} ]
            ports:
              - name: tcp
                port: 9042
            keyspaces:
              - profiles
              - spans
              - metrics
    
        cockroachdbConfigs:
          - hosts: [ ${COCKROACHDB_NODES} ]
            ports:
              - name: tcp
                port: 26257
            databases:
              - butlerdb
              - tenantdb
              - sales
    
        elasticsearchConfig:
          hosts: [ ${ELASTICSEARCH_NODES} ]
          clusterName: onprem_onprem
          ports:
            - name: tcp
              port: 9300
            - name: http
              port: 9200
    
        kafkaConfig:
          hosts: [ ${KAFKA_NODES} ]
          ports:
            - name: tcp
              port: 9092
      emailConfig:
        smtpConfig:
          from: <user_name>>
          host: <smtp_host_name>
          port: 465
          useSSL: true
      imageConfig:
        registry: containers.instana.io
      rawSpansStorageConfig:
        pvcConfig:
          resources:
            requests:
              storage: 2Gi
          storageClassName: nfs-client
      resourceProfile: small
      imagePullSecrets:
        - name: instana-registry
      featureFlags:
        - name: feature.logging.enabled
          enabled: false
    status:
      lastUpdate: null
    EOF
    ```
    
    

    > (2) `unit_instana-units_${INSTANA_TENANT}-${INSTANA_UNIT}.yaml`
      
    You can choose either `v1beta1` or `v1beta2` spec for the CR. Be aware that the `v1beta1` will be deprecated after `v225`
    
    v1beta1:
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/crs/unit_instana-units_${INSTANA_TENANT}-${INSTANA_UNIT}.yaml
    apiVersion: instana.io/v1beta1
    kind: Unit
    metadata:
      creationTimestamp: null
      name: ${INSTANA_TENANT}-${INSTANA_UNIT}
      namespace: instana-units
    spec:
      coreName: instana-core
      coreNamespace: instana-core
      initialAgentKey: ${INSTANA_DOWNLOAD_KEY}
      resourceProfile: small
      tenantName: ${INSTANA_TENANT}
      unitName: ${INSTANA_UNIT}
    status:
      lastUpdate: null
    EOF
    ```
    
    v1beta2:
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/crs/unit_instana-units_${INSTANA_TENANT}-${INSTANA_UNIT}.yaml
    apiVersion: instana.io/v1beta2
    kind: Unit
    metadata:
      creationTimestamp: null
      name: ${INSTANA_TENANT}-${INSTANA_UNIT}
      namespace: instana-units
    spec:
      coreName: instana-core
      coreNamespace: instana-core
      resourceProfile: small
      tenantName: ${INSTANA_TENANT}
      unitName: ${INSTANA_UNIT}
    status:
      lastUpdate: null
    EOF  
    ```

1. Render PVC manifest (required from v221)
    ```
    mkdir -p ${DEPLOY_LOCAL_WORKDIR}/out/pvcs/
    ```
    
    > (1) `appdata-writer-pvc.yaml`
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/pvcs/appdata-writer-pvc.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: appdata-writer
      namespace: instana-core
      labels:
        app.kubernetes.io/component: appdata-writer
        app.kubernetes.io/name: instana
        app.kubernetes.io/part-of: core
        instana.io/group: service
      finalizers:
        - kubernetes.io/pvc-protection
      selfLink: /api/v1/namespaces/instana-core/persistentvolumeclaims/appdata-writer
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 2Gi
      storageClassName: nfs-client
      volumeMode: Filesystem
    EOF
    ```
    
    > (2) `raw-spans-pvc.yaml`
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/pvcs/raw-spans-pvc.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: spans-volume-claim
      namespace: instana-core
      labels:
        app.kubernetes.io/component: appdata-writer
        app.kubernetes.io/name: instana
        app.kubernetes.io/part-of: core
        instana.io/group: service
      finalizers:
        - kubernetes.io/pvc-protection
      selfLink: /api/v1/namespaces/instana-core/persistentvolumeclaims/spans-volume-claim
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 2Gi
      storageClassName: nfs-client
      volumeMode: Filesystem
    EOF
    ```

1. Preform Core & Unit deployment via manifest

    Now you will have all the necessary manifest ready to deploy, check the yaml files:
    ```console
    $ tree ${DEPLOY_LOCAL_WORKDIR}
    /opt/instana/k8s//.work/local/localdev
    ├── dhparams.pem
    ├── out
    │   ├── crs
    │   │   ├── core_instana-core_instana-core.yaml
    │   │   └── unit_instana-units_tenant0-unit0.yaml
    │   ├── pvcs
    │   │   ├── appdata-writer-pvc.yaml
    │   │   └── raw-spans-pvc.yaml
    │   └── secrets
    │       ├── instana-core.yaml
    │       ├── secret_instana-core_instana-registry.yaml
    │       ├── secret_instana-core_instana-tls.yaml
    │       ├── secret_instana-units_instana-registry.yaml
    │       └── tenant0-unit0.yaml
    ├── tls.crt
    └── tls.key
    ```


    Run the following command to start the deployment.
    ```
    kubectl create namespace instana-core
    kubectl label namespaces instana-core app.kubernetes.io/name=instana-core --overwrite=true
    kubectl create namespace instana-units
    kubectl label namespaces instana-units app.kubernetes.io/name=instana-units --overwrite=true
    kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/secrets
    kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/crs/
    kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/pvcs/
    ```
    sample output:
    ```console
    [root@bstan-1 k8s]# kubectl create namespace instana-core
    namespace/instana-core created
    [root@bstan-1 k8s]# kubectl label namespaces instana-core app.kubernetes.io/name=instana-core --overwrite=true
    namespace/instana-core labeled
    [root@bstan-1 k8s]# kubectl create namespace instana-units
    namespace/instana-units created
    [root@bstan-1 k8s]# kubectl label namespaces instana-units app.kubernetes.io/name=instana-units --overwrite=true
    namespace/instana-units labeled
    [root@bstan-1 k8s]# kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/secrets
    secret/instana-core created
    secret/instana-registry created
    secret/instana-tls created
    secret/instana-registry created
    secret/tenant0-unit0 created
    [root@bstan-1 k8s]# kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/crs/
    core.instana.io/instana-core created
    unit.instana.io/tenant0-unit0 created
    [root@bstan-1 k8s]# kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/pvcs/
    persistentvolumeclaim/appdata-writer created
    persistentvolumeclaim/spans-volume-claim created
    ```

### Configure ingress to expose gateway & acceptor

1. Edit ingress-nginx daemonset`kubectl edit ds ingress-nginx-controller -n ingress-nginx` , add args `--enable-ssl-passthrough`
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
    > To expose Instana UI service, we need enable ssl passthrough for ingress contoller, this was disabled by default during ingress controller deployment via kubespray. 

1. Create ingress for gateway service:
    Here are the steps to create the necessary secrets:
    ```
    mkdir -p ${DEPLOY_LOCAL_WORKDIR}/out/ingress/
    ```
    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/ingress/gateway-ingress.yaml
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
        - host: "${INSTANA_FQDN}"
          http:
            paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: gateway
                  port:
                    number: 8443
        - host: "${INSTANA_UNIT}-${INSTANA_TENANT}.${INSTANA_FQDN}"
          http:
            paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: gateway
                  port:
                    number: 8443
    EOF
    ```
    ```
    kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/ingress/gateway-ingress.yaml
    ```

1. Create nodeport service to expose acceptor (deprecated, will revise to use ingress)

    ```
    cat << EOF > ${DEPLOY_LOCAL_WORKDIR}/out/ingress/acceptor-nodeport.yaml
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
    kubectl apply -f ${DEPLOY_LOCAL_WORKDIR}/out/ingress/acceptor-nodeport.yaml
    ```




### Access backend UI

Make sure your dns service can resolve fqdn `${INSTANA_FQDN}` and `${INSTANA_UNIT}-${INSTANA_TENANT}.${INSTANA_FQDN}` to any of your cluster node ip.
> In this example, you can use any of these 3 ip: 9.112.255.202, 9.112.255.165, 9.112.255.167

To Access the Instana Backend UI:
```sh
https://${INSTANA_FQDN}    # Here is https://bstan-1.fyre.ibm.com
admin@instana.local        # Login with the password set from ${INSTANA_ADMINPASSWORD}
```




## Known Limitation

- Playbooks tested on CentOS 7.9.
- Tags based playbook execution has not finished, only support install,config,stop,clean.
- There are more options for expose the gateway & acceptor services.
- Instana Core & Unit automation playbook not ready.
