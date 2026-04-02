# DeepEP Normal 模式 Dispatch 阶段 SM/Warp/Lane 使用分析

## 概述

DeepEP 的 normal 模式 dispatch 是一个复杂的流水线式通信过程，涉及 RDMA 和 NVLink 两级传输。通过精细的 SM、warp、lane 划分，实现高吞吐和高效并行。

---

## 一、Kernel 基本信息

### 1.1 核心 Kernel

**Kernel 名称**: `dispatch` (internode.cu)  
**Launch 配置**: 
- **SMs**: `num_sms` = `num_channels * 2` (channel_id = sm_id / 2)
- **Threads per SM**: 512 线程
- **Warps per SM**: 16 warps (512 / 32)

### 1.2 Warp 角色划分

每个 SM 的 16 个 warp 被划分为 4 类角色:

| Warp 范围 | 数量 | 角色 | 职责 |
|---------|------|------|------|
| warp 0~kNumDispatchRDMASenderWarps | `kNumDispatchRDMASenderWarps` | **RDMA Senders** | 负责 RDMA 数据发送 |
| warp kNumDispatchRDMASenderWarps | 1 | **RDMA Sender Coordinator** | RDMA 发送流控协调 |
| warp kNumDispatchRDMASenderWarps+1 ~ kNumDispatchRDMASenderWarps+NUM_MAX_NVL_PEERS | `NUM_MAX_NVL_PEERS` (8) | **RDMA & NVL Forwarders** | RDMA 接收 + NVL 发送 |
| warp kNumDispatchRDMASenderWarps+NUM_MAX_NVL_PEERS+1 ~ 15 | `NUM_MAX_NVL_PEERS` (8) | **NVL Receivers** | NVL 数据接收 |

**默认参数**:
- `kNumDispatchRDMASenderWarps` = 7 (8 EP 时) 或 3 (4 EP 时)
- `NUM_MAX_NVL_PEERS` = 8

**示例 (8 EP)**:
```
SM 总 warp: 16
├─ RDMA Senders:          warp 0-6  (7 warps)
├─ RDMA Coordinator:      warp 7   (1 warp)
├─ Forwarders:            warp 8-15 (8 warps)
└─ NVL Receivers:         warp 16-23 (8 warps, 但实际只占用 8 个)
```

---

## 二、dispatch 阶段详解

### 阶段 1: RDMA 发送 (RDMA Sender Warps)

**活跃 Warp**: `warp 0 ~ kNumDispatchRDMASenderWarps-1`  
**Lane 使用**: 0~31 (全部分配)

#### 1.1 任务分配
```cpp
// 每个 warp 负责特定 channel 的 RDMA 发送
auto channel_id = sm_id / 2;
auto dst_rdma_rank = warp_id;  // warp_id = dst_rdma_rank (一对一映射)
```

#### 1.2 Lane 分工
每个 warp 的 32 个 lane 用于:
- **Lane 0~NUM_MAX_NVL_PEERS-1 (0~7)**: 
  - 发送 `gbl_channel_prefix_matrix` 的 lane 0-7
  - 计算 gbl channel 起始偏移

- **Lane NUM_MAX_NVL_PEERS~NUM_MAX_NVL_PEERS*2-1 (8~15)**:
  - 发送 gbl_channel_prefix_matrix 的 lane 8-15
  - 计算 gbl channel 结束偏移

- **Lane NUM_MAX_NVL_PEERS*2 (16)**:
  - 发送 rdma_channel_prefix_matrix 的前缀
  - 计算 RDMA channel 前缀和

- **Lane NUM_MAX_NVL_PEERS*2+1 (17)**:
  - 发送 rdma_channel_prefix_matrix 的后缀
  - 计算 RDMA channel 后缀和

