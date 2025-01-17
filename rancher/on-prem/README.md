# On-prem Kubernetes Cluster for Rancher

## Prerequisites
* [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).
* [Hardware, network, certificate requirements](./requirements.md).
* rke (rke version: v1.3.10)

## Virtual machines
* Set up VMs.
* Set up [passwordless SSH](../../docs/ssh.md).
* Create copy of `hosts.ini.sample` as `hosts.ini`. Update the IP addresses.

## Wireguard bastion node
_If you already have a Wireguard bastion host then you may skip this step._

* Open required Wireguard ports.
```
ansible-playbook -i hosts.ini wireguard.yaml
```
* Install [Wireguard bastion](https://docs.mosip.io/1.2.0/deployment/wireguard/wireguard-bastion) with enough number of peers.
- Assign peer1 to yourself and set your Wireguard client before working on the cluster.

## Ports
* Open ports on each of the nodes. Update ``vpc_ip`` variable in ``ports.yaml`` to allow access on the VPC IP.
```
ansible-playbook -i hosts.ini ports.yaml
```
* Disable swap _(perhaps not needed as swap is already disabled)_.
```
ansible-playbook -i hosts.ini swap.yaml
```

## Docker
Install docker on all nodes.
```
ansible-playbook -i hosts.ini docker.yaml
```

## RKE cluster setup
* Create a cluster config file. 
    ```
    rke config
    ```
    *  _controlplane, etcd, worker_: At least one of each. For high availability specify _controlplane_, _etc_ on at least two nodes. All notes may be _worker_.
    * Use default _canal_ networking model
    * Keep the _Pod Security Policies_ disabled.
    * Sample configuration options:
    ```
    [+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]:
    [+] Number of Hosts [1]:
    [+] SSH Address of host (1) [none]: <node1-ip>
    [+] SSH Port of host (1) [22]:
    [+] SSH Private Key Path of host (<node1-ip>) [none]:
    [-] You have entered empty SSH key path, trying fetch from SSH key parameter
    [+] SSH Private Key of host (<node1-ip>) [none]:
    [-] You have entered empty SSH key, defaulting to cluster level SSH key: ~/.ssh/id_rsa
    [+] SSH User of host (<node1-ip>) [ubuntu]:
    [+] Is host (<node1-ip>) a Control Plane host (y/n)? [y]: y
    [+] Is host (<node1-ip>) a Worker host (y/n)? [n]: y
    [+] Is host (<node1-ip>) an etcd host (y/n)? [n]: y
    [+] Override Hostname of host (<node1-ip>) [none]: node2
    [+] Internal IP of host (<node1-ip>) [none]:
    [+] Docker socket path on host (<node1-ip>) [/var/run/docker.sock]:
    [+] Network Plugin Type (flannel, calico, weave, canal) [canal]:
    [+] Authentication Strategy [x509]:
    [+] Authorization Mode (rbac, none) [rbac]:
    [+] Kubernetes Docker image [rancher/hyperkube:v1.17.17-rancher1]:
    [+] Cluster domain [cluster.local]:
    [+] Service Cluster IP Range [10.43.0.0/16]:
    [+] Enable PodSecurityPolicy [n]:
    [+] Cluster Network CIDR [10.42.0.0/16]:
    [+] Cluster DNS Service IP [10.43.0.10]:
    [+] Add addon manifest URLs or YAML files [no]:
    ```
* While opting for roles for different nodes follow below points:
  * In case of odd no of total nodes of cluster opt for (n+1/2) nodes with Control plane, etcd host and worker host role and rest of the nodes with Worker host and etcd host role.
  * In case of even no of total nodes of cluster opt for (n/2) nodes with Control Plane, etcd host and worker host role and rest of the node with Worker host and etcd host role.

* Remove the default Ingress install by editing `cluster.yaml`:
    ```
    ingress:
      provider: none
    ```
* For production deplopyments edit the `cluster.yml`, according to this [RKE Cluster Hardening Guide](../../docs/rke-cluster-hardening.md)

* Bring up the cluster:
```
rke up
```
* After successful creation of cluster a `kube_config_cluster.yaml` will get created. Copy the file to `$HOME/.kube` folder.
  ```
  cp kube_config_cluster.yml $HOME/.kube/<cluster_name>_config
  chmod 400 $HOME/.kube/<cluster_name>_config
  ```
* To set this file as global default for `kubectl`, make sure you have a copy of existing `$HOME/.kube/config`. 
```
cp  $HOME/.kube/<cluster_name>_config  $HOME/.kube/config
```
* Alternatively, set `KUBECOFIG` env variable:
```
export KUBECONFIG="$HOME/.kube/<cluster_name>_config
```
* Test
```
kubect get nodes
```

## Ingress controller
Install [Nginx ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/):

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install \                                                                                                             
  ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --version 4.0.18 \
  --create-namespace  \
  -f ingress-nginx.values.yaml
```

## Nginx Reverse Proxy server
* Install [Nginx reverse proxy](./nginx/) that proxies into ingresscontroller on a seperate node.
* Note that TLS termination is done on Nginx which means traffic from Nginx to cluster is HTTP (not HTTPS). A [Wireguard mesh](https://github.com/mosip/mosip-infra/tree/develop/deployment/v3/utils/wireguard-mesh) may be installed to ensure encrypted traffic.  

## RKE Cluster tools:
Below contains some of the RKE cluster related operations in brief:
* Adding/Removing nodes to cluster
  _This step is only required if you have to add/delete nodes to an existing cluster._
  * Copy the ssh keys, setup Docker and open ports as given above.
  * Edit the `cluster.yml` file and add extra nodes with their IPs and roles.
  * Run `rke up --update-only` to bring up the changes to the cluster.
* Removing the whole RKE cluster:
  _This step is only required if you knowingly want to delete existing complete cluster and its dependent binaries._
  * From the folder cotaining `cluster.yml`, `cluster-rke.state`, `kube-config-cluster.yml` files created while cluster creation, run the below>
    ```
    rke remove
    ```
  * Remove the cluster components and  binaries from all the nodes using `rke-recovery.sh`
    ```
    cd utils
    ansible-playbook -i hosts.ini ../../utils/rke-components-delete.yaml
    ```
* Recovering the original `cluster.rkestate` file and `cluster.yml` file created during cluster intialisation.
  _This step is only required if you have lost the `cluster.rkestate` and cluster.yaml file for a RKE cluster._
  * Set the kubeconfig file pointing to respective RKE cluster using any of the way mentioneed in cluster creation section above.
  * run the below script to recover the `cluster.yml` and rkestate.yml
  ```
  ./../../utils/rke-recovery.sh
  ```
  * ``cluster.yml`` and `cluster.rkestate` will be created in the same directory, do preserve the same for further use.
* RKE cluster certificate rotation:
  _This step is only required if your cluster certificates are expired and the same needs to be rotated._
  Note: Whenever you’re trying to rotate certificates, the `cluster.yml` that was used to deploy the Kubernetes cluster is required. You can reference a different location for this file by using the --config option. In case you dont have the `cluster.yml` follow above steps to recover it.
  * Follow the [instruction](https://rancher.com/docs/rke/latest/en/cert-mgmt/#certificate-rotation) to rotate the certificate on need basis.
  __Note:__ Please do check the version info in the provided Rancher docs.
