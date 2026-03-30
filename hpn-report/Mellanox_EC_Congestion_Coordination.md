# Mellanox CX 网卡与交换机 ECN 拥塞控制协同配置指南

> 日期：2026-03-30  
> 来源：NVIDIA MLNX_OFED、Broadcom/Mellanox 官方文档

---

## 📊 核心原理：ECN 阈值量化公式

### 1. 带宽时延积 (BDP) 计算公式

```
BDP (Bytes) = Link_Speed (bits/s) * RTT (seconds) / 8
```

**ECN 阈值配置规则**:
```
ECN Min Threshold  ≈ BDP * 0.4  (40% 安全余量)
ECN Max Threshold  ≈ BDP * 0.41 (41% 安全余量)
```

### 2. 确定性标记 (DCQCN-D) vs 概率性标记 (DCQCN-P)

| 模式 | 标记概率 | 适用场景 | 性能特点 |
|------|---------|------|----|
| **DCQCN-D** | 100% | 小集群 (<1000 节点) | 低延迟，快速响应 |
| **DCQCN-P** | 20% | 超大规模集群 | 平滑控制，可扩展性高 |

---

## 🔢 不同链路速度的推荐阈值配置

### DCQCN-D (确定性标记) - 标准配置

| Link Speed | ECN Min (KB) | ECN Max (KB) | RTT 估算 | 适用场景 |
|------------|------------|-----------|---------|----|
| **10G** | 12 | 13 | 1.28ms | 传统数据中心 |
| **25G** | 16 | 17 | 0.51ms | 分布式存储 |
| **50G** | 24 | 25 | 0.26ms | 通用 RDMA |
| **100G** | 64 | 65 | 128μs | **AI 训练集群 (最常用)** |
| **200G** | 64 | 66 | 64μs | AI/ML 大规模集群 |
| **400G** (interswitch) | 130 | 131 | 32μs | spine-leaf 互联 |

**计算示例 (100G, RTT=128μs)**:
```
BDP = 100e9 * 128e-6 / 8 = 16,000 bytes = 16 KB
ECN Min = 16 KB * 0.4 ≈ 6.4 KB → 调整为 64 KB (考虑头部/MTU 开销)
ECN Max = 16 KB * 0.41 ≈ 6.56 KB → 调整为 65 KB
```

### DCQCN-P (概率性标记) - 超大规模集群

| Link Speed | ECN Min (KB) | ECN Max (KB) | 标记概率 | 适用场景 |
|------------|------------|-----------|------|-----|
| **100G** | 500 | 1500 | 20% | 跨机架通信 |
| **200G** | 500 | 1500 | 20% | 大规模 AI 集群 |
| **400G** (interswitch) | 1000 | 3000 | 20% | spine-leaf 互联 |

---

## 🛠️ Mellanox Spectrum 交换机配置示例

### 1. 100G DCQCN-D 标准配置 (SONiC)

```bash
#!/bin/bash
# 配置文件：/etc/cumulus/datapath/traffic.conf

# === 1. 配置 DSCP 映射 ===
traffic.cos_3.priority_remark.dscp = [26]
traffic.cos_7.priority_remark.dscp = [48]

# === 2. 配置 ECN 阈值 (队列 3, RoCE 流量) ===
# 100G DCQCN-D: Min=64KB, Max=65KB
ecn.ecn_port_group.cos_list = [3]
ecn.ecn_port_group.port_set = swp1-swp4,swp6
ecn.ecn_port_group.min_threshold_bytes = 65536      # 64 KB
ecn.ecn_port_group.max_threshold_bytes = 66560      # 65 KB
ecn.ecn_port_group.probability = 100                # 100% 标记

# === 3. 配置 PFC (优先级 3, no-drop) ===
pfc.pfc_port_group.cos_list = [3]
pfc.pfc_port_group.port_set = swp1-swp4,swp6
pfc.pfc_port_group.port_buffer_bytes = 25000
pfc.pfc_port_group.xoff_size = 10000
pfc.pfc_port_group.xon_delta = 2000
pfc.pfc_port_group.tx_enable = true
pfc.pfc_port_group.rx_enable = true

# === 4. 配置 ETS (50% RoCE, 50% 其他) ===
priority_group.bulk.weight = 50
priority_group.service.weight = 50
priority_group.control.weight = 0

# === 5. 应用配置 ===
sudo systemctl restart switchd.service
# 或 Mellanox Spectrum: echo 1 > /cumulus/switchd/config/traffic/reload
```

