# instana-onprem-distkit
The automation tool for Instana onprem distributed deployment.

# (temp artifact) Instana On-Prem Distributed K8S deployment + ArgoCD

## Environment


## Kubenertes Cluster Deployment via kubespray

Reference URL: https://kubespray.io/

- Login to a vm which can be access to `cinst` cluster and has docker daemon installed.
```
ssh root@xtool1.fyre.ibm.com
```

- Clone kubespray repo for k8s deployment
```
mkdir -p /root/deploy/k8s/cinst && cd /root/deploy/k8s/cinst
git clone https://github.com/kubernetes-sigs/kubespray
```

- Prepare ssh key for ansible playbook execution

```
ssh-keygen
ssh-copy-id root@cinst-k1.fyre.ibm.com
ssh-copy-id root@cinst-k2.fyre.ibm.com
ssh-copy-id root@cinst-k3.fyre.ibm.com
```

- Create inventory for k8s cluster

`vim kubespray/inventory/sample/inventory.ini` as following (change value to meet your environment):
```
[all]
cinst-k1.fyre.ibm.com ansible_host=cinst-k1.fyre.ibm.com ip=9.112.255.160
cinst-k2.fyre.ibm.com ansible_host=cinst-k2.fyre.ibm.com ip=9.112.255.163
cinst-k3.fyre.ibm.com ansible_host=cinst-k3.fyre.ibm.com ip=9.123.116.50

[kube_control_plane]
cinst-k1.fyre.ibm.com

[etcd]
cinst-k1.fyre.ibm.com

[kube_node]
cinst-k1.fyre.ibm.com
cinst-k2.fyre.ibm.com
cinst-k3.fyre.ibm.com

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```
- Create extra k8s deploy setting
> 现在kubespray默认使用`containerd`作为引擎，image command tool默认使用 `nerdctl` ,但是 `nerdctl` 无法使用`/etc/containerd/config.toml` 中配置的 registry auth, 这导致 playbook 在执行过程中需要 pull docker.io 的 image 时，受限 于 docker pull rate limit 而出现错误, 因此需要将 image command tool 设置成 `crictl` , 并加入 docker.io (kubespray默认使用 `registry-1.docker.io`作为镜像 )的 `containerd_registry_auth`配置, 如果集群节点的网络不好，可考虑添加 proxy 设置(如下):
```
cat << EOF > kubespray/extra-settings.yml
image_command_tool: crictl
containerd_registry_auth:
  - registry: registry-1.docker.io
    auth: "bGxpeGlubjpYaW4xQHhpbmw="
http_proxy: "http://xcoc-proxy.fyre.ibm.com:3128"
https_proxy: "http://xcoc-proxy.fyre.ibm.com:3128"
EOF
```
> 开启需要安装的 addon, 所有选项在 `kubespray/inventory/sample/group_vars/k8s_cluster/addons.yml` 定义
```
cat << EOF > kubespray/extra-addons.yml
helm_enabled: true
ingress_nginx_enabled: true
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


## ArgoCD setup

After kubespray deployed the cluster, argocd automatic installed via addon setting.

- Expose service
You need expose the `argocd-server` service first:
```
kubectl edit svc -n argocd argocd-server
```
change `spec.type` to `NodePort`

- Get admin password
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

- Login to argocd UI
```
https://cinst-k1.fyre.ibm.com:32280
admin/aW2m9X6KekmHwGxb
```
