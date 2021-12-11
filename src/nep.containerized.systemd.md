# 容器中使用systemd加载服务并注入env参数

客户希望在容器中，是用systemd做1号进程，从而让容器方案和裸机方案对齐。

```bash
mkdir -p /data/systemd

cd /data/systemd
cat << 'EOF' > service.sh
#!/bin/bash

# Import our environment variables from systemd
for e in $(tr "\000" "\n" < /proc/1/environ); do
  # if [[ $e == DEMO_ENV* ]]; then
    eval "export $e"
  # fi
done

echo $DEMO_ENV_NIC > /demo.txt
echo $DEMO_ENV_IP >> /demo.txt
echo $DEMO_ENV_MASK >> /demo.txt

EOF
cat << EOF > vbbu.service
[Unit]
Description=vBBU Server
After=network.target

[Service]
Type=forking
User=root
WorkingDirectory=/root/
ExecStart=/service.sh

[Install]
WantedBy=multi-user.target
EOF

cat << EOF > ./vbbu.dockerfile
FROM docker.io/rockylinux/rockylinux:8

USER root
COPY service.sh /service.sh
RUN chmod +x /service.sh
COPY vbbu.service /etc/systemd/system/vbbu.service

RUN systemctl enable vbbu.service

entrypoint ["/usr/sbin/init"]
EOF

buildah bud -t quay.io/wangzheng422/qimgs:systemd -f vbbu.dockerfile .

podman run --rm \
  --env DEMO_ENV_NIC=eth1 \
  --env DEMO_ENV_IP=192.168.100.22 \
  --env DEMO_ENV_MASK=24 \
  quay.io/wangzheng422/qimgs:systemd

podman exec -it `podman ps | grep systemd | awk '{print $1}'` bash

```
