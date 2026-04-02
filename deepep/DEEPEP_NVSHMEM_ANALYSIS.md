# DeepEP NVSHMEM 集成分析

## 概述

DeepEP 是一个专为 MoE (Mixture-of-Experts) 和专家并行 (EP) 设计的高效通信库，通过 NVSHMEM 实现高性能的 GPU 间通信。

---

## 一、NVSHMEM 集成架构

### 1.1 核心依赖

DeepEP 依赖 NVSHMEM v3.3.9+，用于:
- **Intranode 通信**: 通过 NVLink 和 NVSHMEM 实现节点内高速通信
- **Internode 通信**: 通过 RDMA (InfiniBand) 和 NVSHMEM 实现跨节点通信
- **Low-Latency 模式**: 纯 RDMA 通信，适用于推理解码阶段

### 1.2 通信域

```
┌─────────────────────────────────────────────────────────────┐
│                        Intranode                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │    GPU 0    │◄─┤   NVLink    │◄─┤    GPU 1    │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │    GPU 2    │◄─┤   NVLink    │◄─┤    GPU 3    │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼ RDMA / NVSHMEM
┌─────────────────────────────────────────────────────────────┐
│                        Internode                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │    GPU 0    │◄─┤  InfiniBand ├─►│    GPU 0    │          │
│  │   Node 0    │  │   400 Gb/s  │  │   Node 1    │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、NVSHMEM 初始化流程

### 2.1 Python 层初始化

在 `deep_ep/buffer.py` 中，NVSHMEM 的初始化在 `Buffer.__init__` 中完成:

```python
# 设置 NVSHMEM 环境变量
os.environ['NVSHMEM_DISABLE_P2P'] = '0' if allow_nvlink_for_low_latency_mode else '1'
os.environ['NVSHMEM_IB_ENABLE_IBGDA'] = '1'
os.environ['NVSHMEM_IBGDA_NUM_RC_PER_PE'] = f'{num_qps_per_rank}'
os.environ['NVSHMEM_QP_DEPTH'] = str(self.nvshmem_qp_depth)
os.environ['NVSHMEM_MAX_TEAMS'] = '7'
os.environ['NVSHMEM_DISABLE_NVLS'] = '1'
os.environ['NVSHMEM_DISABLE_MNNVL'] = '1' if not allow_mnnvl else '0'
os.environ['NVSHMEM_CUMEM_GRANULARITY'] = f'{2 ** 29}'
```

### 2.2 关键环境变量

| 环境变量 | 作用 | 推荐值 |
|---------|------|--------|
| `NVSHMEM_DISABLE_P2P` | 禁用/启用 P2P | 0 (启用) |
| `NVSHMEM_IB_ENABLE_IBGDA` | 启用 IBGDA 支持 | 1 |
| `NVSHMEM_IBGDA_NUM_RC_PER_PE` | 每 PE 的 RC QP 数量 | 根据 expert 数量 |
| `NVSHMEM_QP_DEPTH` | QP 深度 | 1024 |
| `NVSHMEM_MAX_TEAMS` | 最大 team 数量 | 7 |
| `NVSHMEM_DISABLE_NVLS` | 禁用 NVLink SHArP | 1 |
| `NVSHMEM_DISABLE_MNNVL` | 禁用多节点 NVLink | 1 (不允许多节点 NVLink) |
| `NVSHMEM_CUMEM_GRANULARITY` | 内存粒度 | 256 MiB |

### 2.3 C++ 运行时初始化

NVSHMEM 的初始化通过 C++ 运行时 `deep_ep_cpp.Buffer` 完成:

```python
self.runtime = deep_ep_cpp.Buffer(
    self.rank,              # 本地 rank ID
    self.group_size,        # 通信组大小
    num_nvl_bytes,          # NVLink 缓冲区大小
    num_rdma_bytes,         # RDMA 缓冲区大小
    low_latency_mode,       # 是否启用低延迟模式
    ...
)
```

---

## 三、NVSHMEM 核心 API 调用

### 3.1 数据放 (Puts)

DeepEP 使用 NVSHMEM 的 Puts 进行远程内存写入:

#### 3.1.1 RDMA Write (IBGDA 模式)

在 `ibgda_device.cuh` 中实现了底层的 RDMA Write:

```cuda
__device__ static __forceinline__ void nvshmemi_ibgda_put_nbi_warp(
    uint64_t req_rptr,      // 远程读取指针 (目标地址)
    uint64_t req_lptr,      // 本地写入指针 (源地址)
    size_t bytes,           // 传输字节数
    int dst_pe,             // 目标 PE (远程 GPU)
    int qp_id,              // QP ID
    int lane_id,            // Warp 内的 lane ID
    int message_idx)        // 消息索引
