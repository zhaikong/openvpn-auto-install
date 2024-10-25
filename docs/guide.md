
# 创建 docs 目录
mkdir -p docs

# 创建部署指南
cat > docs/guide.md << 'EOF'
# OpenVPN 服务器部署指南

#!/bin/bash

# OpenVPN 一键安装脚本
# 作者：Assistant
# 更新日期：2024-03-25

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

# 检查是否为root用户
if [[ $EUID -ne 0 ]]; then
   echo -e "${RED}错误：此脚本必须以root权限运行${NC}" 
   exit 1
fi

# 检测系统类型
if [ -f /etc/debian_version ]; then
    DISTRO="debian"
    # 安装基础包
    apt update
    apt install -y openvpn easy-rsa wget curl
elif [ -f /etc/redhat-release ]; then
    DISTRO="centos"
    # 安装基础包
    yum update -y
    yum install -y epel-release
    yum install -y openvpn easy-rsa wget curl
else
    echo -e "${RED}不支持的操作系统${NC}"
    exit 1
fi

# 创建工作目录
mkdir -p /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa

# 初始化PKI
echo -e "${GREEN}初始化 PKI...${NC}"
cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-dh
./easyrsa build-server-full server nopass
./easyrsa build-client-full client1 nopass

# 配置服务器
echo -e "${GREEN}配置 OpenVPN 服务器...${NC}"
cat > /etc/openvpn/server.conf << EOF
port 1194
proto udp
dev tun
ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key /etc/openvpn/easy-rsa/pki/private/server.key
dh /etc/openvpn/easy-rsa/pki/dh.pem
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
keepalive 10 120
cipher AES-256-CBC
user nobody
group nobody
persist-key
persist-tun
status openvpn-status.log
verb 3
EOF

# 启用IP转发
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
sysctl -p

# 配置防火墙
if [ "$DISTRO" == "centos" ]; then
    firewall-cmd --permanent --add-service=openvpn
    firewall-cmd --permanent --add-masquerade
    firewall-cmd --reload
else
    # 对于Debian/Ubuntu使用iptables
    iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
    # 保存iptables规则
    if [ -d /etc/iptables ]; then
        iptables-save > /etc/iptables/rules.v4
    fi
fi

# 生成客户端配置
echo -e "${GREEN}生成客户端配置文件...${NC}"
SERVER_IP=$(curl -s ifconfig.me)

cat > /root/client.ovpn << EOF
client
dev tun
proto udp
remote $SERVER_IP 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
verb 3
EOF

# 添加证书到客户端配置
echo "<ca>" >> /root/client.ovpn
cat /etc/openvpn/easy-rsa/pki/ca.crt >> /root/client.ovpn
echo "</ca>" >> /root/client.ovpn
echo "<cert>" >> /root/client.ovpn
cat /etc/openvpn/easy-rsa/pki/issued/client1.crt >> /root/client.ovpn
echo "</cert>" >> /root/client.ovpn
echo "<key>" >> /root/client.ovpn
cat /etc/openvpn/easy-rsa/pki/private/client1.key >> /root/client.ovpn
echo "</key>" >> /root/client.ovpn

# 启动OpenVPN服务
systemctl start openvpn@server
systemctl enable openvpn@server

echo -e "${GREEN}OpenVPN 安装完成！${NC}"
echo -e "${GREEN}客户端配置文件位置：/root/client.ovpn${NC}"
echo -e "${GREEN}请使用安全的方式将client.ovpn文件传输到客户端设备${NC}"

EOF