#### 1.3 核心操作
```cuda
// 数据分发到多个目标 RDMA rank
for (int i = lane_id; i < num_topk_ranks; i += 32) {
    // Copy x, x_scales, src_meta, topk_idx, topk_weights
    // 每个 lane 负责一个 lane 的数据拷贝
}

// 流控机制
if (is_token_in_rank_uint64 != 0) {
    // 使用锁和窗口机制管理 RDMA 发送
    acquire_lock(rdma_send_channel_lock + lane_id);
    // 检查远程缓冲区槽位
    // 提交 RDMA 写请求
    release_lock(rdma_send_channel_lock + lane_id);
}
```

#### 1.4 特点
- **Warp 级并行**: 每个 warp 独立处理一个 RDMA rank
- **Lane 级并行**: 每个 lane 负责不同部分的数据拷贝
- **流控**: 使用锁和窗口机制避免缓冲区溢出

---

### 阶段 2: RDMA 发送协调 (RDMA Sender Coordinator)

**活跃 Warp**: `warp kNumDispatchRDMASenderWarps`  
**Lane 使用**: 0~31 (全部分配)

#### 2.1 职责
- **共享内存清理**: 初始化锁、tail、window 状态
- **任务分发**: 计算每个 RDMA rank 需要发送的 token 数量
- **流控协调**: 通过 `nvshmemi_ibgda_put_nbi_warp` 批量提交 RDMA 写请求
- **尾指针更新**: 使用原子操作更新远程 tail 指针

#### 2.2 Lane 分工
- **Lane 0~kNumRDMARanks-1 (0~7)**:
  - 每个 lane 负责一个 RDMA rank 的流控
  - 读取和处理该 rank 的发送进度
  - 提交 RDMA 写请求
  - 更新 remote tail

#### 2.3 核心逻辑
```cuda
// 协调所有 RDMA senders
while (__any_sync(0xffffffff, num_tokens_to_send > 0)) {
    for (int i = 0, synced_num_tokens_to_send; i < kNumRDMARanks; ++i) {
        // 读取每个 RDMA rank 的发送进度
        auto num_tokens_processed = ...;
        
        // 检查是否满足发送条件
        if (num_tokens_processed != synced_num_tokens_to_send)
            continue;
        
        // 提交 RDMA 写请求
        nvshmemi_ibgda_put_nbi_warp<true>(...);
        
        // 更新尾指针 (原子操作)
        nvshmemi_ibgda_amo_nonfetch_add(rdma_channel_tail.buffer(rdma_rank), ...);
    }
}
```

#### 2.4 特点
- **集中式流控**: 避免多个 warp 竞争共享资源
- **原子操作**: 通过 AMO 更新尾指针，避免锁
- **批量提交**: 每个循环提交一个 token 块

---

### 阶段 3: RDMA & NVL Forwarder (RDMAAndNVLForwarder)

**活跃 Warp**: `warp kNumDispatchRDMASenderWarps+1 ~ kNumDispatchRDMASenderWarps+NUM_MAX_NVL_PEERS`  
**数量**: 8 warps (对应 8 个 NVL peers)  
**Lane 使用**: 0~31 (全部分配)

#### 3.1 职责
- **RDMA 接收**: 从 RDMA 缓冲区读取 token 数据
- **NVL 发送**: 将 token 通过 NVLink 转发到目标 NVL peer
- **智能路由**: 根据 `SourceMeta` 判断 token 的目标 rank

#### 3.2 Lane 分工
- **Lane 0~kNumRDMARanks-1 (0~7)**:
  - 监听 RDMA 通道元数据 (前缀/后缀信息)
  - 计算本 warp 需要处理的 token 范围
  - 管理 RDMA 接收队列的头尾指针

- **Lane 8~31**:
  - 执行 TMA 数据传输
  - 通过 TMA 加速器从 RDMA 缓冲区加载数据
  - 通过 TMA 加速器写入 NVL 发送缓冲区