```

**工作流程**:
1. 获取本地/远程 key: `ibgda_get_lkey_and_rkey()`
2. 预留 WQE (Work Queue Entry) 槽位：`ibgda_reserve_wqe_slots()`
3. 写入 WQE: `ibgda_write_rdma_write_wqe()`
4. 提交请求：`ibgda_submit_requests()`
5. 环 Doorbell: `ibgda_ring_db()`

#### 3.1.2 原子操作 (AMOs)

支持原子加法操作，用于计数器:

```cuda
__device__ __forceinline__ void nvshmemi_ibgda_amo_nonfetch_add(
    void* rptr,         // 远程原子操作指针
    const int& value,   // 要加的值
    int pe,             // 目标 PE
    int qp_id,          // QP ID
    bool is_local_copy) // 是否本地副本
```

### 3.2 Barrier 同步

#### 3.2.1 NVLink 内 Barrier

```cuda
// 节点内同步：barrier_block
barrier_block<NUM_MAX_NVL_PEERS, true>(barrier_signal_ptrs, nvl_rank);
```

#### 3.2.2 RDMA Barrier

```cuda
// 跨节点同步：使用 NVSHMEM sync
nvshmem_sync(rdma_team);
nvshmem_sync_all();
```

### 3.3 队列对 (QP) 管理

每个 PE (GPU) 维护多个 RC (Reliable Connection) QPs:

```cpp
// 获取 RC QP 指针
__device__ static __forceinline__ nvshmemi_ibgda_device_qp_t* 
ibgda_get_rc(int pe, int id) {
    auto state = ibgda_get_state();
    return ibgda_get_rc_impl(state, pe, id);
}
```

QP 数量由环境变量 `NVSHMEM_IBGDA_NUM_RC_PER_PE` 配置，低延迟模式下要求 QP 数等于本地专家数。

---

## 四、通信模式

### 4.1 Normal Kernels (高吞吐模式)

**适用场景**: 训练、推理 prefilling

**通信路径**:
- **Intranode**: NVLink (最高 ~160 GB/s)
- **Internode**: RDMA (最高 ~50 GB/s)

**NVSHMEM 调用**:
- `nvshmem_put()` - 数据分发
- `nvshmem_fence()` - 同步屏障
- `nvshmem_quiet()` - 等待所有操作完成

### 4.2 Low-Latency Kernels (低延迟模式)

**适用场景**: 推理 decoding

**通信路径**:
- **纯 RDMA** (IBGDA 模式)
- 利用 NVLink 作为辅助 (最新优化)

**NVSHMEM 调用**:
- `nvshmemi_ibgda_put_nbi_warp()` - 非阻塞 put
- `nvshmemi_ibgda_quiet()` - 等待完成
- 钩子机制实现通信 - 计算重叠

**关键特性**:
- CUDA Graph 兼容
- 零 SM 占用 (通过 hook 实现重叠)
- 支持 double-batch overlapping

---

## 五、IBGDA (InfiniBand GPUDirect Async)

### 5.1 工作原理

IBGDA 允许 GPU 直接异步发起 RDMA 操作，无需 CPU 参与:

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│   GPU 0      │     │  InfiniBand NIC  │     │   GPU 1      │
│              │     │                  │     │              │
│  CUDA Stream │────►│  WQEs (async)    │────►│  Destination │
│              │     │                  │     │              │
│  SMs free    │     │  Doorbell Ring   │     │  Memory copy │
└──────────────┘     └──────────────────┘     └──────────────┘
```

