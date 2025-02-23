# bare metal lab infrastructure

All clusters in the lab are deployed on a single Bare Metal host running RHEL9.4. It has 991G RAM, 128 phys.cores.

We use libvirt/kvm and sushy tools to emulate the Bare Metal Controller.

A lot of these instructions could be automated. But its just a lab so 🤷.

- [Base Host setup](#Base-Host-setup)
- [Networking Setup on Base Host](#Networking-Setup-on-Base-Host)
- [DNS Base Host](#DNS-Base-Host)
- [ACM Hub Install](#ACM-Hub-Install)
- [ArgoCD / ACM Bootstrap](#ArgoCD-/-ACM-Bootstrap)
- [ACM app-of-apps](#ACM-app-of-apps)
- [Vault Setup](#Vault-Setup)
- [Deploy Cluster MCE Secret](#Deploy-Cluster-MCE-Secret)
- [Deploy MCE 3-Node compact cluster](#Deploy-MCE-3-Node-compact-cluster)
- [Deploy MCE app-of-apps](#Deploy-MCE-app-of-apps)
- [Deploy HCP Spoke Secrets](#Deploy-HCP-Spoke-Secrets)
- [Deploy HCP Spoke worker nodes](#Deploy-HCP-Spoke-worker-nodes)
- [HAProxy on Base Host](#HAProxy-on-Base-Host)
- [Adding and Deleting HCP Clusters](#Adding-and-Deleting-HCP-Clusters)

## Base Host setup

Install libvirt and tools as root.

```bash
dnf install libvirt virt-install virt-manager xauth libguestfs wget -y
systemctl enable libvirtd --now
virt-host-validate
```

We are using iptables and podman for sushy.

```bash
dnf install podman net-tools bind-utils iptables-services dnsmasq -y
systemctl enable iptables --now
```

Configure sushy to be able to talk to libvirt.

```bash
sudo mkdir -p /etc/sushy/
cat << "EOF" | sudo tee /etc/sushy/sushy-emulator.conf
SUSHY_EMULATOR_LISTEN_IP = u'0.0.0.0'
SUSHY_EMULATOR_LISTEN_PORT = 8000
SUSHY_EMULATOR_SSL_CERT = None
SUSHY_EMULATOR_SSL_KEY = None
SUSHY_EMULATOR_OS_CLOUD = None
SUSHY_EMULATOR_LIBVIRT_URI = u'qemu:///system'
SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = True
SUSHY_EMULATOR_BOOT_LOADER_MAP = {
    u'UEFI': {
        u'x86_64': u'/usr/share/OVMF/OVMF_CODE.secboot.fd'
    },
    u'Legacy': {
        u'x86_64': None
    }
}
EOF
```

Run sushy emulator.

```bash
export SUSHY_TOOLS_IMAGE=${SUSHY_TOOLS_IMAGE:-"quay.io/metal3-io/sushy-tools"}
sudo podman create --net host --privileged --name sushy-emulator -v "/etc/sushy":/etc/sushy -v "/var/run/libvirt":/var/run/libvirt "${SUSHY_TOOLS_IMAGE}" sushy-emulator -i :: -p 8000 --config /etc/sushy/sushy-emulator.conf
sudo podman start sushy-emulator
```

Test.

```bash
curl http://172.23.3.24:8000/redfish/v1/Managers
```

Install nmstate for OCP installer.

```bash
dnf install /usr/bin/nmstatectl -y
```

Create a default pool for libvirt.

```bash
virsh pool-list
virsh pool-define-as --name default --type dir --target /var/lib/libvirt/images/default
virsh pool-autostart default
virsh pool-start default
```

We have a large amount of disk mounted to `/var/lib/libvirt/images/` - you can also use `thin-lvm` here if needed. All VM's will be installed and run from qcow/iso images installed to here.

## Networking Setup on Base Host

We will use qemu network to keep the lab isolated from others. The downside with not using an external dns server is that any changes to dns you must stop vm's, redefine network, restart vm's.

This has the config for all our OCP clusters pre-defined.

```xml
cat <<EOF > /etc/libvirt/qemu/networks/sno.xml
<network>
  <name>sno</name>
  <uuid>fc23191f-de21-4bf5-774b-98711b9f3d9e</uuid>
  <forward mode='nat' size='1500'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:22:4d:3a'/>
  <domain name='sno.redhatlabs.dev'/>
  <ip family='ipv4' address='192.168.130.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.130.20' end='192.168.130.254'/>
      <host mac='52:54:00:22:4d:4a' name='acm' ip='192.168.130.10'/>
      <host mac='52:54:00:22:4d:4b' name='master-0' ip='192.168.130.11'/>
      <host mac='52:54:00:22:4d:4c' name='master-1' ip='192.168.130.12'/>
      <host mac='52:54:00:22:4d:4d' name='master-2' ip='192.168.130.13'/>
      <host mac='52:54:00:22:4d:4e' name='worker-0' ip='192.168.130.14'/>
      <host mac='52:54:00:22:4d:4f' name='worker-1' ip='192.168.130.15'/>
      <host mac='52:54:00:22:4d:5a' name='worker-2' ip='192.168.130.16'/>
      <host mac='52:54:00:22:4d:5b' name='worker-3' ip='192.168.130.17'/>
    </dhcp>
  </ip>
</network>
EOF
```

Create and start the network.

```bash
virsh net-define /etc/libvirt/qemu/networks/sno.xml
virsh net-start sno
virsh net-autostart sno
```

Any changes you must destroy/start vm's.

```bash
virsh net-destroy sno
virsh net-undefine sno
```

## DNS Base Host

Enable dnsmasq through NetworkManager.

```bash
cat << EOF > /etc/NetworkManager/conf.d/00-use-dnsmasq.conf
[main]
dns=dnsmasq
EOF
```

Define hosts - the format is the same as /etc/hosts

```bash
cat << EOF > /etc/sno.redhatlabs.dev.dnsmasq.hosts
##### SNO
192.168.130.10     api.acm.sno.redhatlabs.dev
192.168.130.10     api-int.acm.sno.redhatlabs.dev
192.168.130.10     oauth-openshift.apps.acm.sno.redhatlabs.dev
192.168.130.10     console-openshift-console.apps.acm.sno.redhatlabs.dev
192.168.130.10     canary-openshift-ingress-canary.apps.acm.sno.redhatlabs.dev
192.168.130.10     grafana-openshift-monitoring.apps.acm.sno.redhatlabs.dev
192.168.130.10     thanos-querier-openshift-monitoring.apps.acm.sno.redhatlabs.dev
192.168.130.10     assisted-image-service-multicluster-engine.apps.acm.sno.redhatlabs.dev
192.168.130.10     assisted-service-multicluster-engine.apps.acm.sno.redhatlabs.dev
##### MCE
192.168.130.11     api-int.mce.sno.redhatlabs.dev
192.168.130.100    api.mce.sno.redhatlabs.dev
192.168.130.101    console-openshift-console.apps.mce.sno.redhatlabs.dev
192.168.130.101    oauth-openshift.apps.mce.sno.redhatlabs.dev
192.168.130.101    canary-openshift-ingress-canary.apps.mce.sno.redhatlabs.dev
192.168.130.101    oauth-openshift.apps.mce.sno.redhatlabs.dev
192.168.130.101    assisted-image-service-multicluster-engine.apps.mce.sno.redhatlabs.dev
192.168.130.101    assisted-service-multicluster-engine.apps.mce.sno.redhatlabs.dev
192.168.130.101    oauth-hosted-hcp-1.apps.mce.sno.redhatlabs.dev
192.168.130.101    oauth-hosted-hcp-2.apps.mce.sno.redhatlabs.dev
192.168.130.101    oauth-hosted-hcp-3.apps.mce.sno.redhatlabs.dev
192.168.130.101    oauth-hosted-hcp-4.apps.mce.sno.redhatlabs.dev
##### HCP
192.168.130.14     console-openshift-console.apps.hcp-1.sno.redhatlabs.dev
192.168.130.14     canary-openshift-ingress-canary.apps.hcp-1.sno.redhatlabs.dev
192.168.130.17     console-openshift-console.apps.hcp-2.sno.redhatlabs.dev
192.168.130.17     canary-openshift-ingress-canary.apps.hcp-2.sno.redhatlabs.dev
EOF
```

Create config.

```bash
cat << EOF > /etc/NetworkManager/dnsmasq.d/01-DNS-redhatlabs.dev.conf
# local dns
local=/sno.redhatlabs.dev/
addn-hosts=/etc/sno.redhatlabs.dev.dnsmasq.hosts

domain-needed
bogus-priv

# Automatically add <domain> to simple names in a hosts-file.
expand-hosts

# Upstream public
no-poll
server=172.23.3.254
server=1.1.1.1
EOF
```

Restart NetworkManager to pick up changes.

```bash
systemctl restart NetworkManager
```

## ACM Hub Install

We bootstrap the ACM Hub from the command line. The other clusters (MCE, HCP Spokes) are deployed via GitOps from the ACM Hub.

Download the installer and oc.

```bash
OPENSHIFT_VERSION=4.17.0
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OPENSHIFT_VERSION}/openshift-install-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OPENSHIFT_VERSION}/openshift-client-linux.tar.gz
tar xzvf openshift-install-linux.tar.gz
chmod 755 openshift-install
tar xzvf openshift-client-linux.tar.gz
chmod 755 oc kubectl
mv oc ~/bin/
mv kubectl ~/bin/
```

We are deploying a SNO instance for the ACM Hub in the lab. Replace with your `pullSecret` and public `sshKey`.

```yaml
cat << 'EOF' > acm-install-config.yaml
---
apiVersion: v1
baseDomain: sno.redhatlabs.dev
metadata:
  name: acm
networking:
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.130.0/24
  serviceNetwork:
  - 172.30.0.0/16
compute:
- name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  name: master
  replicas: 1
platform:
  none: {}
pullSecret: '{"auths"...'
sshKey: 'ssh ...'
EOF
```

AgentConfig for ACM Hub.

```yaml
cat > acm-agent-config.yaml << EOF
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: acm
rendezvousIP: 192.168.130.10
hosts: 
  - hostname: acm
    interfaces:
      - name: enp1s0
        macAddress: 52:54:00:22:4d:4a
    rootDeviceHints: 
      deviceName: /dev/vda
    networkConfig: 
      interfaces:
        - name: enp1s0
          type: ethernet
          state: up
          mac-address: 52:54:00:22:4d:4a
          ipv4:
            enabled: true
            address:
              - ip: 192.168.130.10
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.130.1
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.130.1
            next-hop-interface: enp1s0
            table-id: 254
          - destination: 192.168.86.0/0
            next-hop-address: 192.168.130.1
            next-hop-interface: enp1s0
            table-id: 254
EOF
```

Create an install directory, copy in config.

```bash
mkdir acm-cluster
cp acm-install-config.yaml acm-cluster/install-config.yaml
cp acm-agent-config.yaml acm-cluster/agent-config.yaml
```

Create iso for install.

```bash
./openshift-install --dir acm-cluster agent create image
```

Copy it for libvirt.

```bash
sudo cp acm-cluster/agent.x86_64.iso /var/lib/libvirt/images/acm-sno-agent.x86_64.iso
sudo chown qemu:qemu /var/lib/libvirt/images/acm-sno-agent.x86_64.iso
sudo restorecon -rv /var/lib/libvirt/images/acm-sno-agent.x86_64.iso
```

Setup DNS for ACM on Base Host.

```bash
# vi /etc/hosts
192.168.130.10     api.acm.sno.redhatlabs.dev
192.168.130.10     oauth-openshift.apps.acm.sno.redhatlabs.dev
192.168.130.10     console-openshift-console.apps.acm.sno.redhatlabs.dev
192.168.130.10     grafana-openshift-monitoring.apps.acm.sno.redhatlabs.dev
192.168.130.10     thanos-querier-openshift-monitoring.apps.acm.sno.redhatlabs.dev
```

Create `qcow2` images for ACM SNO vm.

```bash
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/acm-sno.qcow2 120G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/acm-sno-extra.qcow2 250G
```

Install VM as root.

```bash
VM_NAME=acm
NET_NAME=sno
DISK=/var/lib/libvirt/images/acm-sno.qcow2,format=qcow2
OS_VARIANT="fedora-coreos-stable"
RAM_MB="65536"
CPU_CORE="12"
RHCOS_ISO=/var/lib/libvirt/images/acm-sno-agent.x86_64.iso
MAC=52:54:00:22:4d:4a

rm -f nohup.out
nohup virt-install \
    --virt-type kvm \
    --connect qemu:///system \
    -n "${VM_NAME}" \
    -r "${RAM_MB}" \
    --vcpus "${CPU_CORE}" \
    --os-variant="${OS_VARIANT}" \
    --cpu=host-passthrough,cache.mode=passthrough \
    --import \
    --network=network:${NET_NAME},mac=${MAC},driver.queues=4 \
    --events on_reboot=restart \
    --cdrom "${RHCOS_ISO}" \
    --disk path=${DISK},io=io_uring,cache='writeback',discard='unmap' \
    --boot hd,cdrom \
    --wait=-1 &
```

Watch OCP install progress.

```bash
./openshift-install --dir acm-cluster agent wait-for bootstrap-complete --log-level=debug
./openshift-install --dir acm-cluster agent wait-for install-complete --log-level=debug
```

Add in a secondary disk for ACM that we will use for LVM storage (from virt-manager or cli).

```bash
virsh edit acm
```

```xml
   <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='writeback' discard='unmap'/>
      <source file='/var/lib/libvirt/images/acm-sno-extra.qcow2'/>
      <target dev='vdb' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
    </disk>
```

## ArgoCD / ACM Bootstrap

Once ACM Hub deployed, install an admin user.

```bash
./users.sh
```

Bootstrap ArgoCD and ACM.

```bash
./install.sh -d
```

## ACM app-of-apps

[acm-app-of-apps.yaml](gitops/app-of-apps/acm-app-of-apps.yaml) defines all of the Applications in the ACM Hub cluster.

When learning what these apps do - you can rename `*.yaml` files to `*.yaml.undeployed` in the folder and add in the apps as you need.

## Vault Setup

First step is to prepare Vault so we can use it for secrets [- online doco here](https://eformat.github.io/rainforest-docs/#/2-platform-work/3-secrets).

Initialize the Vault.

```bash
oc -n vault exec -ti vault-0 -- vault operator init -key-threshold=1 -key-shares=1
```

Use the given secrets:

```bash
export UNSEAL_KEY=
export ROOT_TOKEN=
```

Unlock the vault.

```bash
oc -n vault exec -ti vault-0 -- vault operator unseal $UNSEAL_KEY
oc apply -f ~/tmp/vault-unseal-cronjob.yaml
```

Create a cronjob so you don't have to continually unlock the vault when the SNO node is rebooted.

```bash
oc apply -f ~/tmp/vault-unseal-cronjob.yaml
```

Setup vault login.

```bash
export VAULT_ROUTE=vault.apps.acm.sno.redhatlabs.dev
export VAULT_ADDR=https://${VAULT_ROUTE}
export VAULT_SKIP_VERIFY=true
```

Connect via cli.

```bash
vault login token=${ROOT_TOKEN}
```

Follow the docs - setup ArgoCD SA using Kubernetes Auth and enable KV2.

```bash
export APP_NAME=vault
export PROJECT_NAME=openshift-gitops
export CLUSTER_DOMAIN=apps.acm.sno.redhatlabs.dev

vault auth enable -path=$CLUSTER_DOMAIN-${PROJECT_NAME} kubernetes

export MOUNT_ACCESSOR=$(vault auth list -format=json | jq -r ".\"$CLUSTER_DOMAIN-$PROJECT_NAME/\".accessor")

vault policy write $CLUSTER_DOMAIN-$PROJECT_NAME-kv-read -<< EOF
path "kv/data/{{identity.entity.aliases.$MOUNT_ACCESSOR.metadata.service_account_namespace}}/*" {
capabilities=["read","list"]
}
EOF

vault secrets enable -path=kv/ -version=2 kv

vault write auth/$CLUSTER_DOMAIN-$PROJECT_NAME/role/$APP_NAME \
bound_service_account_names=$APP_NAME \
bound_service_account_namespaces=$PROJECT_NAME \
policies=$CLUSTER_DOMAIN-$PROJECT_NAME-kv-read \
period=120s

vault write auth/$CLUSTER_DOMAIN-${PROJECT_NAME}/config \
kubernetes_host="$(oc whoami --show-server)"
```

Create ArgoCD connection information.

```yaml
cat <<EOF | oc apply -f-
kind: Secret
apiVersion: v1
metadata:
  name: team-avp-credentials
  namespace: openshift-gitops
stringData:
  AVP_AUTH_TYPE: "k8s"
  AVP_K8S_MOUNT_PATH: "auth/apps.acm.sno.redhatlabs.dev-openshift-gitops"
  AVP_K8S_ROLE: "vault"
  AVP_TYPE: "vault"
  VAULT_ADDR: "https://vault.vault.svc:8200"
  VAULT_SKIP_VERIFY: "true"
type: Opaque
EOF
```

## Deploy Cluster MCE Secret

We need a Vault secret to deploy MCE 3-Node from ACM.

```bash
export PROJECT_NAME=openshift-gitops
export APP_NAME=cluster-mce
export PULL_SECRET=$(cat ~/tmp/pull-secret)
export HTPASSWORD=YWRtaW46JDJ5JDA1JEpDa25jcWh1eVhOenVyZ3VFdXBUeGVFamJPREk1bWI1UU1Sb3g3VlB0R0pZQ05nOVlDSkZpCg==
export BMC_PASSWORD=admin
export BMC_USERNAME=admin

vault kv put kv/$PROJECT_NAME/$APP_NAME \
  PULL_SECRET="${PULL_SECRET}" \
  HTPASSWORD="${HTPASSWORD}" \
  BMC_PASSWORD="${BMC_PASSWORD}" \
  BMC_USERNAME="${BMC_USERNAME}"
```

## Deploy MCE 3-Node compact cluster

The [MCE cluster](gitops/applications/managed-clusters/mce) is deployed from GitOps. This cluster hosts our HCP Spoke control planes.

We need to setup VM's for MCE cluster first. Once the [InfraEnv](gitops/applications/managed-clusters/mce/input/08-infraenv.yaml) is deployed into ACM, an ISO is created.

Broswe in ACM > Infrastructure > Host Inventory > mce. Select Add Hosts > With Discovery ISO.

```bash
wget -O discovery.iso 'https://assisted-image-service-multicluster-engine.apps.acm.sno.redhatlabs.dev/byapikey/.../4.17/x86_64/minimal.iso
```

Copy for libvirt.

```bash
sudo cp discovery.iso /var/lib/libvirt/images/secrhcos-live.x86_64.iso
sudo chown qemu:qemu /var/lib/libvirt/images/mce-rhcos-live.x86_64.iso
sudo restorecon -rv /var/lib/libvirt/images/mce-rhcos-live.x86_64.iso
```

Now install 3x VM's for 3-Node ACM MCE cluster.

```bash
VM_NAME=master-0
NET_NAME=sno
DISK=/var/lib/libvirt/images/mce-master-0.qcow2,format=qcow2
OS_VARIANT="fedora-coreos-stable"
RAM_MB="32768"
CPU_CORE="8"
RHCOS_ISO=/var/lib/libvirt/images/mce-rhcos-live.x86_64.iso
MAC=52:54:00:22:4d:4b

VM_NAME=master-1
NET_NAME=sno
DISK=/var/lib/libvirt/images/mce-master-1.qcow2,format=qcow2
OS_VARIANT="fedora-coreos-stable"
RAM_MB="32768"
CPU_CORE="8"
RHCOS_ISO=/var/lib/libvirt/images/mce-rhcos-live.x86_64.iso
MAC=52:54:00:22:4d:4c

VM_NAME=master-2
NET_NAME=sno
DISK=/var/lib/libvirt/images/mce-master-2.qcow2,format=qcow2
OS_VARIANT="fedora-coreos-stable"
RAM_MB="32768"
CPU_CORE="8"
RHCOS_ISO=/var/lib/libvirt/images/mce-rhcos-live.x86_64.iso
MAC=52:54:00:22:4d:4d
```

Virt Install Script to run.

```bash
rm -f nohup.out
nohup virt-install \
    --virt-type kvm \
    --connect qemu:///system \
    -n "${VM_NAME}" \
    -r "${RAM_MB}" \
    --vcpus "${CPU_CORE}" \
    --os-variant="${OS_VARIANT}" \
    --cpu=host-passthrough,cache.mode=passthrough \
    --import \
    --network=network:${NET_NAME},mac=${MAC},driver.queues=4 \
    --events on_reboot=restart \
    --cdrom "${RHCOS_ISO}" \
    --disk path=${DISK},io=io_uring,cache='writeback',discard='unmap' \
    --boot hd,cdrom \
    --wait=-1 &
```

For now, we need to update and create the BMH inventory with the UUID's for out vm's (TODO: i think we can simplify this by configuring sushy to use vm names). Edit [09-baremetalhost.yaml](gitops/applications/managed-clusters/mce/input/09-baremetalhost.yaml). Do foreach node:

```bash
# get the UUID
virsh edit master-0
# update the Systems UUID to match
address: redfish-virtualmedia+http://172.23.3.24:8000/redfish/v1/Systems/aa035696-f470-4dd3-93a3-f5418d2743b5
```

The cluster should now install. Follow progress in ACM Hub.

Post install - add in a secondary disk for each 3 Nodes in MCE that we will use for LVM storage (from virt-manager or cli). Note this lvm disk is local only and is *not* replicated across nodes - so only use in a lab (or put nfs on top for RWX).

```bash
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/mce-extra.qcow2 250G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/mce-extra-1.qcow2 250G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/mce-extra-2.qcow2 250G
```

Repeat for each master node.

```bash
virsh edit master-0
```

```xml
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='writeback' discard='unmap'/>
      <source file='/var/lib/libvirt/images/mce-extra.qcow2'/>
      <target dev='vdb' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
    </disk>
```

## Deploy MCE app-of-apps

[mce-app-of-apps.yaml](gitops/app-of-apps/mce-app-of-apps.yaml) defines all of the Applications in the MCE Hub cluster.

When learning what these apps do - you can rename `*.yaml` files to `*.yaml.undeployed` in the folder and add in the apps as you need.

```bash
oc apply gitops/app-of-apps/mce-app-of-apps.yaml
```

## Deploy HCP Spoke Secrets

We need Vault secrets for our HCP clusters to deploy from ACM.

`hcp-1`

```bash
export PROJECT_NAME=openshift-gitops
export APP_NAME=cluster-hcp-1
export PULL_SECRET=$(cat ~/tmp/pull-secret)
export HTPASSWORD=YWRtaW46JDJ5JDA1JEpDa25jcWh1eVhOenVyZ3VFdXBUeGVFamJPREk1bWI1UU1Sb3g3VlB0R0pZQ05nOVlDSkZpCg==
export BMC_PASSWORD=admin
export BMC_USERNAME=admin
export ETCD_DECRYPTION_KEY="8FJKJdNUVpu9LtMmTO20BkVSF0HiFBcDnIJHiA+T2GY="

vault kv put kv/$PROJECT_NAME/$APP_NAME \
  PULL_SECRET="${PULL_SECRET}" \
  ETCD_DECRYPTION_KEY="${ETCD_DECRYPTION_KEY}" \
  HTPASSWORD="${HTPASSWORD}" \
  BMC_PASSWORD="${BMC_PASSWORD}" \
  BMC_USERNAME="${BMC_USERNAME}"
```

`hcp-2`

```bash
export PROJECT_NAME=openshift-gitops
export APP_NAME=cluster-hcp-2
export PULL_SECRET=$(cat ~/tmp/pull-secret)
export HTPASSWORD=YWRtaW46JDJ5JDA1JEpDa25jcWh1eVhOenVyZ3VFdXBUeGVFamJPREk1bWI1UU1Sb3g3VlB0R0pZQ05nOVlDSkZpCg==
export BMC_PASSWORD=admin
export BMC_USERNAME=admin
export ETCD_DECRYPTION_KEY="8FJKJdNUVpu9LtMmTO20BkVSF0HiFBcDnIJHiA+T2GY="

vault kv put kv/$PROJECT_NAME/$APP_NAME \
  PULL_SECRET="${PULL_SECRET}" \
  ETCD_DECRYPTION_KEY="${ETCD_DECRYPTION_KEY}" \
  HTPASSWORD="${HTPASSWORD}" \
  BMC_PASSWORD="${BMC_PASSWORD}" \
  BMC_USERNAME="${BMC_USERNAME}"
```

## Deploy HCP Spoke worker nodes

We use GitOps for the HCP clusters. An easy way to render this initially is by using the cli. For example:

```bash
export CLUSTER_NAME=hcp-1
export AGENT_NAMESPACE=hcp-1
export PULL_SECRET=~/tmp/pull-secret
export SSH_KEY=~/.ssh/id_rsa.pub
export MEM="8Gi"
export CPU="8"
export WORKER_COUNT="1"
export CLUSTER_CIDR="10.128.0.0/14"

hcp create cluster agent \
--name $CLUSTER_NAME \
--agent-namespace $AGENT_NAMESPACE \
--node-pool-replicas $WORKER_COUNT \
--control-plane-availability-policy SingleReplica \
--pull-secret $PULL_SECRET \
--ssh-key $SSH_KEY \
--release-image=quay.io/openshift-release-dev/ocp-release@sha256:115bba6836b9feffb81ad9101791619edd5f19d333580b7f62bd6721eeda82d2 \
--cluster-cidr $CLUSTER_CIDR \
--render
```

The output is checked into [here](gitops/applications/managed-clusters/hcp-1/input).

We similarly need to create VM's for our Bare Metal install this time using the ISO from the MCE cluster [IfraEnv](gitops/applications/managed-clusters/hardware-inventory/input/infra-env.yaml)

Broswe in MCE > Infrastructure > Host Inventory > bare-metal. Select Add Hosts > With Discovery ISO. Note that 4.16 is max supported HCP version at time of writing.

```bash
wget -O mce-discovery.x86_64.iso 'https://assisted-image-service-multicluster-engine.apps.mce.sno.redhatlabs.dev/byapikey/.../4.16/x86_64/minimal.iso
```

Copy for libvirt.

```bash
sudo cp mce-discovery.x86_64.iso /var/lib/libvirt/images/mce-discovery.x86_64.iso
sudo chown qemu:qemu /var/lib/libvirt/images/mce-discovery.x86_64.iso
sudo restorecon -rv /var/lib/libvirt/images/mce-discovery.x86_64.iso
```

Now install worker node VM's for our HCP clusters.

```bash
VM_NAME=worker-0
NET_NAME=sno
DISK=/var/lib/libvirt/images/mce-worker-0.qcow2,format=qcow2
OS_VARIANT="fedora-coreos-stable"
RAM_MB="32768"
CPU_CORE="8"
RHCOS_ISO=/var/lib/libvirt/images/mce-discovery.x86_64.iso
MAC=52:54:00:22:4d:4e

VM_NAME=worker-1
NET_NAME=sno
DISK=/var/lib/libvirt/images/mce-worker-1.qcow2,format=qcow2
OS_VARIANT="fedora-coreos-stable"
RAM_MB="32768"
CPU_CORE="8"
RHCOS_ISO=/var/lib/libvirt/images/mce-discovery.x86_64.iso
MAC=52:54:00:22:4d:4f

VM_NAME=worker-2
NET_NAME=sno
DISK=/var/lib/libvirt/images/mce-worker-2.qcow2,format=qcow2
OS_VARIANT="fedora-coreos-stable"
RAM_MB="32768"
CPU_CORE="8"
RHCOS_ISO=/var/lib/libvirt/images/mce-discovery.x86_64.iso
MAC=52:54:00:22:4d:5a

VM_NAME=worker-3
NET_NAME=sno
DISK=/var/lib/libvirt/images/mce-worker-3.qcow2,format=qcow2
OS_VARIANT="fedora-coreos-stable"
RAM_MB="32768"
CPU_CORE="8"
RHCOS_ISO=/var/lib/libvirt/images/mce-discovery.x86_64.iso
MAC=52:54:00:22:4d:5b
```

Virt Install Script to run.

```bash
rm -f nohup.out
nohup virt-install \
    --virt-type kvm \
    --connect qemu:///system \
    -n "${VM_NAME}" \
    -r "${RAM_MB}" \
    --vcpus "${CPU_CORE}" \
    --os-variant="${OS_VARIANT}" \
    --cpu=host-passthrough,cache.mode=passthrough \
    --import \
    --network=network:${NET_NAME},mac=${MAC},driver.queues=4 \
    --events on_reboot=restart \
    --cdrom "${RHCOS_ISO}" \
    --disk path=${DISK},io=io_uring,cache='writeback',discard='unmap' \
    --boot hd,cdrom \
    --wait=-1 &
```

We have x2 HCP Spoke clusters defined in `managed-clusters` [hcp-1](gitops/applications/managed-clusters/hcp-1/), [hcp-2](gitops/applications/managed-clusters/hcp-2/).

The HCP Spokes should deploy from the MCE cluster.

Once the HCP spokes have deployed - for these to auto-join the ACM Hub we have [this auto-import Policy](gitops/policy-collection/overlays/acm/CM-Configuration-Management/policy-mce-hcp-autoimport.yaml) which is in the product docs. This can take some time to import into ACM - you can check to `oc get klusterlets.operator.open-cluster-management.io -A` in MCE cluster or `oc get discoveredcluster -A` in ACM cluster.

## HAProxy on Base Host

As we only have a single Base Host (which is also our point of entry for all clusters) we use HAproxy as an external load balancer so we can route from the Client.

```bash
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/1.8/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# openshift 4
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          300s
    timeout server          300s
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 20000

# Useful for debugging, dangerous for production
listen stats
    bind :9000
    mode http
    stats enable
    stats uri /

frontend openshift-api-server
    bind *:6443
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    acl is_acm req.ssl_sni -m end .acm.sno.redhatlabs.dev
    acl is_mce req.ssl_sni -m end .mce.sno.redhatlabs.dev
    use_backend api.acm.sno.redhatlabs.dev if is_acm
    use_backend api.mce.sno.redhatlabs.dev if is_mce
    default_backend api.acm.sno.redhatlabs.dev
    option tcplog

backend api.acm.sno.redhatlabs.dev
    balance source
    mode tcp
    server dl380-02 192.168.130.10:6443 check

backend api.mce.sno.redhatlabs.dev
    balance source
    mode tcp
    server master-0 192.168.130.11:6443 check
    server master-1 192.168.130.12:6443 check
    server master-3 192.168.130.13:6443 check

frontend ingress-http
    bind *:80
    mode http
    acl is_acm hdr_dom(host) -i acm.sno.redhatlabs.dev
    acl is_mce hdr_dom(host) -i mce.sno.redhatlabs.dev
    use_backend apps.acm.sno.redhatlabs.dev if is_acm
    use_backend apps.mce.sno.redhatlabs.dev if is_mce
    default_backend apps.acm.sno.redhatlabs.dev
    option tcplog

backend apps.acm.sno.redhatlabs.dev
    balance source
    mode http
    server dl380-02 192.168.130.10:80 check

backend apps.mce.sno.redhatlabs.dev
    balance source
    mode http
    server master-0 192.168.130.11:80 check
    server master-1 192.168.130.12:80 check
    server master-2 192.168.130.13:80 check

frontend ingress-https
    bind *:443
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    acl is_acm req_ssl_sni -m end .acm.sno.redhatlabs.dev
    acl is_mce req_ssl_sni -m end .mce.sno.redhatlabs.dev
    acl is_hcp1 req_ssl_sni -m end .hcp-1.sno.redhatlabs.dev
    acl is_hcp2 req_ssl_sni -m end .hcp-2.sno.redhatlabs.dev
    use_backend ssl.apps.acm.sno.redhatlabs.dev if is_acm
    use_backend ssl.apps.mce.sno.redhatlabs.dev if is_mce
    use_backend ssl.apps.hcp-1.sno.redhatlabs.dev if is_hcp1
    use_backend ssl.apps.hcp-2.sno.redhatlabs.dev if is_hcp2
    default_backend ssl.apps.acm.sno.redhatlabs.dev
    option tcplog

backend ssl.apps.acm.sno.redhatlabs.dev
    balance source
    mode tcp
    server dl380-02 192.168.130.10:443 check

backend ssl.apps.mce.sno.redhatlabs.dev
    balance source
    mode tcp
    server master-0 192.168.130.11:443 check
    server master-1 192.168.130.12:443 check
    server master-2 192.168.130.13:443 check

backend ssl.apps.hcp-1.sno.redhatlabs.dev
    balance source
    mode tcp
    server master-0 192.168.130.14:443 check

backend ssl.apps.hcp-2.sno.redhatlabs.dev
    balance source
    mode tcp
    server master-0 192.168.130.17:443 check
```

## DNS on a Client Host

I'm running from a linux laptop client that VPN's into the Base Host - 172.23.3.24

Since we only have simple dnsmasq in the lab - we point our laptop Clients `/etc/hosts` to the Base Host and let HAProxy do the rest.

```bash
##### ACM
172.23.3.24     api.acm.sno.redhatlabs.dev
172.23.3.24     oauth-openshift.apps.acm.sno.redhatlabs.dev
172.23.3.24     console-openshift-console.apps.acm.sno.redhatlabs.dev
172.23.3.24     grafana-openshift-monitoring.apps.acm.sno.redhatlabs.dev
172.23.3.24     thanos-querier-openshift-monitoring.apps.acm.sno.redhatlabs.dev global-policy-server-openshift-gitops.apps.acm.sno.redhatlabs.dev vault.apps.acm.sno.redhatlabs.dev hcp-cli-download-multicluster-engine.apps.acm.sno.redhatlabs.dev
##### MCE
172.23.3.24     api.mce.sno.redhatlabs.dev
172.23.3.24     oauth-openshift.apps.mce.sno.redhatlabs.dev
172.23.3.24     console-openshift-console.apps.mce.sno.redhatlabs.dev
172.23.3.24     grafana-openshift-monitoring.apps.mce.sno.redhatlabs.dev
172.23.3.24     thanos-querier-openshift-monitoring.apps.mce.sno.redhatlabs.dev hcp-cli-download-multicluster-engine.apps.mce.sno.redhatlabs.dev assisted-image-service-multicluster-engine.apps.mce.sno.redhatlabs.dev
##### HCP
172.23.3.24     console-openshift-console.apps.hcp-1.sno.redhatlabs.dev
172.23.3.24     oauth-hosted-hcp-1.apps.mce.sno.redhatlabs.dev
172.23.3.24     console-openshift-console.apps.hcp-2.sno.redhatlabs.dev
172.23.3.24     oauth-hosted-hcp-2.apps.mce.sno.redhatlabs.dev
```

## Adding and Deleting HCP Clusters

### ADD cluster

To ADD a new hcp cluster.

- copy+paste new gitops folder [gitops/applications/managed-clusters/hcp-{1..4}](gitops/applications/managed-clusters)
- create new appset [gitops/applications/managed-clusters/applications/acm/hcp-{1..4}.yaml](gitops/applications/managed-clusters/applications/acm)
- check this in to git + sync argocd

On Base Host.

- update dnsmasq hosts - `vi etc/sno.redhatlabs.dev.dnsmasq.hosts`
- restart NetworkManager - `systemctl restart NetworkManager`
- update haproxy config - `vi /etc/haproxy/haproxy.cfg`
- restart haproxy - `systemctl restart haproxy`

### DELETE cluster

This could be streamlined some more - but opting to not prune in ArgoCD for now (safety).

- rename cluster appset [gitops/applications/managed-clusters/applications/acm/hcp-{1..4}.yaml](gitops/applications/managed-clusters/applications/acm) to somthing like this - `hcp-1.yaml.undeploy`
- delete hcp cluster from MCE UI
- delete hcp ArgoCD AppSet (not set to prune ATM)
- detach hcp cluster from ACM
- update worker to reboot from cdrom

```bash
virsh edit worker-3
# change this:
# <boot dev='fd'/>
# to this:
# <boot dev='cdrom'/>
virsh destroy worker-3
virsh start worker-3
```

Host should become `Available` again in Inventory.