#### 3.3 核心逻辑
```cuda
// 1. 等待 RDMA 元数据就绪
while (true) {
    meta_0 = rdma_channel_meta.recv_buffer(lane_id) + dst_nvl_rank;
    meta_1 = ...;
    if (meta_0 < 0 && meta_1 < 0 && meta_2 < 0 && meta_3 < 0)
        break;  // 元数据就绪
}

// 2. 计算 token 范围
int num_tokens_to_recv_from_rdma = -meta_2 - 1 - (-meta_3 - 1);

// 3. 转发 token
while (num_tokens_to_recv_from_rdma > 0) {
    // 查找下一个 RDMA rank (轮询)
    src_rdma_rank = (src_rdma_rank + 1) % kNumRDMARanks;
    
    // 等待 RDMA 数据就绪
    while (cached_rdma_channel_head == cached_rdma_channel_tail)
        ;
    
    // 通过 TMA 传输数据
    if (elect_one_sync()) {
        tma_load_1d(tma_buffer, rdma_buffer, num_bytes_per_token);
    }
    __syncwarp();
    mbarrier_wait(tma_mbarrier, tma_phase);
    if (elect_one_sync()) {
        tma_store_1d(tma_buffer, nvl_buffer, num_bytes_per_token);
    }
    
    // 更新流控指针
    if (lane_id == src_rdma_rank)
        num_tokens_to_recv_from_rdma -= 1;
}
```

#### 3.4 特点
- **流水线并行**: TMA 加载/存储与计算重叠
- **轮询调度**: 公平地从多个 RDMA rank 获取 token
- **流控**: 使用 mbarrier 管理 TMA 完成状态

---

### 阶段 4: Forwarder Coordinator

**活跃 Warp**: `warp kNumDispatchRDMASenderWarps+NUM_MAX_NVL_PEERS+1`  
**数量**: 1 warp (target_rank=0)  
**Lane 使用**: 0~31 (全部分配)

#### 4.1 职责
- **全局流控协调**: 监控所有 Forwarder 的进度
- **RDMA 释放**: 通知 RDMA rank 哪些缓冲区已释放
- **流控推进**: 通过原子操作更新 RDMA head 指针

#### 4.2 Lane 分工
- **Lane 0~kNumRDMARanks-1 (0~7)**:
  - 读取 local Forwarder 的 head 指针
  - 计算最小 head (阻塞点)
  - 通过 AMO 推进 RDMA head 指针

#### 4.3 核心逻辑
```cuda
while (true) {
    // 找出所有 Forwarder 中最小的 head
    int min_head = std::numeric_limits<int>::max();
    for (int i = 0; i < NUM_MAX_NVL_PEERS; ++i)
        if (!forward_channel_retired[i])
            min_head = min(min_head, forward_channel_head[i][target_rdma]);
    
    // 如果所有 Forwarder 都完成了，退出
    if (__all_sync(0xffffffff, min_head == max))
        break;
    
    // 推进 RDMA head (允许 RDMA rank 释放缓冲区)
    if (min_head >= last_head + num_max_rdma_chunked_send_tokens) {
        nvshmemi_ibgda_amo_nonfetch_add(
            rdma_channel_head.buffer(rdma_rank),
            min_head - last_head,
            channel_id + num_channels,  // 控制通道
            lane_id == rdma_rank);
        last_head = min_head;
    }
    
    // 让其他 warps 继续工作
    __nanosleep(NUM_WAIT_NANOSECONDS);
}
```

#### 4.4 特点
- **最小 head 策略**: 确保 RDMA 缓冲区不会过早被覆盖
- **非阻塞推进**: 仅在必要时更新 head 指针
- **协同等待**: 通过 nanosleep 避免频繁轮询

---

### 阶段 5: NVL Receiver (NVLReceivers)

