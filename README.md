# okd-macbook-sno
Experiment to install Single Node OpenShift on an arm64 MacBook

## Build OKD

```bash
crc setup
crc config set disk-size 100
crc config set memory 16384
crc start
export CRC_SSH_KEY=${HOME}/.crc/machines/crc/id_ecdsa
ssh core@127.0.0.1 -p 2222 -i ${CRC_SSH_KEY}
```

Trust The Cert

```bash
openssl s_client -showcerts -connect console-openshift-console.apps-crc.testing </dev/null 2>/dev/null|openssl x509 -outform PEM > /tmp/crc-cert
sudo security add-trusted-cert -d -r trustAsRoot -k "/Library/Keychains/System.keychain" /tmp/crc-cert
```

Install components

```bash
git clone https://github.com/cgruver/okd-centos9-rebuild.git
cd okd-centos9-rebuild

oc apply -f buildconfigs
oc apply -f pipelines
```

## SNO on MacBook

```bash
qemu-img create -f qcow2 ${WORK_DIR}/bootstrap/bootstrap-node.qcow2 ${root_vol}G
    
qemu-system-x86_64 -accel accel=hvf -m ${memory}M -smp ${cpu} -display none -nographic -drive file=${WORK_DIR}/bootstrap/bootstrap-node.qcow2,if=none,id=disk1  -device ide-hd,bus=ide.0,drive=disk1,id=sata0-0-0,bootindex=1 -boot n -netdev vde,id=nic0,sock=/var/run/vde.bridged.${bridge_dev}.ctl -device virtio-net-pci,netdev=nic0,mac=52:54:00:a1:b2:c3
```

```bash
qemu-img create -f qcow2 ${HOME}/local-okd/okd-disk.qcow2 50G
qemu-system-aarch64 -accel accel=hvf -m 16384M -smp 8 -display none -nographic -drive file=${HOME}/local-okd/okd-disk.qcow2,if=none,id=disk1 -device ide-hd,bus=ide.0,drive=disk1,id=sata0-0-0,bootindex=1 -boot n -netdev vde,id=nic0,sock=/var/run/vde.bridged.${bridge_dev}.ctl -device virtio-net-pci,netdev=nic0,mac=52:54:00:a1:b2:c3

```

## Build CLI Tools on MacBook aarch64

```bash
OKD_RELEASE=$(basename $(curl -Ls -o /dev/null -w %{url_effective} https://github.com/openshift/okd/releases/latest))
export OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=registry.ci.openshift.org/origin/release:${OKD_RELEASE}
```

### Build OC on Mac M2

```bash
git clone https://github.com/openshift/oc.git
brew install heimdal
brew install gpgme
export PATH="/opt/homebrew/opt/heimdal/bin:$PATH"
export PATH="/opt/homebrew/opt/heimdal/sbin:$PATH"
export LDFLAGS="-L/opt/homebrew/opt/heimdal/lib"
export CPPFLAGS="-I/opt/homebrew/opt/heimdal/include"
make oc
```

### Build `openshift-install` on Mac M2

```bash
git clone https://github.com/openshift/installer.git 
cd installer
git checkout release-4.11
DEFAULT_ARCH=aarch64 TAGS=okd LDFLAGS="-X github.com/openshift/installer/pkg/releaseimage.default.defaultReleaseImageOriginal=${OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE}" hack/build.sh
```

WORK_DIR=${HOME}/okd-lab/sno-config

mkdir -p ${WORK_DIR}/okd-install-dir
mkdir ${WORK_DIR}/ipxe-work-dir


```bash
if [[ ${BIP} == "true" ]]
then
read -r -d '' SNO_BIP << EOF
BootstrapInPlace:
  InstallationDisk: "--copy-network ${install_dev}"
EOF
fi

cat << EOF > ${WORK_DIR}/install-config-upi.yaml
apiVersion: v1
baseDomain: ${DOMAIN}
metadata:
  name: ${CLUSTER_NAME}
networking:
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: ${CLUSTER_CIDR}
    hostPrefix: 23 
  serviceNetwork: 
  - ${SERVICE_CIDR}
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
  architecture: aarch64
platform:
  none: {}
pullSecret: '${PULL_SECRET_TXT}'
sshKey: ${SSH_KEY}
additionalTrustBundle: |
${NEXUS_CERT}
imageContentSources:
- mirrors:
  - ${PROXY_REGISTRY}/okd
  source: quay.io/openshift/okd
- mirrors:
  - ${PROXY_REGISTRY}/okd
  source: quay.io/openshift/okd-content
${SNO_BIP}
EOF
}

```

```bash
openshift-install --dir=${WORK_DIR}/okd-install-dir create single-node-ignition-config
cp ${WORK_DIR}/okd-install-dir/bootstrap-in-place-for-live-iso.ign ${WORK_DIR}/ipxe-work-dir/sno.ign
```

```bash
cat << EOF > ${WORK_DIR}/ipxe-work-dir/${mac//:/-}-config.yml
variant: fcos
version: 1.4.0
ignition:
  config:
    merge:
      - local: sno.ign
kernel_arguments:
  should_exist:
    - mitigations=auto
  should_not_exist:
    - mitigations=auto,nosmt
    - mitigations=off
storage:
  files:
    - path: /etc/zincati/config.d/90-disable-feature.toml
      mode: 0644
      contents:
        inline: |
          [updates]
          enabled = false
    - path: /etc/systemd/network/25-nic0.link
      mode: 0644
      contents:
        inline: |
          [Match]
          MACAddress=${mac}
          [Link]
          Name=nic0
    - path: /etc/NetworkManager/system-connections/nic0.nmconnection
      mode: 0600
      overwrite: true
      contents:
        inline: |
          [connection]
          type=ethernet
          interface-name=nic0

          [ethernet]
          mac-address=${mac}

          [ipv4]
          method=manual
          addresses=${ip_addr}/${DOMAIN_NETMASK}
          gateway=${DOMAIN_ROUTER}
          dns=${DOMAIN_ROUTER}
          dns-search=${DOMAIN}
    - path: /etc/hostname
      mode: 0420
      overwrite: true
      contents:
        inline: |
          ${host_name}
EOF

cat ${WORK_DIR}/ipxe-work-dir/${mac//:/-}-config.yml | butane -d ${WORK_DIR}/ipxe-work-dir/ -o ${WORK_DIR}/ipxe-work-dir/ignition/${mac//:/-}.ign
```

Expose internal registry with a Route

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge

oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}'
```

Trust Nexus Cert:

```bash
NEXUS_CERT=$(openssl s_client -showcerts -connect nexus.mac.clg.lab:5001 </dev/null 2>/dev/null|openssl x509 -outform PEM | while read line; do echo "    $line"; done)

cat << EOF | oc apply -n openshift-config -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nexus-ca
data:
  ca-bundle.crt: |
    # Nexus Cert
${NEXUS_CERT}

EOF

oc patch proxy cluster --type=merge --patch '{"spec":{"trustedCA":{"name":"nexus-ca"}}}'
```

```bash
read NEXUS_USER
read -s NEXUS_PASSWD

cat << EOF | oc apply -n okd -f -
apiVersion: v1
kind: Secret
metadata:
    name: nexus-secret
    annotations:
      tekton.dev/docker-0: nexus.mac.clg.lab:5001
type: kubernetes.io/basic-auth
data:
  username: $(echo -n ${NEXUS_USER} | base64)
  password: $(echo -n ${NEXUS_PASSWD} | base64)
EOF

oc patch sa pipeline --type json --patch '[{"op": "add", "path": "/secrets/-", "value": {"name":"nexus-secret"}}]' -n okd
```
