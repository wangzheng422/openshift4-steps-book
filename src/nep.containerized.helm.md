# helm chart / helm operator 制作

## build helm operator

```bash
mkdir -p /data/down
cd /data/down
wget https://mirror.openshift.com/pub/openshift-v4/clients/operator-sdk/latest/operator-sdk-linux-x86_64.tar.gz
tar zvxf operator-sdk-linux-x86_64.tar.gz
install operator-sdk /usr/local/bin/

operator-sdk init --plugins helm --help

mkdir -p /data/helm
cd /data/helm

# 初始化项目
operator-sdk init \
    --plugins=helm \
    --project-name nep-helm-operator \
    --domain=nep.com \
    --group=apps \
    --version=v1alpha1 \
    --kind=VBBU 

make bundle
# operator-sdk generate kustomize manifests -q

# Display name for the operator (required):
# > nep vBBU

# Description for the operator (required):
# > nep vRAN application including fpga driver, vCU, vDU

# Provider's name for the operator (required):
# > nep

# Any relevant URL for the provider name (optional):
# > na.nep.com

# Comma-separated list of keywords for your operator (required):
# > nep,vbbu,vran,vcu,vdu

# Comma-separated list of maintainers and their emails (e.g. 'name1:email1, name2:email2') (required):
# >
# No list provided.
# Comma-separated list of maintainers and their emails (e.g. 'name1:email1, name2:email2') (required):
# > wangzheng:wangzheng422@foxmail.com
# cd config/manager && /data/helm/bin/kustomize edit set image controller=quay.io/nep/nep-helm-operator:latest
# /data/helm/bin/kustomize build config/manifests | operator-sdk generate bundle -q --overwrite --version 0.0.1
# INFO[0001] Creating bundle.Dockerfile
# INFO[0001] Creating bundle/metadata/annotations.yaml
# INFO[0001] Bundle metadata generated suceessfully
# operator-sdk bundle validate ./bundle
# INFO[0000] All validation tests have completed successfully

cd /data/helm/helm-charts/vbbu
helm lint

dnf install -y podman-docker

cd /data/helm/
make docker-build
# docker build -t quay.io/nep/nep-helm-operator:v01 .
# Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
# STEP 1/5: FROM registry.redhat.io/openshift4/ose-helm-operator:v4.9
# STEP 2/5: ENV HOME=/opt/helm
# --> 1eec2f9c094
# STEP 3/5: COPY watches.yaml ${HOME}/watches.yaml
# --> 1836589a08c
# STEP 4/5: COPY helm-charts  ${HOME}/helm-charts
# --> b6cd9f24e47
# STEP 5/5: WORKDIR ${HOME}
# COMMIT quay.io/nep/nep-helm-operator:v01
# --> 1f9bcc4cecc
# Successfully tagged quay.io/nep/nep-helm-operator:v01
# 1f9bcc4cecc55e68170e2a6f45dad7b318018df8bf3989bd990f567e3ccdfcd9

make docker-push
# docker push quay.io/nep/nep-helm-operator:v01
# Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
# Getting image source signatures
# Copying blob 8cd9b2cfbe06 skipped: already exists
# Copying blob 5bc03dec6239 skipped: already exists
# Copying blob 525ed45dbdb1 skipped: already exists
# Copying blob 758ace4ace74 skipped: already exists
# Copying blob deb6b0f93acd skipped: already exists
# Copying blob ac83cd3b61fd skipped: already exists
# Copying blob 12f964d7475b [--------------------------------------] 0.0b / 0.0b
# Copying config 1f9bcc4cec [--------------------------------------] 0.0b / 4.0KiB
# Writing manifest to image destination
# Copying config 1f9bcc4cec [--------------------------------------] 0.0b / 4.0KiB
# Writing manifest to image destination
# Storing signatures

make bundle-build BUNDLE_IMG=quay.io/nep/nep-helm-operator:bundle-v01
# docker build -f bundle.Dockerfile -t quay.io/nep/nep-helm-operator:bundle-v01 .
# Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
# STEP 1/14: FROM scratch
# STEP 2/14: LABEL operators.operatorframework.io.bundle.mediatype.v1=registry+v1
# --> Using cache b67edfbd23d6ba9c3f484a1e01f9da79fbffdc44e913423e2f616e477df372e1
# --> b67edfbd23d
# STEP 3/14: LABEL operators.operatorframework.io.bundle.manifests.v1=manifests/
# --> Using cache f2eef5180d3c9c63f40a98880ec95088b8395845e0f90960a194326d77a6f3b4
# --> f2eef5180d3
# STEP 4/14: LABEL operators.operatorframework.io.bundle.metadata.v1=metadata/
# --> Using cache 6fc10718a71e30d31cc652b47ac27ca87901ff4fda17a25e2d6bc53344e50673
# --> 6fc10718a71
# STEP 5/14: LABEL operators.operatorframework.io.bundle.package.v1=nep-helm-operator
# --> Using cache 6664d1d6c64c0954c18a432194845551e5a0c6f9bba33175d77c8791e2b0f6e0
# --> 6664d1d6c64
# STEP 6/14: LABEL operators.operatorframework.io.bundle.channels.v1=alpha
# --> Using cache 32878b9e903851bb51b6c0635c77112b4244f4ce7e9d8a7b0a0d8cf7fe7bbe0e
# --> 32878b9e903
# STEP 7/14: LABEL operators.operatorframework.io.metrics.builder=operator-sdk-v1.10.1-ocp
# --> Using cache c5482c80a3287494a5f35ee8df782f4499ad6def2aaa55652e5fc57d4dfa8f0d
# --> c5482c80a32
# STEP 8/14: LABEL operators.operatorframework.io.metrics.mediatype.v1=metrics+v1
# --> Using cache 68822f2fae03c5efc8b980882f66e870d8942d80dbf697e3d784c46f95c50437
# --> 68822f2fae0
# STEP 9/14: LABEL operators.operatorframework.io.metrics.project_layout=helm.sdk.operatorframework.io/v1
# --> Using cache a85519d2774008b3071baf6098ec59561102ef1f337acd19b2c7ef739ebae89e
# --> a85519d2774
# STEP 10/14: LABEL operators.operatorframework.io.test.mediatype.v1=scorecard+v1
# --> Using cache 17a1b08e1dca2295f98e3288d592a08636d15d7461e25e11744a499160a1546c
# --> 17a1b08e1dc
# STEP 11/14: LABEL operators.operatorframework.io.test.config.v1=tests/scorecard/
# --> Using cache 9b6a20b0ff75b501a321fe4fbdfd1d284763e65596dc85675f119e5e3de69657
# --> 9b6a20b0ff7
# STEP 12/14: COPY bundle/manifests /manifests/
# --> Using cache ff3aa5b299dae11f464d8ad56f4ae5130974e1cebd0cf273bc03aba11fcb7377
# --> ff3aa5b299d
# STEP 13/14: COPY bundle/metadata /metadata/
# --> Using cache 19395ef3259bbb4e1f5da9616195139698a3ef18e7f904a2a1cd7515cd9829f3
# --> 19395ef3259
# STEP 14/14: COPY bundle/tests/scorecard /tests/scorecard/
# --> Using cache 2268eb0a731f424f70e5b46222a1accd5344560ac9ab609ca3ccb5a4d0cd6669
# COMMIT quay.io/nep/nep-helm-operator:bundle-v01
# --> 2268eb0a731
# Successfully tagged quay.io/nep/nep-helm-operator:bundle-v01
# Successfully tagged quay.io/nep/nep-helm-operator-bundle:v0.0.1
# 2268eb0a731f424f70e5b46222a1accd5344560ac9ab609ca3ccb5a4d0cd6669


make bundle-push BUNDLE_IMG=quay.io/nep/nep-helm-operator:bundle-v01
# make docker-push IMG=quay.io/nep/nep-helm-operator:bundle-v01
# make[1]: Entering directory '/data/helm'
# docker push quay.io/nep/nep-helm-operator:bundle-v01
# Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
# Getting image source signatures
# Copying blob 24b54377030e skipped: already exists
# Copying blob 1929cd83db02 skipped: already exists
# Copying blob 44ef63131a17 [--------------------------------------] 0.0b / 0.0b
# Copying config 2268eb0a73 done
# Writing manifest to image destination
# Copying config 2268eb0a73 [--------------------------------------] 0.0b / 3.3KiB
# Writing manifest to image destination
# Storing signatures
# make[1]: Leaving directory '/data/helm'

make catalog-build CATALOG_IMG=quay.io/nep/nep-helm-operator:catalog-v01  BUNDLE_IMG=quay.io/nep/nep-helm-operator:bundle-v01 
# ./bin/opm index add --mode semver --tag quay.io/nep/nep-helm-operator:catalog-v01 --bundles quay.io/nep/nep-helm-operator:bundle-v01
# INFO[0000] building the index                            bundles="[quay.io/nep/nep-helm-operator:bundle-v01]"
# INFO[0000] resolved name: quay.io/nep/nep-helm-operator:bundle-v01
# INFO[0000] fetched                                       digest="sha256:1365e5913f05b733124a2a88c3113899db0c42f62b5758477577ef2117aff09f"
# INFO[0000] fetched                                       digest="sha256:be008c9c2b4f2c031b301174608accb8622c8d843aba2d1af4d053d8b00373c2"
# INFO[0000] fetched                                       digest="sha256:2268eb0a731f424f70e5b46222a1accd5344560ac9ab609ca3ccb5a4d0cd6669"
# INFO[0000] fetched                                       digest="sha256:d8e28b323fec2e4de5aecfb46c4ce3e315e20f49b78f43eb7a1d657798695655"
# INFO[0000] fetched                                       digest="sha256:c19ac761be31fa163ea3da95cb63fc0c2aaca3b316bfb049f6ee36f77522d323"
# INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:d8e28b323fec2e4de5aecfb46c4ce3e315e20f49b78f43eb7a1d657798695655 2985 [] map[] <nil>}
# INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:c19ac761be31fa163ea3da95cb63fc0c2aaca3b316bfb049f6ee36f77522d323 398 [] map[] <nil>}
# INFO[0001] unpacking layer: {application/vnd.docker.image.rootfs.diff.tar.gzip sha256:be008c9c2b4f2c031b301174608accb8622c8d843aba2d1af4d053d8b00373c2 438 [] map[] <nil>}
# INFO[0001] Could not find optional dependencies file     dir=bundle_tmp582129875 file=bundle_tmp582129875/metadata load=annotations
# INFO[0001] found csv, loading bundle                     dir=bundle_tmp582129875 file=bundle_tmp582129875/manifests load=bundle
# INFO[0001] loading bundle file                           dir=bundle_tmp582129875/manifests file=apps.nep.com_vbbus.yaml load=bundle
# INFO[0001] loading bundle file                           dir=bundle_tmp582129875/manifests file=nep-helm-operator-controller-manager-metrics-service_v1_service.yaml load=bundle
# INFO[0001] loading bundle file                           dir=bundle_tmp582129875/manifests file=nep-helm-operator-manager-config_v1_configmap.yaml load=bundle
# INFO[0001] loading bundle file                           dir=bundle_tmp582129875/manifests file=nep-helm-operator-metrics-reader_rbac.authorization.k8s.io_v1_clusterrole.yaml load=bundle
# INFO[0001] loading bundle file                           dir=bundle_tmp582129875/manifests file=nep-helm-operator.clusterserviceversion.yaml load=bundle
# INFO[0001] Generating dockerfile                         bundles="[quay.io/nep/nep-helm-operator:bundle-v01]"
# INFO[0001] writing dockerfile: index.Dockerfile322782265  bundles="[quay.io/nep/nep-helm-operator:bundle-v01]"
# INFO[0001] running podman build                          bundles="[quay.io/nep/nep-helm-operator:bundle-v01]"
# INFO[0001] [podman build --format docker -f index.Dockerfile322782265 -t quay.io/nep/nep-helm-operator:catalog-v01 .]  bundles="[quay.io/nep/nep-helm-operator:bundle-v01]"

make catalog-push CATALOG_IMG=quay.io/nep/nep-helm-operator:catalog-v01
# make docker-push IMG=quay.io/nep/nep-helm-operator:catalog-v01
# make[1]: Entering directory '/data/helm'
# docker push quay.io/nep/nep-helm-operator:catalog-v01
# Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
# Getting image source signatures
# Copying blob 8a20ae5d4166 done
# Copying blob a98a386b6ec2 skipped: already exists
# Copying blob 4e7f383eb531 skipped: already exists
# Copying blob bc276c40b172 skipped: already exists
# Copying blob b15904f6a114 skipped: already exists
# Copying blob 86aadf4df7dc skipped: already exists
# Copying config 5d5d1c219c done
# Writing manifest to image destination
# Storing signatures
# make[1]: Leaving directory '/data/helm'

export OPERATOR_VERION=v04

make docker-build IMG=quay.io/nep/nep-helm-operator:$OPERATOR_VERION

make docker-push IMG=quay.io/nep/nep-helm-operator:$OPERATOR_VERION

make bundle IMG=quay.io/nep/nep-helm-operator:$OPERATOR_VERION

make bundle-build bundle-push catalog-build catalog-push \
    BUNDLE_IMG=quay.io/nep/nep-helm-operator:bundle-$OPERATOR_VERION \
    CATALOG_IMG=quay.io/nep/nep-helm-operator:catalog-$OPERATOR_VERION

# on openshift helper node
cat << EOF > /data/install/nep.catalog.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: nep
  namespace: openshift-marketplace
spec:
  displayName: nep
  publisher: nep
  sourceType: grpc
  image: ghcr.io/wangzheng422/nep-helm-operator:catalog-2021-12-03-0504
  updateStrategy:
    registryPoll:
      interval: 10m
EOF
oc create -f /data/install/nep.catalog.yaml
# to restore
oc delete -f /data/install/nep.catalog.yaml

```

## helm repository
https://medium.com/@mattiaperi/create-a-public-helm-chart-repository-with-github-pages-49b180dbb417

```bash
# try to build the repo, and add it into github action
# mkdir -p /data/helm/helm-repo
cd /data/helm/helm-repo

helm package ../helm-charts/*

helm repo index --url https://wangzheng422.github.io/nep-helm-operator/ .

# try to use the repo
helm repo add myhelmrepo https://wangzheng422.github.io/nep-helm-operator/

helm repo list
# NAME            URL
# myhelmrepo      https://wangzheng422.github.io/nep-helm-operator/

helm search repo vbbu
# NAME            CHART VERSION   APP VERSION     DESCRIPTION
# myhelmrepo/vbbu 0.1.0           1.16.0          A Helm chart for Kubernetes

# for ocp, if you are disconnected
cat << EOF > /data/install/helm.nep.yaml
apiVersion: helm.openshift.io/v1beta1
kind: HelmChartRepository
metadata:
  name: nep-helm-charts-wzh
spec:
 # optional name that might be used by console
  name: nep-helm-charts-wzh
  connectionConfig:
    url: http://nexus.ocp4.redhat.ren:8082/repository/wangzheng422.github.io/
EOF
oc create -f /data/install/helm.nep.yaml

```