**活跃 Warp**: `warp kNumDispatchRDMASenderWarps+NUM_MAX_NVL_PEERS+2 ~ 15`  
**数量**: `NUM_MAX_NVL_PEERS-1` (7 warps, 其中一个 warp 被 Forwarder Coordinator 占用)  
**Lane 使用**: 0~31 (全部分配)

#### 5.1 职责
- **NVL 接收**: 从 NVLink 接收 token 数据
- **数据重组**: 根据 SourceMeta 将 token 重组到本地
- **Top-k 处理**: 过滤和转换 top-k 索引

#### 5.2 Lane 分工
- **Lane 0~kNumRDMARanks-1 (0~7)**:
  - 监听 NVL 通道元数据
  - 计算本 lane 需要接收的 token 范围
  - 管理 NVL 接收队列的头尾指针

- **Lane 8~31**:
  - 通过 TMA 加速器从 NVL 缓冲区读取数据
  - 将数据写入本地 receive buffer
  - 转换 top-k 索引

#### 5.3 核心逻辑
```cuda
// 1. 等待 NVL 元数据
while (lane_id < kNumRDMARanks) {
    start_offset = ld_volatile_global(nvl_channel_prefix_start);
    end_offset = ld_volatile_global(nvl_channel_prefix_end);
    if (start_offset < 0 && end_offset < 0)
        break;
}

// 2. 计算 token 范围
int num_tokens_to_recv = warp_reduce_sum(end_offset - start_offset);
total_offset += start_offset;

// 3. 接收 token
while (num_tokens_to_recv > 0) {
    // 等待 NVL 数据就绪
    while (cached_channel_head_idx == cached_channel_tail_idx)
        ;
    
    // 通过 TMA 接收数据
    for (int chunk_idx = 0; chunk_idx < num_recv_tokens; ++chunk_idx) {
        auto shifted = nvl_channel_x.buffer() + token_idx_in_buffer * num_bytes_per_token;
        
        // 加载数据
        if (elect_one_sync()) {
            tma_load_1d(tma_buffer, shifted, tma_mbarrier, tma_load_bytes);
        }
        __syncwarp();
        mbarrier_wait(tma_mbarrier, tma_phase);
        
        // 存储数据
        if (elect_one_sync()) {
            tma_store_1d(tma_buffer, recv_x + recv_token_idx * hidden_int4, hidden_bytes);
        }
        __syncwarp();
        
        // 处理 top-k
        if (lane_id < num_topk) {
            auto idx_value = ld_nc_global(topk_idx + lane_id);
            auto weight_value = ld_nc_global(topk_weights + lane_id);
            
            // 转换索引
            idx_value = (idx_value >= local_expert_begin && idx_value < local_expert_end) 
                        ? idx_value - local_expert_begin : -1;
            weight_value = idx_value >= 0 ? weight_value : 0.0f;
            
            st_na_global(recv_topk_idx + recv_idx, idx_value);
            st_na_global(recv_topk_weights + recv_idx, weight_value);
        }
    }
}
```

#### 5.4 特点
- **TMA 优化**: 使用 TMA 加速器加速数据传输
- **流控**: 使用原子操作更新 head/tail
- **并行处理**: 每个 lane 独立处理不同部分的数据

---

## 三、SM 级并行

### 3.1 Channel 划分

**Channel 数量**: `num_channels = num_sms / 2`  
**Channel ID**: `channel_id = sm_id / 2`  
**配对关系**:
- `sm_id = 2*channel_id` 和 `sm_id = 2*channel_id+1` 共享 channel_id
- 前者负责 Forwarder 角色
- 后者负责 Receiver 角色

### 3.2 SM 任务分布

| SM ID | 角色 | 职责 |
|------|------|------|
| `sm_id % 2 == 0` | **Forwarder** | RDMA 接收 + NVL 发送 |
| `sm_id % 2 == 1` | **Receiver** | NVL 数据接收 |

### 3.3 数据流

