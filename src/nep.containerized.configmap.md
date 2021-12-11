# configmap方式注入license

是用config map的方式，向容器注入license. 同时用host port + router的方式，暴露管理段服务。

```bash
# license file 加载到config map中
oc create configmap license.for.nep  \
    --from-file=license=./3496531EC238AD91DED6DBA5BD6B.lic

# to updated config map
oc create configmap license.for.nep --from-file=license=./3496531EC238AD91DED6DBA5BD6B.lic -o yaml --dry-run=client | oc apply -f -

cat << EOF > /data/install/vbbu.yaml
---

apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: host-device-du
spec:
  config: '{
    "cniVersion": "0.3.0",
    "type": "host-device",
    "device": "xeth",
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.160.0/24",
      "gateway": "192.168.160.254",
      "rangeStart": "192.168.160.1",
      "rangeEnd": "192.168.160.1"
    }
  }'


---

apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: host-device-du-ens
spec:
  config: '{
    "cniVersion": "0.3.0",
    "type": "host-device",
    "device": "enp103s0f0",
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.12.0/24",
      "rangeStart": "192.168.12.105",
      "rangeEnd": "192.168.12.106"
    }
  }'



---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: du-deployment1
  labels:
    app: du-deployment1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: du-pod1
  template:
    metadata:
      labels:
        app: du-pod1
      annotations:
        k8s.v1.cni.cncf.io/networks: '[
          { "name": "host-device-du-ens",
            "interface": "veth11" },
          { "name": "host-device-du",
            "interface": "xeth" }
          ]'
      cpu-load-balancing.crio.io: "true"
    spec:
      runtimeClassName: performance-wzh-performanceprofile
      containers:
      - name: du-container1
        image: "registry.ocp4.redhat.ren:5443/ocp4/du:v1-1623-wzh-01"
        imagePullPolicy: IfNotPresent
        tty: true
        stdin: true
        env:
          - name: duNetProviderDriver
            value: "host-netdevice"
        # command: ["/usr/sbin/init"]
        # - sleep
        # - infinity
        securityContext:
            privileged: true
            capabilities:
                add:
                - CAP_SYS_ADMIN
        volumeMounts:
          - mountPath: /hugepages
            name: hugepage
          - name: lib-modules
            mountPath: /lib/modules
          - name: src
            mountPath: /usr/src
          - name: dev
            mountPath: /dev
          - name: cache-volume
            mountPath: /dev/shm
          - name: license-volume
            mountPath: /nep/lic
        resources:
          requests:
            cpu: 14
            memory: 64Gi
            hugepages-1Gi: 16Gi
          limits:
            cpu: 14
            memory: 64Gi
            hugepages-1Gi: 16Gi
      volumes:
        - name: hugepage
          emptyDir:
            medium: HugePages
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: src
          hostPath:
            path: /usr/src
        - name: dev
          hostPath:
            path: "/dev"
        - name: cache-volume
          emptyDir:
            medium: Memory
            sizeLimit: 16Gi
        - name: license-volume
          configMap:
            name: license.for.nep
            items:
            - key: license
              path: license.lic
      nodeSelector:
        node-role.kubernetes.io/master: ""

---

apiVersion: v1
kind: Service
metadata:
  name: du-http 
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80 
    nodePort: 31071
  type: NodePort 
  selector:
    app: du-pod1

---

apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: du-http 
spec:
  port:
    targetPort: 80
  to:
    kind: Service
    name: du-http 

---

EOF

oc create -f /data/install/vbbu.yaml

# to restore
oc delete -f /data/install/vbbu.yaml

# open browser, to access vbbu console
# http://du-http-default.apps.ocp4s.redhat.ren/

# license file locates in /nep/lic/license.lic

```
