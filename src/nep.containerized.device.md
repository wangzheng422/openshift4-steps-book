# 设备驱动容器化方式加载

从容器向宿主机注入设备驱动，是用init container的方式，把驱动文件注入到容器中，然后容器向宿主机注入。

红帽官方的方案，是device kit/SRO，但是那个太复杂了，我们使用简单粗暴的方式实现。

- https://stackoverflow.com/questions/55291850/kubernetes-how-to-copy-a-cfg-file-into-container-before-contaner-running
- https://access.redhat.com/solutions/4929021

```bash

mkdir -p /data/wzh/fpga
cd /data/wzh/fpga

cat << 'EOF' > ./ocp4.install.sh
#!/bin/bash

if  chroot /host lsmod  | grep nr_drv > /dev/null 2>&1
then
    echo NR Driver Module had loaded!
else
    echo Inserting NR Driver Module
    chroot /host rmmod nr_drv > /dev/null 2>&1

    if [ $(uname -r) == "4.18.0-305.19.1.rt7.91.el8_4.x86_64" ];
    then
        echo insmod nr_drv_wr.ko ...
        /bin/cp -f nr_drv_wr.ko /host/tmp/nr_drv_wr.ko
        chroot /host insmod /tmp/nr_drv_wr.ko load_xeth=1
        /bin/rm -f /host/tmp/nr_drv_wr.ko

        CON_NAME=`chroot /host nmcli -g GENERAL.CONNECTION dev show xeth`

        chroot /host nmcli connection modify "$CON_NAME" con-name xeth
        chroot /host nmcli connection modify xeth ipv4.method disabled ipv6.method disabled
        chroot /host nmcli dev conn xeth
    else
        echo insmod nr_drv_ko Failed!
    fi

fi
EOF

cat << EOF > ./fpga.dockerfile
FROM docker.io/busybox:1.34

USER root
COPY Driver.PKG /Driver.PKG

COPY ocp4.install.sh /ocp4.install.sh
RUN chmod +x /ocp4.install.sh

WORKDIR /
EOF

buildah bud -t registry.ocp4.redhat.ren:5443/nep/fgpa-driver:v04 -f fpga.dockerfile .

buildah push registry.ocp4.redhat.ren:5443/nep/fgpa-driver:v04

cat << EOF > /data/install/fpga.driver.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fpga-driver
  # namespace: default
  labels:
    app: fpga-driver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fpga-driver
  template:
    metadata:
      labels:
        app: fpga-driver
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - fpga-driver
              topologyKey: "kubernetes.io/hostname"
      nodeselector:
        node-role.kubernetes.io/master: ""
      # restartPolicy: Never
      initContainers:
      - name: copy
        image: registry.ocp4.redhat.ren:5443/nep/fgpa-driver:v04
        command: ["/bin/sh", "-c", "tar zvxf /Driver.PKG --strip 1 -C /nep/driver/ && /bin/cp -f /ocp4.install.sh /nep/driver/ "]
        imagePullPolicy: Always
        volumeMounts:
        - name: driver-files
          mountPath: /nep/driver/
      containers:
      - name: driver
        image: registry.redhat.io/rhel8/support-tools:8.4
        # imagePullPolicy: Always
        command: [ "/usr/bin/bash","-c","cd /nep/driver/ && bash ./ocp4.install.sh && sleep infinity " ]
        # command: [ "/usr/bin/bash","-c","tail -f /dev/null || true " ]
        resources:
          requests:
            cpu: 10m
            memory: 20Mi
        securityContext:
          privileged: true
          runAsUser: 0
        volumeMounts:
        - name: driver-files
          mountPath: /nep/driver/
        - name: host
          mountPath: /host
      volumes: 
      - name: driver-files
        emptyDir: {}
      - name: host
        hostPath:
          path: /
          type: Directory
EOF
oc create -f /data/install/fpga.driver.yaml

# to restore
oc delete -f /data/install/fpga.driver.yaml


```