```
SM (Forwarder)                        SM (Receiver)
┌───────────────────┐                ┌───────────────────┐
│ RDMA Receive     │◄─── Data ─────►│ NVL Receive      │
│ (warp 0-7)       │                │ (warp 8-15)      │
│                  │                │                   │
│ NVL Send         │                │                   │
│ (warp 8-15)      │                │                   │
└───────────────────┘                └───────────────────┘
```

---

## 四、Warp 并行度

### 4.1 默认配置

以 8 EP 为例:

```
SM 配置:
├─ Warp 0-6:        RDMA Senders (7 warps)
│   ├─ Lane 0-7:    gbl_prefix lane 0-7
│   ├─ Lane 8-15:   gbl_prefix lane 8-15
│   ├─ Lane 16:     rdma_prefix start
│   └─ Lane 17:     rdma_prefix end
│
├─ Warp 7:          RDMA Coordinator (1 warp)
│   └─ Lane 0-7:    协调 8 个 RDMA rank
│
├─ Warp 8-15:       Forwarders (8 warps)
│   ├─ Lane 0-7:    RDMA meta 监听
│   └─ Lane 8-31:   TMA 数据传输
│
└─ Warp 16-23:      NVL Receivers (7 warps)
    ├─ Lane 0-7:    NVL meta 监听
    └─ Lane 8-31:   TMA 数据接收
```

### 4.2 并行度分析

**RDMA Senders**: 7 warps 并行处理 7 个 RDMA rank  
**Forwarders**: 8 warps 并行处理 8 个 NVL peers  
**NVL Receivers**: 7 warps 并行接收 7 个 NVL peers  

**总并行度**: 22 warps × 32 threads = 704 threads

---

## 五、Lane 级优化

### 5.1 Lane 协作模式

#### 模式 1: 广播/规约
```cuda
// 每个 lane 持有一个值
auto value = __shfl_sync(0xffffffff, my_value, lane_id);
// 规约求和
int total = warp_reduce_sum(value);
```

#### 模式 2: 分块处理
```cuda
// 每个 lane 处理不同块
#pragma unroll
for (int i = lane_id; i < num_elements; i += 32) {
    process(i);
}
```

#### 模式 3: 流水线传输
```cuda
// TMA 加载/存储
if (elect_one_sync()) {
    tma_load_1d(tma_buffer, src, bytes);
}
__syncwarp();
mbarrier_wait(tma_mbarrier, phase);
if (elect_one_sync()) {
    tma_store_1d(tma_buffer, dst, bytes);
}
```

### 5.2 Lane 同步机制

| 同步点 | 机制 | 用途 |
|------|------|------|
| `__syncwarp()` | Warp 级屏障 | 确保所有 lane 完成 |
| `elect_one_sync()` | 选举同步 | 确保一个 lane 执行 |
| `mbarrier_init/wait` | 屏障计数器 | TMA 完成同步 |
| `__syncwarp()` | Warp 级屏障 | 等待 TMA 完成 |

---

## 六、共享内存使用

### 6.1 关键共享内存

```cpp
// RDMA sender 同步
__shared__ int rdma_send_channel_lock[kNumRDMARanks];        // 发送锁
__shared__ int rdma_send_channel_tail[kNumRDMARanks];        // 发送尾指针
__shared__ uint32_t rdma_send_channel_window[kNumRDMARanks]; // 发送窗口

// Forwarder 同步
__shared__ volatile int forward_channel_head[NUM_MAX_NVL_PEERS][kNumRDMARanks];  // 发送头
__shared__ volatile bool forward_channel_retired[NUM_MAX_NVL_PEERS];             // 完成状态

// TMA 缓冲区
extern __shared__ __align__(1024) uint8_t smem_tma_buffer[];
auto tma_buffer = smem_tma_buffer + target_rank * kNumTMABytesPerWarp;
auto tma_mbarrier = reinterpret_cast<uint64_t*>(tma_buffer + num_bytes_per_token);
```