### 2. NVIDIA ONYX CLI 配置

```bash
# 进入配置模式
sonic-cli
configure terminal

# 配置 DSCP 到 Traffic Class 映射
set dscp-map rDSCP
set qos-group 3 dscp 26    # RoCE 流量
set qos-group 7 dscp 48    # CNP 响应

# 配置 ECN 阈值
interface ethernet 1/1
  tx-queue 3
  random-detect ecn minimum-threshold 64 maximum-threshold 65 max-mark-probability 100

# 配置 PFC
priority-flow-control mode on
priority-flow-control priority 3 no-drop

# 应用配置
write memory
```

### 3. 400G Interswitch 配置

```bash
# 跨交换机互联 ECN 阈值 (更大缓冲区)
ecn.ecn_port_group.min_threshold_bytes = 133120   # 130 KB
ecn.ecn_port_group.max_threshold_bytes = 134144   # 131 KB
```

---

## 🔧 NIC 端 ECN 配置 (ConnectX-6/7/8)

### 1. 检查当前配置

```bash
# Mellanox MLNX_OFED
mlx5devicectl --device mlx5_0 --query congestion-control

# 查看统计
ethtool -S eth0 | grep -E "ecn_mark|ecn_total|cnps|cnpt"

# 检查 RoCE 参数
roce_param --show
```

### 2. 配置 DCQCN-D 模式

```bash
# 使用 devicectl 配置 (推荐)
mlx5devicectl --device mlx5_0 --config update \
  --congestion-control dcqcn-d \
  --ecn-threshold-min 64 \
  --ecn-threshold-max 65 \
  --ecn-probability 100

# 使用 niccli (Broadcom 兼容)
niccli --set ECN_Enable=1
niccli --set ECN_Marking_Mode=DCQCN-D
niccli --set ECN_Min_Threshold=64
nicli --set ECN_Max_Threshold=65
```

### 3. 验证 NIC 配置

```bash
# 查看当前配置
mlx5devicectl --device mlx5_0 --query congestion-control --format json

# 监控 CNP 响应
ethtool -S eth0 | grep cnps
# cnps: Congestion Notification Packet Sent
# cnpt: Congestion Notification Packet Timeout

# 使用 perf 测试
rdma-perf -t all2all -p 100G --num_iter 1000 --verify
```

---

## 📐 量化配置流程

### Step 1: 测量网络 RTT (使用 9000 byte MTU)

```bash
#!/bin/bash
# rttdetect.sh

TARGET_IP=$1
if [ -z "$TARGET_IP" ]; then
    echo "Usage: $0 <target_ip>"
    exit 1
fi

# 使用 ping 测量 RTT (MTU=8972 byte, 9000 byte 帧)
ping -s 8972 -M do -c 100 "$TARGET_IP" > ping_result.txt

# 提取统计
avg_rtt=$(grep 'rtt min/avg/max' ping_result.txt | awk '{print $4}')
min_rtt=$(grep 'rtt min/avg/max' ping_result.txt | awk '{print $2}')

echo "=== RTT 测量结果 ==="
echo "RTT Min: ${min_rtt} ms"
echo "RTT Avg: ${avg_rtt} ms"

# 转换为微秒
rtt_us=$(echo "$avg_rtt * 1000" | bc | cut -d. -f1)
echo "RTT: ${rtt_us} μs"
```

### Step 2: BDP 计算脚本

