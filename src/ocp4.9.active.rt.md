# Real-Time Kernel for Openshift4.9

# 硬件部署

假设有专属的FPGA，每个FPGA可以连4个RRU另外GPS连接也是在FPGA上。

# 先使用performance addon operator，这个是官方推荐的方法。

performance addon operator 是openshift4里面的一个operator，他的作用是，让用户进行简单的yaml配置，然后operator帮助客户进行复杂的kernel parameter, kubelet, tuned配置。

```bash

# install performance addon operator following offical document
# https://docs.openshift.com/container-platform/4.9/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html

cat << EOF > /data/install/pao-namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-performance-addon-operator
  annotations:
    workload.openshift.io/allowed: management
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-performance-addon-operator
  namespace: openshift-performance-addon-operator
---
EOF
oc create -f /data/install/pao-namespace.yaml

# then install pao in project openshift-performance-addon-operator

# then create mcp, be careful, the label must be there
cat << EOF > /data/install/worker-rt.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: worker-rt
  labels:
    machineconfiguration.openshift.io/role: worker-rt
spec:
  machineConfigSelector:
    matchExpressions:
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,worker-rt]}
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/worker-rt: ""

EOF
oc create -f /data/install/worker-rt.yaml

# to restore
oc delete -f /data/install/worker-rt.yaml

oc label node worker-0 node-role.kubernetes.io/worker-rt=""

# 以下的配置，是保留了0-1核给系统，剩下的2-19核给应用。
cat << EOF > /data/install/performance.yaml
apiVersion: performance.openshift.io/v2
kind: PerformanceProfile
metadata:
   name: wzh-performanceprofile
spec:
  additionalKernelArgs:
    - no_timer_check
    - clocksource=tsc
    - tsc=perfect
    - selinux=0
    - enforcing=0
    - nmi_watchdog=0
    - softlockup_panic=0
    - isolcpus=2-19
    - nohz_full=2-19
    - idle=poll
    - default_hugepagesz=1G
    - hugepagesz=1G
    - hugepages=16
    - skew_tick=1
    - rcu_nocbs=2-19
    - kthread_cpus=0-1
    - irqaffinity=0-1
    - rcu_nocb_poll
    - iommu=pt
    - intel_iommu=on
    # profile creator
    - audit=0
    - idle=poll
    - intel_idle.max_cstate=0
    - mce=off
    - nmi_watchdog=0
    - nosmt
    - processor.max_cstate=1
  globallyDisableIrqLoadBalancing: true
  cpu:
      isolated: "2-19"
      reserved: "0-1"
  realTimeKernel:
      enabled: true
  numa:  
      topologyPolicy: "single-numa-node"
  nodeSelector:
      node-role.kubernetes.io/worker-rt: ""
  machineConfigPoolSelector:
    machineconfiguration.openshift.io/role: worker-rt
EOF
oc create -f /data/install/performance.yaml

# it will create following
  # runtimeClass: performance-wzh-performanceprofile
  # tuned: >-
  #   openshift-cluster-node-tuning-operator/openshift-node-performance-wzh-performanceprofile

oc get mc/50-nto-worker-rt -o yaml
oc get runtimeClass -o yaml
oc get -n openshift-cluster-node-tuning-operator tuned/openshift-node-performance-wzh-performanceprofile -o yaml

# restore
oc delete -f /data/install/performance.yaml

# enable sctp
# https://docs.openshift.com/container-platform/4.9/networking/using-sctp.html
cat << EOF > /data/install/sctp-module.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-rt-load-sctp-module
  labels:
    machineconfiguration.openshift.io/role: worker-rt
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - path: /etc/modprobe.d/sctp-blacklist.conf
          mode: 0644
          overwrite: true
          contents:
            source: data:,
        - path: /etc/modules-load.d/sctp-load.conf
          mode: 0644
          overwrite: true
          contents:
            source: data:,sctp
EOF
oc create -f /data/install/sctp-module.yaml

# check the result
ssh core@worker-0
uname -a
# Linux worker-0 4.18.0-193.51.1.rt13.101.el8_2.x86_64 #1 SMP PREEMPT RT Thu Apr 8 17:21:44 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux

ps -ef | grep stalld
# root        4416       1  0 14:04 ?        00:00:00 /usr/local/bin/stalld -p 1000000000 -r 10000 -d 3 -t 20 --log_syslog --log_kmsg --foreground --pidfile /run/stalld.pid
# core        6601    6478  0 14:08 pts/0    00:00:00 grep --color=auto stalld

```
# create demo vBBU app
```yaml
---

apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: host-device-du
spec:
  config: '{
    "cniVersion": "0.3.0",
    "type": "host-device",
    "device": "ens18f1",
    "ipam": {
      "type": "host-local",
      "subnet": "192.168.12.0/24",
      "rangeStart": "192.168.12.105",
      "rangeEnd": "192.168.12.106",
      "routes": [{
        "dst": "0.0.0.0/0"
      }],
      "gateway": "192.168.12.1"
    }
  }'

# apiVersion: "k8s.cni.cncf.io/v1"
# kind: NetworkAttachmentDefinition
# metadata:
#   name: host-device-du
# spec:
#   config: '{
#     "cniVersion": "0.3.0",
#     "type": "host-device",
#     "device": "ens18f1",
#     "ipam": {
#       "type": "static",
#       "addresses": [ 
#             {
#               "address": "192.168.12.105/24"
#             },
#             {
#               "address": "192.168.12.106/24"
#             }
#           ],
#       "routes": [ 
#           {
#             "dst": "0.0.0.0/0", 
#             "gw": "192.168.12.1"
#           }
#         ]
#       }
#     }'


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
          { "name": "host-device-du",
            "interface": "veth11" }
          ]'
      cpu-load-balancing.crio.io: "true"
    spec:
      runtimeClassName: performance-wzh-performanceprofile
      containers:
      - name: du-container1
        image: "registry.ocp4.redhat.ren:5443/ocp4/du:v1-wzh-shell-03"
        imagePullPolicy: IfNotPresent
        tty: true
        stdin: true
        env:
          - name: duNetProviderDriver
            value: "host-netdevice"
        #command:
        #  - sleep
        #  - infinity
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
        resources:
          requests:
            cpu: 16
            memory: 48Gi
            hugepages-1Gi: 8Gi
          limits:
            cpu: 16
            memory: 48Gi
            hugepages-1Gi: 8Gi
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
      nodeSelector:
        node-role.kubernetes.io/worker-rt: ""

---

# apiVersion: v1
# kind: Service
# metadata:
#   name: du-http 
# spec:
#   ports:
#   - name: http
#     port: 80
#     targetPort: 80 
#   type: NodePort 
#   selector:
#     app: du-pod1

---

```


