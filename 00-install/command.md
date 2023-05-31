### Install Package & files for Openshift ###
# Packages
  yum install wget jq nfs-utils
  dnf install haproxy bind bind-utils httpd podman net-tools bash-completion chrony

# OC, Openshift-installer tool
  curl -s  https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.12.3/openshift-client-linux-4.12.3.tar.gz | tar zxvf - oc
  curl -s https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.12.3/openshift-install-linux-4.12.3.tar.gz | tar zxvf - openshift-install
  mv oc openshift-install /usr/local/bin
  oc completion bash > /etc/bash_completion.d/oc_bash_completion

# RHCOS
  https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.12/latest/rhcos-live.x86_64.iso
  https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.12/latest/rhcos-metal.x86_64.raw.gz

### Private Registry(Optional) ###
# Create SSL
  mkdir -p /opt/registry/{auth,certs,data} 
  cd /opt/registry/certs
  cert_c="KR"
  cert_s="" # Certificate State (S)
  cert_l="" # Certificate Locality (L)
  cert_o="timegate" # Certificate Organization (O)
  cert_ou="coretech" # Certificate Organizational Unit (OU)
  cert_cn="registry.ocp4.example.io" # Certificate Common Name (CN)
  
  openssl req \
  -newkey rsa:4096 \
  -nodes \
  -sha256 \
  -keyout /opt/registry/certs/registry.ocp4.example.io.key \
  -x509 \
  -days 365 \
  -out /opt/registry/certs/registry.ocp4.example.io.crt \
  -addext "subjectAltName=DNS:*.ocp4.example.io" \
  -subj "/C=${cert_c}/ST=${cert_s}/L=${cert_l}/O=${cert_o}/OU=${cert_ou}/CN=${cert_cn}"
# Update CA Trust
  cp /opt/registry/certs/registry.ocp4.example.io.crt /etc/pki/ca-trust/source/anchors/
  update-ca-trust extract

# Create Registry
  htpasswd -bBc /opt/registry/auth/htpasswd openshift redhat
  podman create \
  --name mirror-registry \
  -p 5000:5000 \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry" \
  -e "REGISTRY_HTTP_SECRET=ALongRandomSecretForRegistry" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  -e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.ocp4.example.io.crt" \
  -e "REGISTRY_HTTP_TLS_KEY=/certs/registry.ocp4.example.io.key" \
  -e "REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true" \
  -v /opt/registry/auth:/auth \
  -v /opt/registry/certs:/certs \
  -v /opt/registry/data:/var/lib/registry \
  docker.io/library/registry:2

# Mirror files from redhat Registry
  VERSION=4.12
  LOCAL_REG='registry.ocp4.example.io:5000'
  LOCAL_REPO='ocp4/openshift4'
  UPSTREAM_REPO='quay.io/openshift-release-dev/ocp-release:4.12.3-x86_64'

  oc adm release mirror \
  -a /opt/registry/certs/pull-secret-update.txt \
  --from=$UPSTREAM_REPO \
  --to-release-image=$LOCAL_REG/$LOCAL_REPO:${VERSION} \
  --to=$LOCAL_REG/$LOCAL_REPO


### Pull-Secret ###
# Pull-Secret Download URL
  https://console.redhat.com/openshift/install/metal/user-provisioned

# Add Private Registry Auth to Pull-Secret(Optional)
  registry_fqdn="registry.ocp4.example.io"
  auth=$(echo -n 'openshift:redhat' | openssl base64)
  AUTHSTRING="{\"$registry_fqdn:5000\": {\"auth\": \"$auth\",\"email\":\"cw.jeong@time-gate.com\"}}"
  jq ".auths += $AUTHSTRING" pull-secret.json > pull-secret-update.txt


### SSH Key ###
# Create SSH KEY for coreos
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa

### Add Private Registry Certificate & Mirror Source to Install-Config ###
  echo "additionalTrustBundle: |" >> install-config.yaml
  sed -e 's/^/ /' /opt/registry/certs/registry.ocp4.example.io.crt >> install-config.yaml
  echo "imageContentSources:" >> install-config.yaml
  echo "- mirrors:" >> install-config.yaml
  echo "  - registry.ocp4.example.io:5000/ocp4/openshift4" >> install-config.yaml
  echo "  source: quay.io/openshift-release-dev/ocp-release" >> install-config.yaml
  echo "- mirrors:" >> install-config.yaml
  echo "  - registry.ocp4.example.io:5000/ocp4/openshift4" >> install-config.yaml
  echo "  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev" >> install-config.yaml


### IF X509 Certificate Issue alert ###
oc rsh -n openshift-authentication <authentication_pod_name> cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > ~/ingress-ca.crt
cp ingress-ca.crt /etc/pki/ca-trust/source/anchors/