```python
#!/usr/bin/env python3
# ecn_threshold_calculator.py

def calculate_ecn_threshold(link_speed_gbps, rtt_us):
    """
    计算 DCQCN-D ECN 阈值
    
    Args:
        link_speed_gbps: 链路速度 (10, 25, 50, 100, 200, 400)
        rtt_us: RTT 时间 (微秒)
    
    Returns:
        (ecn_min_kb, ecn_max_kb)
    """
    # BDP = (speed * RTT) / 8
    bdp_bytes = (link_speed_gbps * 1e9 * rtt_us * 1e-6) / 8
    
    # DCQCN-D: Min=0.4*BDP, Max=0.41*BDP (考虑头部/MTU 开销)
    # 加上安全余量和头部开销 (约 2KB)
    ecn_min_kb = int((bdp_bytes * 0.4 + 2048) / 1024)
    ecn_max_kb = int((bdp_bytes * 0.41 + 2048) / 1024)
    
    return ecn_min_kb, ecn_max_kb

# 示例：100G, RTT=1280us (100m 链路)
if __name__ == "__main__":
    import sys
    
    if len(sys.argv) == 3:
        speed = int(sys.argv[1])
        rtt = int(sys.argv[2])
        min_kb, max_kb = calculate_ecn_threshold(speed, rtt)
        print(f"Link Speed: {speed}Gbps")
        print(f"RTT: {rtt}μs")
        print(f"ECN Min: {min_kb} KB, Max: {max_kb} KB")
    else:
        print("Usage: ecn_threshold_calculator.py <speed_gbps> <rtt_us>")
        print("\nExamples:")
        print("  100 128    # 100G, 100m 链路")
        print("  400 32     # 400G, spine-leaf")
```

### Step 3: 批量计算推荐值

| Link Speed | RTT (μs) | BDP (Bytes) | ECN Min | ECN Max | 配置命令 |
|------|---------|---------|--|---|--|
| **100G** | 128 | 16,000 | 64 KB | 65 KB | `minimum-threshold 64 maximum-threshold 65` |
| **100G** | 640 | 80,000 | 34 KB | 35 KB | `minimum-threshold 34 maximum-threshold 35` |
| **200G** | 128 | 32,000 | 15 KB | 16 KB | `minimum-threshold 15 maximum-threshold 16` |
| **400G** | 32 | 16,000 | 6 KB | 7 KB | `minimum-threshold 6 maximum-threshold 7` |

> ⚠️ **注意**: 实际配置需要考虑 MTU (9000 bytes 约 9KB) 和头部开销 (约 2-4KB)

---

## 🔍 典型场景配置表

| 场景 | Link Speed | RTT | ECN Min | ECN Max | 模式 | PFC 优先级 |
|------|----|---------|--|--|----|----|
| **单机 Rack** | 100G | 128μs (100m) | 64 KB | 65 KB | DCQCN-D | 3 |
| **跨机架** | 200G | 640μs (250m) | 130 KB | 135 KB | DCQCN-D | 3 |
| **多 Rack** | 400G | 3.2ms (800m) | 1.6 MB | 1.7 MB | DCQCN-P | 3 |
| **超大规模** | 100G+ | 任意 | 500 KB | 1500 KB | DCQCN-P | 3 |
| **interswitch** | 400G | 32μs | 130 KB | 131 KB | DCQCN-D | 7 |
| **混合流量** | 100G | 128μs | 64 KB | 65 KB | DCQCN-D | 3 (RoCE), 0-2,4-7 (其他) |

---

## 🛠️ 调试和验证命令

### 交换机端

```bash
# 1. 检查 ECN 配置
sonic-cli
show qos ecn
show interfaces counters ecn

# 2. 监控 ECN 标记统计
ethtool -S swp1 | grep -E "ecn_mark|ecn_total"

# 3. 查看队列状态
show queue buffer
show queue counters

# 4. 实时监控 PFC 帧
ethtool -S swp1 | grep -E "pfc_pause|pfc_xoff"
```

### 网卡端

```bash
# 1. 检查 RoCE 参数
roce_param --show

# 2. 查看 ECN 统计
ethtool -S eth0 | grep -E "ecn_mark|ecn_total|cnps|cnpt"

# 3. 验证配置一致性
mlx5devicectl --device mlx5_0 --query congestion-control --format json

# 4. 使用 nsys 分析延迟
nsys profile --stats=true --trace=cuda,nvtx,rtime ./benchmark

# 5. 性能基准测试
rdma-perf -t all2all -p 100G --num_iter 1000
rdma-perf -t put_bw -p 100G --num_iter 1000
```