### 5.2 DeepEP 的 IBGDA 实现

在 `ibgda_device.cuh` 中:

```cuda
// 异步 post-send 模式
template <bool kAlwaysDoPostSend>
__device__ static __forceinline__ void 
ibgda_submit_requests(nvshmemi_ibgda_device_qp_t* qp,
                     uint64_t base_wqe_idx,
                     uint32_t num_wqes,
                     int message_idx) {
    auto state = ibgda_get_state();
    nvshmemi_ibgda_device_qp_management_t* mvars = &qp->mvars;
    uint64_t new_wqe_idx = base_wqe_idx + num_wqes;

    // 等待 prior WQE slots
    while (atomicCAS(ready_idx, base_wqe_idx, new_wqe_idx) != base_wqe_idx)
        ;

    // 总是 post，不分批
    if (!state->use_async_postsend) {
        if (kAlwaysDoPostSend or (message_idx + 1) % kNumRequestInBatch == 0)
            ibgda_post_send(qp, new_wqe_idx);
    }
}
```

### 5.3 配置要求

**驱动配置**:
```bash
# /etc/modprobe.d/nvidia.conf
options nvidia NVreg_EnableStreamMemOPs=1 
NVreg_RegistryDwords="PeerMappingOverride=1;"
```

**可选：GDRCopy 支持**:
```bash
# 通过 GPU 内存直接映射提高性能
sudo modprobe gdrdrv
```

---

## 六、内存管理

### 6.1 缓冲区结构

```
┌─────────────────────────────────────────────────────────────┐
│                    RDMA Buffer                               │
├─────────────────────────────────────────────────────────────┤
│  Dispatch Buffers (2 directions × num_channels)             │
├─────────────────────────────────────────────────────────────┤
│  Combine Buffers (2 directions × num_channels)              │
├─────────────────────────────────────────────────────────────┤
│  Metadata (SourceMeta, counters, barriers)                  │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 NVSHMEM 远程内存

NVSHMEM 提供全局虚拟地址空间:
- `nvshmemi_device_state_d.heap_base` - 堆基址
- `nvshmemi_device_state_d.peer_heap_base_remote[]` - 远程堆基址
- `lkeys[]` - 本地内存 key
- `rkeys[]` - 远程内存 key

### 6.3 Rkey 管理

```cuda
__device__ static __forceinline__ void 
ibgda_get_rkey(uint64_t addr,              // 远程地址
               int dst_pe,                 // 目标 PE
               uint64_t* out_raddr,        // 输出：实际远程地址
               __be32* out_rkey,           // 输出：Rkey
               uint32_t dev_idx)           // 设备索引
{
    auto state = ibgda_get_state();
    auto heap_start = reinterpret_cast<uint64_t>(nvshmemi_device_state_d.heap_base);
    uint64_t roffset = addr - heap_start;
    
    // 从常量内存或全局内存获取 rkey
    uint64_t idx = ((roffset >> log2_cumem_granularity) * nvshmemi_device_state_d.npes 
                   * state->num_devices_initialized) +
                   dst_pe * state->num_devices_initialized + dev_idx;
    
    if (idx < NVSHMEMI_IBGDA_MAX_CONST_RKEYS)
        device_key = state->constmem.rkeys[idx];
    else
        device_key = state->globalmem.rkeys[idx - NVSHMEMI_IBGDA_MAX_RKEYS];
    
    *out_raddr = reinterpret_cast<uint64_t>(nvshmemi_device_state_d.peer_heap_base_remote[dst_pe]) + roffset;
    *out_rkey = device_key.key;
}
```

---

## 七、性能优化

### 7.1 Warp 级并行

每个 Warp (32 threads) 负责多个消息:

```cuda
__device__ static __forceinline__ void 
nvshmemi_ibgda_put_nbi_warp(...) {
    // 每个 lane 处理一个 WQE
    uint32_t num_wqes = 0;
    
    while (remaining_bytes > 0) {
        if (lane_id == num_wqes) {
            // 获取 chunk 信息
        }
        
        // 广播到所有 lane
        auto chunk_size = __shfl_sync(0xffffffff, my_chunk_size, 
                                      static_cast<int>(num_wqes));
        remaining_bytes -= chunk_size;
        ++num_wqes;
    }
}
```

### 7.2 共享内存优化

```cuda
// 使用共享内存缓存 lkey/rkey
__shared__ uint64_t shared_lkeys[32];
__shared__ uint64_t shared_rkeys[32];
```

### 7.3 SM 资源管理

**Normal 模式**: 可配置 SM 数量 (`Buffer.set_num_sms(24)`)

**Low-Latency 模式**: 通过 hook 实现零 SM 占用

```python
# 通信 - 计算重叠
recv_hidden_states, hook = buffer.low_latency_dispatch(x, topk_idx, 
                                                        return_recv_hook=True)