# trouble shooting
```bash
# the most important, kernel.sched_rt_runtime_us should be -1, it is setting for realtime, and for stalld
sysctl kernel.sched_rt_runtime_us

sysctl -w kernel.sched_rt_runtime_us=-1

ps -e -o uid,pid,ppid,cls,rtprio,pri,ni,cmd | grep 'stalld\|rcuc\|softirq\|worker\|bin_read\|dumgr\|duoam' 

oc adm must-gather \
--image=registry.redhat.io/openshift4/performance-addon-operator-must-gather-rhel8 

oc get performanceprofile wzh-performanceprofile -o yaml > wzh-performanceprofile.output.yaml

oc describe node/worker-0 > node.worker-0.output

oc describe mcp/worker-rt > mcp.worker-rt.output

# if host device can't release ip address
cd /var/lib/cni/networks/host-device-du
rm -f 192.168.12.105

```

# profile creator

https://docs.openshift.com/container-platform/4.9/scalability_and_performance/cnf-create-performance-profiles.html

```bash
# on helper
oc adm must-gather --image=quay.io/openshift-kni/performance-addon-operator-must-gather:4.9-snapshot --dest-dir=must-gather

podman run --rm --entrypoint performance-profile-creator quay.io/openshift-kni/performance-addon-operator:4.9-snapshot -h

podman run --rm --entrypoint performance-profile-creator -v /data/install/tmp/must-gather:/must-gather:z quay.io/openshift-kni/performance-addon-operator:4.9-snapshot --mcp-name=worker-rt --reserved-cpu-count=2 --topology-manager-policy=single-numa-node --rt-kernel=true --profile-name=wzh-performanceprofile --power-consumption-mode=ultra-low-latency --disable-ht=true  --must-gather-dir-path /must-gather > my-performance-profile.yaml
# ---
# apiVersion: performance.openshift.io/v2
# kind: PerformanceProfile
# metadata:
#   name: wzh-performanceprofile
# spec:
#   additionalKernelArgs:
#   - audit=0
#   - idle=poll
#   - intel_idle.max_cstate=0
#   - mce=off
#   - nmi_watchdog=0
#   - nosmt
#   - processor.max_cstate=1
#   cpu:
#     isolated: 2-19
#     reserved: 0-1
#   machineConfigPoolSelector:
#     machineconfiguration.openshift.io/role: worker-rt
#   nodeSelector:
#     node-role.kubernetes.io/worker-rt: ""
#   numa:
#     topologyPolicy: single-numa-node
#   realTimeKernel:
#     enabled: true
```