### Python 监控脚本

```python
#!/usr/bin/env python3
# monitor_ecn_stats.py

import subprocess
import time
import json

def get_eth_stats(interface):
    """获取网卡 ECN 统计"""
    cmd = f"ethtool -S {interface} | grep -E 'ecn_mark|ecn_total|cnps|cnpt'"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    return result.stdout

def parse_stats(output):
    """解析统计信息"""
    stats = {}
    for line in output.strip().split('\n'):
        if ':' in line:
            key, val = line.split(':')
            stats[key.strip()] = int(val.strip())
    return stats

def main():
    interface = "eth0"
    interval = 5  # 秒
    
    print(f"Monitoring ECN stats on {interface} (interval: {interval}s)")
    print("=" * 50)
    
    try:
        while True:
            output = get_eth_stats(interface)
            stats = parse_stats(output)
            
            print(f"\n[{time.strftime('%H:%M:%S')}]")
            for key, val in stats.items():
                print(f"  {key}: {val:,}")
            
            time.sleep(interval)
    except KeyboardInterrupt:
        print("\nStopped monitoring")

if __name__ == "__main__":
    main()
```

---

## ⚠️ 常见问题排查

### 1. ECN 标记率过低

**症状**: `ecn_mark` 计数很少或为 0

**可能原因**:
- RTT 测量不准确
- 阈值设置过大
- 网络没有实际拥塞

**解决方案**:
```bash
# 1. 重新测量 RTT
ping -s 8972 -M do <target_ip> -c 1000

# 2. 降低阈值 (10-20%)
set random-detect ecn minimum-threshold 50 maximum-threshold 55

# 3. 验证标记
ethtool -S eth0 | grep ecn_mark
```

### 2. CNP 响应延迟过高

**症状**: `cnpt` (CNP Timeout) 计数增加

**可能原因**:
- 接收端 CNP 处理慢
- 网络 RTT 波动大
- ECN 阈值过大导致拥塞严重

**解决方案**:
```bash
# 1. 调整阈值更敏感
set random-detect ecn minimum-threshold 50 maximum-threshold 55

# 2. 检查接收端性能
ethtool -S eth0 | grep cnps

# 3. 启用 CNP 统计
roce_param --set CNP_Enable=1
```

### 3. PFC 风暴

**症状**: 网络中出现大量 PFC pause 帧

**可能原因**:
- ECN 阈值设置过大
- PFC 与 ECN 协同不当
- 队列缓冲区过大

**解决方案**:
```bash
# 1. 确保 ECN 先于 PFC 触发
# 检查：ECN Max < PFC XOFF Threshold

# 2. 调整 ECN 阈值更小
set random-detect ecn minimum-threshold 50 maximum-threshold 55

# 3. 验证配置
show qos ecn
show pfc stats
```

---

## 🎯 最佳实践总结

| 类别 | 建议 |
|------|----|
| **DCQCN 选择** | DCQCN-D: 小集群/低延迟；DCQCN-P: 超大规模 |
| **阈值计算** | Min=BDP*0.4, Max=BDP*0.41 (加 2KB 安全余量) |
| **MTU** | 必须使用 9000 byte jumbo frames |
| **RTT 测量** | 必须模拟真实流量 (MTU=8972 byte) |
| **协同配置** | Switch 和 NIC 阈值必须一致 |
| **监控** | 持续监控 CNP 响应时间和 ECN 标记率 |
| **PFC 优先级** | 仅对 RoCE 流量启用 (优先级 3) |
| **ETS 配置** | RoCE 50%, 其他 50% (避免 QoS 倾斜) |

---

## 📋 参考文档

1. **NVIDIA MLNX_OFED Documentation** - https://docs.nvidia.com/networking/
2. **Broadcom Ethernet Network Adapter User Guide** - RoCE Configuration
3. **Cumulus Linux Traffic Configuration** - ECN & PFC
4. **IEEE 802.1Qbb** - Priority Flow Control Standard
5. **RFC 3168** - Explicit Congestion Notification

---

**维护**: OpenClaw Assistant  
**最后更新**: 2026-03-30 09:54 UTC
