#!/bin/bash

# 自动获取网卡
NIC=$(ip route show default | awk '/default/ {print $5}' | head -n1)
[ -z "$NIC" ] && NIC=eth0

echo "🔍 开始测试 $NIC 上所有全局 IPv6 连通性..."
echo "=================================================="

# 提取所有全局 IPv6 地址（排除 fe80:: 链路本地）
ipv6_list=$(ip -6 addr show "$NIC" | grep -E 'inet6 .* scope global' | awk '{print $2}' | cut -d'/' -f1)

if [ -z "$ipv6_list" ]; then
  echo "❌ 未找到任何全局 IPv6 地址"
  exit 1
fi

total=0
success=0
fail=0

for ip in $ipv6_list; do
  total=$((total + 1))
  echo -n "测试 $ip ... "
  
  # 使用 ping6 测试连通性（发2个包，超时2秒）
  ping6 -c 2 -W 2 -I "$ip" aliyun.com &>/dev/null
  
  if [ $? -eq 0 ]; then
    echo "✅ 通"
    success=$((success + 1))
  else
    echo "❌ 不通"
    fail=$((fail + 1))
  fi
done

echo "=================================================="
echo "📊 测试结果汇总："
echo "总地址数：$total"
echo "✅ 正常：$success"
echo "❌ 失败：$fail"

# 特别提示 DAD 失败的地址
dadfailed=$(ip -6 addr show "$NIC" | grep -o 'inet6 [0-9a-f:]*dadfailed' | awk '{print $2}')
if [ -n "$dadfailed" ]; then
  echo -e "\n⚠️  以下地址因 DAD 失败，未参与测试（已被系统禁用）："
  echo "$dadfailed"
fi





# 查看全部 IP
ip addr

# 只看 IPv4
ip -4 addr

# 只看 IPv6
ip -6 addr
