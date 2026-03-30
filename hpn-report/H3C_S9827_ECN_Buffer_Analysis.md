# H3C S9827 交换机 ECN 拥塞控制水位与 Buffer 关系分析

> 日期：2026-03-30  
> 来源：H3C S6805 & S6825 & S6850 & S9850 & S9820 RDMA Technical Reference-6W100

---

## 核心发现

### 1. ECN 触发条件

ECN 的触发基于 **WRED 队列长度阈值**：

| 队列长度 | 处理动作 |
|---------|---------|
| **< low-limit** | 直接转发，不标记 |
| **low-limit ~ high-limit** | 根据丢包概率**标记** ECN 位（ECT 00 → 11），**不丢包** |
| **> high-limit** | 非 ECN 包：直接丢包；ECN 包：标记后继续转发 |

### 2. ECN 与 PFC Buffer 的优先级关系

**关键配置要求**：

> "For ECN to take effect before PFC, make sure the **back pressure frame triggering threshold** (cell resources) is **greater than the ECN high-limit value**."

```
ECN 高阈值 < PFC 背压帧触发阈值 (XOFF)
```

如果 ECN 高阈值 > PFC XOFF，PFC 会先触发，ECN 无法发挥拥塞通知作用。

### 3. S9820/S9827 Buffer 架构

```
Buffer 空间分布:
├── Fixed Area (固定区域)
│   └── 每个队列独立分配，不可抢占
├── Shared Area (共享区域)  
│   ├── Tail Drop Threshold (尾丢阈值，默认 20%)
│   └── 动态共享分配，可抢占
└── Headroom Area (头存区域)
    └── PFC 预留空间 (XOFF 触发)
        - 默认：S9820-64H=28672 cells
        - 默认：S9820-8C=12288 cells
```

### 4. Cell 资源配置 (S9820-64H)

| 参数 | 值 |
|------|---|
| Cell 资源大小 | 208 bytes |
| 总 buffer | ~204K cells |
| 默认共享区比例 | 20% |
| 默认固定区比例 | 13% |

### 5. 动态背压触发阈值参考表

根据流量占比动态调整背压阈值：

| Alpha 值 | 缓冲占比 | 动态阈值 (%) |
|---------|---------|------------|
| 1/128 | 0.77% | 0 |
| 1/64 | 1.53% | 1 |
| 1/32 | 3.03% | 2-3 |
| 1/16 | 5.88% | 4-5 |
| 1/8 | 11.11% | 6-11 |
| 1/4 | 20.00% | 12-20 |
| 1/2 | 33.33% | 21-33 |
| 1 | 50.00% | 34-50 |
| 2 | 66.66% | 51-66 |
| 4 | 80.00% | 67-80 |
| 8 | 88.88% | 81-100 |

---

## 配置示例

### WRED ECN 配置

```bash
# 创建 WRED 表
qos wred queue table queue-table5
  queue 5 weighting-constant 12
  queue 5 drop-level 0 low-limit 10 high-limit 20 discard-probability 30
  queue 5 ecn
  quit

# 应用 WRED 表到接口
interface twenty-fivegige 1/0/3
  qos wred apply queue-table5
```

### PFC Buffer 阈值配置

```bash
# 确保 ECN 先于 PFC 触发
# 背压触发阈值 > ECN high-limit (20 cells)

interface twenty-fivegige 1/0/3
  # 动态背压阈值设为 33%
  priority-flow-control dot1p 5 ingress-buffer dynamic 33
  
  # 确保 33% > ECN high-limit 20 cells
  priority-flow-control no-drop dot1p 5
```

### Buffer 空间分配

```bash
# 为特定队列分配 buffer 空间
interface twenty-fivegige 1/0/3
  # 固定区域 15%
  buffer egress cell queue 5 guaranteed ratio 15
  
  # 共享区域 25%
  buffer egress cell queue 5 shared ratio 25
```

---

## 监控命令

```bash
# 查看 buffer 使用详情
display buffer usage interface twenty-fivegige 1/0/1 verbose

# 查看 ECN 标记和 WRED 丢包统计
display packet-drop interface twenty-fivegige 1/0/1

# 查看 PFC 配置和状态
display priority-flow-control interface twenty-fivegige 1/0/1
```

---

## 最佳实践

1. **ECN 先触发**：确保 PFC 背压阈值 > ECN high-limit
2. **buffer 分配**：RDMA 流量队列建议共享区 100%
3. **监控告警**：启用 buffer 使用率告警 (默认 70%)
4. **PFC  deadlock**：启用 PFC deadlock detection，间隔 50ms

---

**维护**: OpenClaw Assistant  
**最后更新**: 2026-03-30 06:26 UTC
