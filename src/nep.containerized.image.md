# 基础镜像制作

帮助客户梳理基础镜像构建，开始构建CI/CD流程.

基于UBI的镜像，有订阅证书问题，现在准备一下。
```bash
# on a vultr host, rockylinux
mkdir -p /data/rhel8/entitle
cd /data/rhel8/entitle

# goto https://access.redhat.com/management/subscriptions
# search employee sku, find a system, go into, and download from subscription
# or goto: https://access.redhat.com/management/systems/4d1e4cc0-2c99-4431-99ce-2f589a24ea11/subscriptions
dnf install -y unzip 
unzip *
unzip consumer_export.zip
find . -name *.pem -exec cp {} ./ \;

mkdir -p /data/dockerfile/
cd /data/dockerfile/

ls /data/rhel8/entitle/*.pem | sed -n '2p' | xargs -I DEMO /bin/cp -f DEMO ./ 

```

## redhat ubi8 
UBI的容器，有注入和使用证书的技巧。

```bash
cat << EOF > /data/dockerfile/nep.redhat.ubi8.dockerfile
FROM registry.access.redhat.com/ubi8

COPY *.pem /etc/pki/entitlement/entitlement.pem
COPY *.pem /etc/pki/entitlement/entitlement-key.pem

RUN dnf -y update || true && \
  sed -i 's|enabled=1|enabled=0|g' /etc/yum/pluginconf.d/subscription-manager.conf && \
  sed -i 's|%(ca_cert_dir)sredhat-uep.pem|/etc/rhsm/ca/redhat-uep.pem|g' /etc/yum.repos.d/redhat.repo && \
  dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm && \
  dnf -y update && \
  dnf -y install net-tools pciutils lksctp-tools iptraf-ng htop vim tcpdump wget bzip2 lrzsz dhcp-server dhcp-client  && \
  dnf -y clean all 

EOF
buildah bud -t quay.io/nep/base-image:ubi8 -f /data/dockerfile/nep.redhat.ubi8.dockerfile .

```

## redhat ubi7

UBI的容器，有注入和使用证书的技巧。

```bash
cat << EOF > /data/dockerfile/nep.redhat.ubi.7.dockerfile
FROM registry.access.redhat.com/ubi7/ubi

COPY *.pem /etc/pki/entitlement/entitlement.pem
COPY *.pem /etc/pki/entitlement/entitlement-key.pem

RUN sed -i 's|%(ca_cert_dir)sredhat-uep.pem|/etc/rhsm/ca/redhat-uep.pem|g' /etc/rhsm/rhsm.conf && \
  yum -y update || true && \
  sed -i 's|enabled=1|enabled=0|g' /etc/yum/pluginconf.d/subscription-manager.conf && \
  sed -i 's|%(ca_cert_dir)sredhat-uep.pem|/etc/rhsm/ca/redhat-uep.pem|g' /etc/yum.repos.d/redhat.repo && \
  yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm && \
  yum -y update && \
  yum -y install net-tools pciutils lksctp-tools iptraf-ng htop vim tcpdump wget bzip2 lrzsz dhcp && \
  yum clean all 

EOF
buildah bud -t quay.io/nep/base-image:ubi7 -f /data/dockerfile/nep.redhat.ubi.7.dockerfile .

```