# 计算可以在后台进行
compute_hidden_states()
# 然后等待接收完成
hook()
```

---

## 八、错误处理与验证

### 8.1 编译时检查

```cpp
EP_STATIC_ASSERT(NVSHMEMI_IBGDA_MIN_QP_DEPTH >= 64, "Invalid QP minimum depth");
EP_STATIC_ASSERT(sizeof(SourceMeta) % sizeof(int) == 0, "Invalid size of SourceMeta");
```

### 8.2 运行时验证

```python
# Python 层验证
import subprocess
subprocess.run("nvshmem-info -a", shell=True)
```

### 8.3 驱动配置验证

```bash
# 检查驱动配置
cat /sys/module/nvidia/parameters/NVreg_EnableStreamMemOPs
```

---

## 九、总结

### 9.1 NVSHMEM 调用层次

```
┌─────────────────────────────────────────┐
│        Python API (buffer.py)           │
│  - Buffer initialization                │
│  - dispatch() / combine()               │
└─────────────────────────────────────────┘
                 ▼
┌─────────────────────────────────────────┐
│      C++ Runtime (deep_ep_cpp)          │
│  - Buffer allocation                    │
│  - NVSHMEM initialization               │
└─────────────────────────────────────────┘
                 ▼
┌─────────────────────────────────────────┐
│    CUDA Kernels (internode.cu)          │
│  - Dispatch/Combine kernels             │
│  - Barrier synchronization              │
└─────────────────────────────────────────┘
                 ▼
┌─────────────────────────────────────────┐
│   IBGDA Device Layer (ibgda_device.cuh) │
│  - RDMA Write (NVSHMEM put)             │
│  - Atomic operations                    │
│  - QP management                        │
└─────────────────────────────────────────┘
                 ▼
┌─────────────────────────────────────────┐
│      Hardware (InfiniBand NVLink)       │
└─────────────────────────────────────────┘
```

### 9.2 关键设计特点

1. **IBGDA 优先**: 使用 InfiniBand GPUDirect Async 实现零 CPU 参与
2. **Warp 级并行**: 每个 Warp 负责多个 WQE，最大化带宽利用率
3. **原子操作优化**: 使用原子加法实现计数器，减少同步开销
4. **内存 key 缓存**: 使用常量内存和全局内存两级缓存 rkey，减少延迟
5. **异步 post-send**: 支持批量提交 RDMA 请求，降低延迟
6. **零 SM 占用**: 通过 hook 实现通信 - 计算完全重叠

### 9.3 适用场景

| 模式 | 场景 | NVSHMEM 调用 | 性能 |
|------|------|-------------|------|
| **Normal + NVLink** | Intranode | `nvshmem_put`, `nvshmem_fence` | 153 GB/s |
| **Normal + RDMA** | Internode | `nvshmem_put`, `nvshmem_quiet` | 43-58 GB/s |
| **Low-Latency** | Inference | `nvshmemi_ibgda_put_nbi_warp` | 98 GB/s (8 EP) |

---

**生成时间**: 2026-04-02  
**数据来源**: DeepEP GitHub 仓库 v1.2+  
**NVSHMEM 版本**: 3.3.9+