### 6.2 共享内存总量

| 用途 | 大小 | 说明 |
|------|------|------|
| RDMA sender 状态 | `kNumRDMARanks × 12 bytes` | 锁、尾、窗口 |
| Forwarder 状态 | `NUM_MAX_NVL_PEERS × kNumRDMARanks × 4 + NUM_MAX_NVL_PEERS` bytes | 头、完成标志 |
| TMA 缓冲区 | `NUM_MAX_NVL_PEERS × kNumTMABytesPerWarp` bytes | TMA 加载/存储 |

---

## 七、性能优化要点

### 7.1 Warp 负载均衡

- **RDMA Senders**: 每个 warp 负责一个 RDMA rank，负载均衡
- **Forwarders**: 8 warps 对应 8 NVL peers，无竞争
- **NVL Receivers**: 7 warps 并行接收，无竞争

### 7.2 TMA 优化

- **向量化加载**: 使用 `int4` 加载 16 字节数据
- **异步传输**: TMA 加载/存储与计算重叠
- **屏障同步**: 精确控制 TMA 完成时序

### 7.3 流控优化

- **窗口机制**: RDMA sender 使用滑动窗口管理发送
- **原子操作**: 通过 AMO 更新尾指针，避免锁
- **流控协调**: 专用 warp 管理全局流控，避免竞争

### 7.4 流水线并行

- **Stage 1**: RDMA 发送 → **Stage 2**: Forwarder → **Stage 3**: NVL 接收
- 三个 stage 并行执行，实现端到端流水线

---

## 八、总结

### 8.1 SM/Warp/Lane 分工矩阵

| 阶段 | SM 范围 | Warp 范围 | Lane 分工 | 主要职责 |
|------|--------|---------|---------|---------|
| **RDMA Sender** | 全部 | 0~kNumDispatchRDMASenderWarps | 0~17 (元数据), 18~31 (数据) | RDMA 数据发送 |
| **RDMA Coordinator** | 全部 | kNumDispatchRDMASenderWarps | 0~kNumRDMARanks-1 | 流控协调 |
| **Forwarder** | 0,2,4... | kNumDispatchRDMASenderWarps+1~NUM_MAX_NVL_PEERS+7 | 0~kNumRDMARanks-1 (元数据), 8~31 (TMA) | RDMA 接收 + NVL 发送 |
| **Forwarder Coordinator** | 全部 | kNumDispatchRDMASenderWarps+NUM_MAX_NVL_PEERS | 0~kNumRDMARanks-1 | 全局流控 |
| **NVL Receiver** | 1,3,5... | kNumDispatchRDMASenderWarps+NUM_MAX_NVL_PEERS+2~15 | 0~kNumRDMARanks-1 (元数据), 8~31 (TMA) | NVL 数据接收 |

### 8.2 关键设计特点

1. **Warp 级角色分离**: 每类任务由专用 warp 处理，无角色重叠
2. **Lane 级并行**: 每个 lane 负责不同部分，最大化并行度
3. **TMA 加速**: 使用 TMA 加速器处理数据加载/存储，解放 SM
4. **精细流控**: 多层流控机制 (warp, lock, atomic, window)
5. **流水线并行**: 三个 stage 并行执行，实现端到端高吞吐

### 8.3 性能数据

以 8 EP, 512 SMs 为例:
- **总 SMs**: 16
- **总 Warps**: 16 × 16 = 256 warps
- **总 Threads**: 256 × 32 = 8192 threads
- **RDMA Senders**: 7 warps/SM × 16 SMs = 112 warps
- **Forwarders**: 8 warps/SM × 8 SMs = 64 warps
- **NVL Receivers**: 7 warps/SM × 8 SMs = 56 warps

---

**生成时间**: 2026-04-02  
**数据来源**: DeepEP 源代码 `/home/xiegang/src/DeepEP/csrc/kernels/internode.cu`
