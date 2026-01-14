# Skype 去中心化 P2P 架构深度解析

> 本文档系统性地分析 Skype 早期（2003-2011）的去中心化 P2P 架构设计，涵盖核心理论、关键技术和工程实践方法，供从事去中心化 P2P 系统研发的团队参考。

---

## 目录

1. [架构概述](#1-架构概述)
2. [网络拓扑与节点分类](#2-网络拓扑与节点分类)
3. [NAT 穿透技术](#3-nat-穿透技术)
4. [Overlay 网络与路由机制](#4-overlay-网络与路由机制)
5. [用户发现与搜索机制](#5-用户发现与搜索机制)
6. [安全与加密机制](#6-安全与加密机制)
7. [实时通信核心流程](#7-实时通信核心流程)
8. [工程实践要点](#8-工程实践要点)
9. [架构演进与经验教训](#9-架构演进与经验教训)
10. [参考文献](#10-参考文献)

---

## 1. 架构概述

### 1.1 设计背景与目标

Skype 于 2003 年由 Niklas Zennström 和 Janus Friis 创立，其核心技术团队来自 Kazaa（P2P 文件共享系统）。Skype 的架构目标是：

- **去中心化**：最小化中心服务器依赖，降低运营成本
- **NAT 友好**：支持位于各种 NAT/防火墙后的用户
- **高可用性**：无单点故障，网络自愈能力强
- **端到端加密**：保障通信安全与隐私

### 1.2 架构全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Skype P2P Network                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌──────────┐     ┌──────────┐     ┌──────────┐              │
│    │  Super   │◄───►│  Super   │◄───►│  Super   │              │
│    │  Node A  │     │  Node B  │     │  Node C  │              │
│    └────┬─────┘     └────┬─────┘     └────┬─────┘              │
│         │                │                │                     │
│    ┌────┴────┐      ┌────┴────┐      ┌────┴────┐               │
│    │         │      │         │      │         │               │
│  ┌─┴─┐ ┌─┴─┐     ┌─┴─┐ ┌─┴─┐     ┌─┴─┐ ┌─┴─┐                 │
│  │ON1│ │ON2│     │ON3│ │ON4│     │ON5│ │ON6│                 │
│  └───┘ └───┘     └───┘ └───┘     └───┘ └───┘                 │
│   Ordinary Nodes (普通节点)                                     │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                    Central Services (最小化)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Login Server│  │ Bootstrap  │  │  Payment    │             │
│  │  (认证服务)  │  │  Server    │  │  Server     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 核心设计原则

| 原则 | 描述 |
|------|------|
| **混合 P2P** | 结合纯 P2P 和 Client-Server 模式的优点 |
| **Super Node 选举** | 动态选择网络条件优越的节点作为超级节点 |
| **多层 Overlay** | 使用分层网络结构优化路由效率 |
| **自适应** | 根据网络条件动态调整策略 |

---

## 2. 网络拓扑与节点分类

### 2.1 节点类型

Skype 网络中存在三种节点类型：

#### 2.1.1 普通节点 (Ordinary Node, ON)

普通节点是网络中的基本单元：
- 位于 NAT/防火墙后的客户端
- 不具备公网 IP 或足够的带宽/计算资源
- 必须连接到至少一个超级节点才能使用网络
- 维护与超级节点的心跳连接

```python
class OrdinaryNode:
    def __init__(self):
        self.super_node = None          # 连接的超级节点
        self.host_cache = []            # 超级节点缓存列表
        self.nat_type = None            # NAT 类型
        self.buddy_list = []            # 好友列表
        
    def bootstrap(self):
        """启动时连接超级节点"""
        for sn in self.host_cache:
            if self.try_connect(sn):
                self.super_node = sn
                break
        if not self.super_node:
            # 使用 Bootstrap Server 获取超级节点列表
            self.host_cache = self.fetch_from_bootstrap_server()
            self.bootstrap()
```

#### 2.1.2 超级节点 (Super Node, SN)

超级节点是网络的骨干：

**选举条件**：
- 具有公网 IP 地址（非 NAT 或 Full Cone NAT）
- 足够的 CPU 和内存资源
- 高带宽上行连接（> 1 Mbps）
- 长时间在线（uptime > 数小时）
- 未被用户禁用超级节点功能

**核心职责**：
- 维护连接的普通节点列表
- 存储用户在线状态和路由信息
- 参与全局搜索查询转发
- 协助 NAT 穿透（作为中继）
- 参与 Overlay 网络路由

```python
class SuperNode:
    def __init__(self):
        self.connected_nodes = {}       # node_id -> NodeInfo
        self.routing_table = {}         # 分布式路由表
        self.peer_super_nodes = []      # 其他超级节点
        
    def handle_search_query(self, query):
        """处理搜索查询"""
        # 首先搜索本地连接的节点
        local_results = self.search_local(query)
        
        # 转发给其他超级节点
        for sn in self.peer_super_nodes:
            remote_results = sn.forward_query(query)
            local_results.extend(remote_results)
            
        return local_results
    
    def register_node(self, node):
        """注册普通节点"""
        self.connected_nodes[node.id] = {
            'endpoint': node.endpoint,
            'nat_type': node.nat_type,
            'last_seen': time.now(),
            'username': node.username
        }
```

#### 2.1.3 中继节点 (Relay Node)

当两个节点无法建立直接连接时，使用中继：

```
  Client A                Relay Node               Client B
     │                        │                        │
     │    Media Stream        │    Media Stream        │
     │ ──────────────────────>│ ──────────────────────>│
     │<────────────────────── │<────────────────────── │
     │                        │                        │
```

### 2.2 超级节点选举算法

```python
def calculate_super_node_score(node):
    """计算节点成为超级节点的得分"""
    score = 0
    
    # 网络条件
    if node.has_public_ip:
        score += 100
    elif node.nat_type == 'FULL_CONE':
        score += 80
    elif node.nat_type == 'ADDRESS_RESTRICTED':
        score += 40
    
    # 带宽
    score += min(node.upload_bandwidth_kbps / 10, 50)
    
    # 在线时间
    score += min(node.uptime_hours * 2, 50)
    
    # CPU 可用率
    score += node.cpu_available_percent * 0.5
    
    # 用户设置
    if node.user_disabled_sn:
        score = 0
        
    return score

def elect_super_nodes(network_nodes, threshold=200):
    """选举超级节点"""
    super_nodes = []
    for node in network_nodes:
        if calculate_super_node_score(node) >= threshold:
            super_nodes.append(node)
    return super_nodes
```

### 2.3 节点状态机

```
                    ┌───────────────────────┐
                    │      OFFLINE          │
                    └───────────┬───────────┘
                                │ start
                                ▼
                    ┌───────────────────────┐
                    │    BOOTSTRAPPING      │
                    └───────────┬───────────┘
                                │ connected to SN
                    ┌───────────┴───────────┐
                    ▼                       ▼
        ┌───────────────────┐   ┌───────────────────┐
        │   ORDINARY_NODE   │   │    SUPER_NODE     │
        │   (普通节点模式)   │   │   (超级节点模式)   │
        └───────────────────┘   └───────────────────┘
                    │                       │
                    │ SN election           │ demotion
                    └───────────►───────────┘
```

---

## 3. NAT 穿透技术

NAT 穿透是 Skype 成功的关键技术之一。

### 3.1 NAT 类型分类

```
┌─────────────────────────────────────────────────────────────────┐
│                        NAT 类型分类                              │
├─────────────────┬───────────────────────────────────────────────┤
│ Full Cone       │ 最宽松，任何外部主机可通过映射端口访问内部主机    │
├─────────────────┼───────────────────────────────────────────────┤
│ Address         │ 只有内部主机曾通信过的外部 IP 可以响应          │
│ Restricted Cone │                                               │
├─────────────────┼───────────────────────────────────────────────┤
│ Port Restricted │ 只有内部主机曾通信过的 IP:Port 可以响应         │
│ Cone            │                                               │
├─────────────────┼───────────────────────────────────────────────┤
│ Symmetric NAT   │ 对不同目标使用不同的外部端口，最难穿透          │
└─────────────────┴───────────────────────────────────────────────┘
```

### 3.2 NAT 类型检测 (STUN)

```python
class STUNClient:
    def detect_nat_type(self, stun_server):
        """使用 STUN 协议检测 NAT 类型"""
        
        # Test 1: 基本连通性测试
        response1 = self.send_binding_request(stun_server)
        if not response1:
            return 'BLOCKED'
        
        mapped_addr = response1.mapped_address
        local_addr = self.local_address
        
        if mapped_addr == local_addr:
            # 没有 NAT，公网 IP
            return 'OPEN_INTERNET'
        
        # Test 2: 测试不同 IP 是否可达
        response2 = self.send_binding_request(
            stun_server, 
            change_ip=True, 
            change_port=True
        )
        
        if response2:
            return 'FULL_CONE'
        
        # Test 3: 相同 IP 不同端口
        response3 = self.send_binding_request(
            stun_server,
            change_ip=False,
            change_port=True
        )
        
        if response3:
            return 'ADDRESS_RESTRICTED_CONE'
        
        # Test 4: 使用不同 STUN 服务器测试端口映射
        response4 = self.send_binding_request(stun_server_alt)
        
        if response4.mapped_address.port == mapped_addr.port:
            return 'PORT_RESTRICTED_CONE'
        else:
            return 'SYMMETRIC'
```

### 3.3 NAT 穿透策略矩阵

| 节点 A \ 节点 B | Open | Full Cone | Addr Restricted | Port Restricted | Symmetric |
|----------------|------|-----------|-----------------|-----------------|-----------|
| **Open** | Direct | Direct | Direct | Direct | Direct |
| **Full Cone** | Direct | Direct | Direct | Direct | Direct |
| **Addr Restricted** | Direct | Direct | Hole Punch | Hole Punch | Relay |
| **Port Restricted** | Direct | Direct | Hole Punch | Hole Punch | Relay |
| **Symmetric** | Direct | Direct | Relay | Relay | Relay |

### 3.4 UDP 打洞 (Hole Punching)

```python
class HolePunching:
    """UDP 打洞实现"""
    
    def punch_hole(self, peer_public_addr, super_node):
        """
        执行 UDP 打洞
        
        前提条件：双方都已与超级节点建立连接
        """
        # Step 1: 通过超级节点交换公网地址
        my_public_addr = super_node.get_my_mapped_address()
        peer_public_addr = super_node.get_peer_address(peer_id)
        
        # Step 2: 同时向对方发送 UDP 包
        # 这会在本地 NAT 上创建"洞"
        for i in range(5):
            self.socket.sendto(
                b'PUNCH', 
                peer_public_addr
            )
            time.sleep(0.1)
        
        # Step 3: 等待对方的包到达
        # 由于双方同时打洞，包应该能够通过
        self.socket.settimeout(5)
        try:
            data, addr = self.socket.recvfrom(1024)
            if addr == peer_public_addr:
                return True  # 打洞成功
        except socket.timeout:
            return False  # 打洞失败，需要使用中继

    def simultaneous_punch(self, peer_info, coordinator):
        """
        协调双方同时打洞
        """
        # 使用超级节点作为协调者
        coordinator.signal_ready(self.node_id)
        
        # 等待对方也准备好
        coordinator.wait_for_peer(peer_info.node_id, timeout=5)
        
        # 收到信号后立即开始打洞
        start_time = coordinator.get_sync_time()
        
        while time.now() < start_time:
            time.sleep(0.001)
        
        # 同时开始发送
        return self.punch_hole(peer_info.public_addr, coordinator)
```

### 3.5 TURN 中继

当打洞失败时，使用中继：

```python
class TURNRelay:
    """TURN 中继实现"""
    
    def __init__(self, relay_server):
        self.relay_server = relay_server
        self.allocation = None
        
    def allocate(self):
        """在中继服务器上分配端口"""
        request = TURNMessage(type='ALLOCATE')
        response = self.send(request)
        
        if response.success:
            self.allocation = {
                'relay_address': response.relay_address,
                'lifetime': response.lifetime
            }
        return self.allocation
    
    def create_permission(self, peer_address):
        """创建对等方权限"""
        request = TURNMessage(
            type='CREATE_PERMISSION',
            peer_address=peer_address
        )
        return self.send(request)
    
    def send_through_relay(self, data, peer_address):
        """通过中继发送数据"""
        # 封装数据，添加 peer 地址信息
        indication = TURNMessage(
            type='SEND',
            peer_address=peer_address,
            data=data
        )
        self.socket.sendto(
            indication.encode(), 
            self.relay_server
        )
```

### 3.6 ICE (Interactive Connectivity Establishment)

Skype 使用类似 ICE 的连接建立流程：

```python
class ICEAgent:
    """ICE 连接建立代理"""
    
    def gather_candidates(self):
        """收集所有可能的连接候选"""
        candidates = []
        
        # Host candidates (本地地址)
        for iface in self.get_network_interfaces():
            candidates.append(Candidate(
                type='host',
                address=iface.address,
                priority=self.calc_priority('host')
            ))
        
        # Server reflexive candidates (STUN 映射地址)
        for stun_server in self.stun_servers:
            mapped = self.stun_binding(stun_server)
            if mapped:
                candidates.append(Candidate(
                    type='srflx',
                    address=mapped,
                    base=iface.address,
                    priority=self.calc_priority('srflx')
                ))
        
        # Relay candidates (TURN 中继地址)
        for turn_server in self.turn_servers:
            relay = self.turn_allocate(turn_server)
            if relay:
                candidates.append(Candidate(
                    type='relay',
                    address=relay,
                    priority=self.calc_priority('relay')
                ))
        
        return sorted(candidates, key=lambda c: -c.priority)
    
    def connectivity_check(self, local_cand, remote_cand):
        """连通性检查"""
        # 发送 STUN Binding Request
        request = STUNMessage(type='BINDING_REQUEST')
        request.add_attribute('USERNAME', self.make_ice_username())
        request.add_attribute('MESSAGE_INTEGRITY', self.calc_hmac())
        
        try:
            self.socket.sendto(request.encode(), remote_cand.address)
            response = self.socket.recvfrom(1024)
            return self.validate_response(response)
        except socket.timeout:
            return False
    
    def establish_connection(self, remote_candidates):
        """建立连接的完整流程"""
        local_candidates = self.gather_candidates()
        
        # 形成候选对并排序
        pairs = []
        for local in local_candidates:
            for remote in remote_candidates:
                if local.ip_version == remote.ip_version:
                    pairs.append(CandidatePair(local, remote))
        
        pairs.sort(key=lambda p: p.priority, reverse=True)
        
        # 按优先级检查连通性
        for pair in pairs:
            if self.connectivity_check(pair.local, pair.remote):
                return pair  # 找到可用的候选对
        
        return None  # 无法建立连接
```

---

## 4. Overlay 网络与路由机制

### 4.1 分布式哈希表 (DHT) 概念

Skype 使用了类似 Kademlia 的 DHT 结构：

```python
class KademliaNode:
    """Kademlia 风格的 DHT 节点"""
    
    def __init__(self, node_id):
        self.node_id = node_id  # 160-bit 节点 ID
        self.k = 20             # K-bucket 大小
        self.alpha = 3          # 并发查询数
        
        # K-buckets: 按距离分组的节点列表
        # bucket[i] 存储距离在 [2^i, 2^(i+1)) 范围内的节点
        self.buckets = [[] for _ in range(160)]
    
    def xor_distance(self, id1, id2):
        """计算 XOR 距离"""
        return int(id1, 16) ^ int(id2, 16)
    
    def bucket_index(self, other_id):
        """确定节点应该在哪个 bucket"""
        distance = self.xor_distance(self.node_id, other_id)
        if distance == 0:
            return 0
        return distance.bit_length() - 1
    
    def update_bucket(self, node):
        """更新 K-bucket"""
        idx = self.bucket_index(node.id)
        bucket = self.buckets[idx]
        
        if node in bucket:
            # 移到末尾（最近访问）
            bucket.remove(node)
            bucket.append(node)
        elif len(bucket) < self.k:
            bucket.append(node)
        else:
            # bucket 已满，检查最久未访问的节点
            oldest = bucket[0]
            if not oldest.is_alive():
                bucket.remove(oldest)
                bucket.append(node)
    
    def find_closest_nodes(self, target_id, count=20):
        """查找最接近目标的 k 个节点"""
        all_nodes = []
        for bucket in self.buckets:
            all_nodes.extend(bucket)
        
        all_nodes.sort(
            key=lambda n: self.xor_distance(n.id, target_id)
        )
        return all_nodes[:count]
```

### 4.2 节点查找算法

```python
class NodeLookup:
    """迭代式节点查找"""
    
    def lookup(self, target_id):
        """查找最接近目标的节点"""
        # 初始化：从本地路由表获取最近的节点
        closest = self.node.find_closest_nodes(target_id, self.alpha)
        queried = set()
        results = set(closest)
        
        while True:
            # 选择未查询过的最近节点
            to_query = [n for n in closest if n not in queried][:self.alpha]
            
            if not to_query:
                break
            
            # 并行发送查询
            responses = self.parallel_query(to_query, target_id)
            
            for node, new_nodes in responses:
                queried.add(node)
                results.update(new_nodes)
            
            # 更新最近节点列表
            all_nodes = list(results)
            all_nodes.sort(
                key=lambda n: self.xor_distance(n.id, target_id)
            )
            closest = all_nodes[:self.k]
            
            # 收敛检查
            if set(closest) == set(previous_closest):
                break
            previous_closest = closest
        
        return closest
    
    def parallel_query(self, nodes, target_id):
        """并行查询多个节点"""
        import asyncio
        
        async def query_node(node):
            try:
                response = await asyncio.wait_for(
                    node.find_node(target_id),
                    timeout=2.0
                )
                return (node, response)
            except asyncio.TimeoutError:
                return (node, [])
        
        loop = asyncio.get_event_loop()
        tasks = [query_node(n) for n in nodes]
        return loop.run_until_complete(asyncio.gather(*tasks))
```

### 4.3 超级节点间的路由

```
Super Node Overlay Network (超级节点覆盖网络)

       ┌────────────────────────────────────────────────┐
       │                                                │
       │    SN-A ◄──────────────► SN-B                 │
       │      │                     │                   │
       │      │                     │                   │
       │      ▼                     ▼                   │
       │    SN-C ◄──────────────► SN-D                 │
       │      │                     │                   │
       │      └──────────► SN-E ◄───┘                   │
       │                                                │
       └────────────────────────────────────────────────┘

每个超级节点维护:
- 直接连接的其他超级节点列表
- 路由表（指向更多超级节点）
- 连接到该超级节点的普通节点索引
```

### 4.4 消息路由流程

```python
class MessageRouter:
    """消息路由器"""
    
    def route_message(self, message, destination_user):
        """路由消息到目标用户"""
        
        # Step 1: 查找目标用户位置
        target_info = self.lookup_user(destination_user)
        
        if not target_info:
            raise UserOfflineError(destination_user)
        
        # Step 2: 检查是否可以直接连接
        if self.can_direct_connect(target_info):
            return self.send_direct(message, target_info)
        
        # Step 3: 通过超级节点中继
        relay_sn = self.find_best_relay(target_info)
        return self.send_via_relay(message, relay_sn, target_info)
    
    def lookup_user(self, username):
        """在 P2P 网络中查找用户"""
        # 计算用户名的 hash 作为查找键
        user_hash = sha1(username.encode()).hexdigest()
        
        # 使用 DHT 查找存储用户信息的节点
        responsible_nodes = self.dht.lookup(user_hash)
        
        for node in responsible_nodes:
            user_info = node.get_user_info(username)
            if user_info:
                return user_info
        
        return None
```

---

## 5. 用户发现与搜索机制

### 5.1 全局用户搜索

Skype 实现了高效的分布式用户搜索：

```python
class GlobalSearch:
    """全局搜索实现"""
    
    def search(self, query, max_results=100):
        """
        搜索用户
        
        搜索流程:
        1. 在本地超级节点搜索
        2. 扩散到其他超级节点
        3. 聚合结果
        """
        results = []
        visited_sns = set()
        
        # 从当前超级节点开始
        start_sn = self.get_my_super_node()
        
        # BFS 扩散搜索
        queue = [start_sn]
        ttl = 7  # 搜索深度限制
        
        while queue and len(results) < max_results and ttl > 0:
            current_sn = queue.pop(0)
            
            if current_sn.id in visited_sns:
                continue
            visited_sns.add(current_sn.id)
            
            # 在当前超级节点搜索
            local_results = current_sn.local_search(query)
            results.extend(local_results)
            
            # 添加相邻超级节点到队列
            for neighbor in current_sn.get_neighbors():
                if neighbor.id not in visited_sns:
                    queue.append(neighbor)
            
            ttl -= 1
        
        # 去重和排序
        return self.deduplicate_and_rank(results)[:max_results]
    
    def local_search(self, query):
        """超级节点本地搜索"""
        results = []
        query_lower = query.lower()
        
        for node_id, node_info in self.connected_nodes.items():
            username = node_info.get('username', '')
            display_name = node_info.get('display_name', '')
            
            # 模糊匹配
            if (query_lower in username.lower() or 
                query_lower in display_name.lower()):
                results.append({
                    'username': username,
                    'display_name': display_name,
                    'online': node_info.get('online', False),
                    'relevance': self.calc_relevance(query, node_info)
                })
        
        return results
```

### 5.2 在线状态同步

```python
class PresenceService:
    """在线状态服务"""
    
    def __init__(self):
        self.status = 'offline'
        self.subscribers = []  # 订阅我状态的用户
        self.subscriptions = []  # 我订阅的用户状态
    
    def set_status(self, new_status):
        """设置状态并通知订阅者"""
        self.status = new_status
        
        # 通知所有订阅者
        for subscriber in self.subscribers:
            self.notify_status_change(subscriber, new_status)
        
        # 更新超级节点上的状态
        self.super_node.update_presence(self.user_id, new_status)
    
    def subscribe_to_user(self, target_user):
        """订阅用户状态"""
        # 查找用户的超级节点
        target_sn = self.lookup_user_super_node(target_user)
        
        # 发送订阅请求
        target_sn.add_presence_subscription(
            subscriber=self.user_id,
            target=target_user
        )
        
        self.subscriptions.append(target_user)
    
    def handle_status_update(self, user, new_status):
        """处理状态更新通知"""
        # 更新本地缓存
        self.buddy_status_cache[user] = {
            'status': new_status,
            'timestamp': time.now()
        }
        
        # 触发 UI 更新
        self.emit_event('presence_changed', user, new_status)
```

### 5.3 好友关系存储

```python
class BuddyListManager:
    """好友列表管理"""
    
    def __init__(self):
        # 本地存储
        self.local_buddies = []
        
        # 分布式存储（冗余备份）
        self.dht_key = sha1(f"buddylist:{self.user_id}".encode())
    
    def sync_buddy_list(self):
        """同步好友列表"""
        # 从 DHT 获取
        stored_list = self.dht.get(self.dht_key)
        
        if stored_list:
            # 解密并验证
            decrypted = self.decrypt(stored_list)
            if self.verify_signature(decrypted):
                self.local_buddies = decrypted['buddies']
    
    def add_buddy(self, buddy_username):
        """添加好友"""
        self.local_buddies.append(buddy_username)
        
        # 加密并存储到 DHT
        encrypted = self.encrypt({
            'buddies': self.local_buddies,
            'timestamp': time.now()
        })
        
        signed = self.sign(encrypted)
        self.dht.store(self.dht_key, signed)
        
        # 订阅好友状态
        self.presence_service.subscribe_to_user(buddy_username)
```

---

## 6. 安全与加密机制

### 6.1 加密架构概述

```
┌─────────────────────────────────────────────────────────────────┐
│                      Skype 加密架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  应用层加密                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │   Voice     │  │   Video     │  │   Text      │     │   │
│  │  │   AES-256   │  │   AES-256   │  │   AES-256   │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  密钥交换层                              │   │
│  │           RSA-2048 + Diffie-Hellman                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  身份认证层                              │   │
│  │              RSA 证书 + 用户密码                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 密钥生成与管理

```python
class CryptoManager:
    """加密管理器"""
    
    def __init__(self):
        self.rsa_key_pair = None
        self.session_keys = {}
    
    def generate_identity_keys(self):
        """生成身份密钥对"""
        from cryptography.hazmat.primitives.asymmetric import rsa
        
        # RSA-2048 密钥对
        self.rsa_key_pair = rsa.generate_private_key(
            public_exponent=65537,
            key_size=2048
        )
        
        return self.rsa_key_pair.public_key()
    
    def derive_session_key(self, peer_public_key):
        """
        使用 ECDH 派生会话密钥
        """
        from cryptography.hazmat.primitives.kdf.hkdf import HKDF
        from cryptography.hazmat.primitives import hashes
        from cryptography.hazmat.primitives.asymmetric import ec
        
        # 生成临时 ECDH 密钥对
        private_key = ec.generate_private_key(ec.SECP256R1())
        
        # 执行密钥交换
        shared_secret = private_key.exchange(
            ec.ECDH(), 
            peer_public_key
        )
        
        # 使用 HKDF 派生 AES 密钥
        session_key = HKDF(
            algorithm=hashes.SHA256(),
            length=32,
            salt=None,
            info=b'skype-session-key'
        ).derive(shared_secret)
        
        return session_key
```

### 6.3 端到端加密流程

```python
class E2EEncryption:
    """端到端加密实现"""
    
    def encrypt_message(self, plaintext, session_key):
        """加密消息"""
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
        import os
        
        # 生成随机 IV
        iv = os.urandom(16)
        
        # AES-256-CTR 加密
        cipher = Cipher(
            algorithms.AES(session_key),
            modes.CTR(iv)
        )
        encryptor = cipher.encryptor()
        ciphertext = encryptor.update(plaintext) + encryptor.finalize()
        
        # 添加 HMAC 完整性保护
        mac = self.compute_hmac(session_key, iv + ciphertext)
        
        return iv + ciphertext + mac
    
    def decrypt_message(self, encrypted_data, session_key):
        """解密消息"""
        iv = encrypted_data[:16]
        mac = encrypted_data[-32:]
        ciphertext = encrypted_data[16:-32]
        
        # 验证 HMAC
        expected_mac = self.compute_hmac(session_key, iv + ciphertext)
        if not self.constant_time_compare(mac, expected_mac):
            raise IntegrityError("MAC verification failed")
        
        # 解密
        cipher = Cipher(
            algorithms.AES(session_key),
            modes.CTR(iv)
        )
        decryptor = cipher.decryptor()
        plaintext = decryptor.update(ciphertext) + decryptor.finalize()
        
        return plaintext
```

### 6.4 身份验证

```python
class AuthenticationService:
    """身份验证服务"""
    
    def login(self, username, password):
        """
        登录流程:
        1. 连接到登录服务器 (少数中心化组件之一)
        2. 使用用户名/密码认证
        3. 获取签名证书
        4. 加入 P2P 网络
        """
        # 派生密钥
        password_hash = self.hash_password(username, password)
        
        # 发送认证请求
        auth_request = {
            'username': username,
            'password_hash': password_hash,
            'public_key': self.crypto.get_public_key(),
            'timestamp': time.now()
        }
        
        # 登录服务器验证并签发证书
        response = self.login_server.authenticate(auth_request)
        
        if response.success:
            self.identity_certificate = response.certificate
            self.join_p2p_network()
            return True
        
        return False
    
    def verify_peer_identity(self, peer_certificate):
        """验证对等方身份"""
        # 验证证书签名
        if not self.verify_certificate_signature(peer_certificate):
            return False
        
        # 检查证书是否过期
        if peer_certificate.expires_at < time.now():
            return False
        
        # 检查证书撤销列表 (CRL)
        if self.is_certificate_revoked(peer_certificate):
            return False
        
        return True
```

---

## 7. 实时通信核心流程

### 7.1 呼叫建立流程

```
┌─────────┐         ┌─────────┐         ┌─────────┐         ┌─────────┐
│ Caller  │         │Caller's │         │Callee's │         │ Callee  │
│         │         │   SN    │         │   SN    │         │         │
└────┬────┘         └────┬────┘         └────┬────┘         └────┬────┘
     │                   │                   │                   │
     │  1. INVITE        │                   │                   │
     │──────────────────>│                   │                   │
     │                   │  2. Route INVITE  │                   │
     │                   │──────────────────>│                   │
     │                   │                   │  3. Forward       │
     │                   │                   │──────────────────>│
     │                   │                   │                   │
     │                   │                   │  4. RINGING       │
     │                   │                   │<──────────────────│
     │                   │  5. RINGING       │                   │
     │                   │<──────────────────│                   │
     │  6. RINGING       │                   │                   │
     │<──────────────────│                   │                   │
     │                   │                   │                   │
     │                   │                   │  7. ACCEPT        │
     │                   │                   │<──────────────────│
     │                   │  8. ACCEPT + ICE  │                   │
     │                   │<──────────────────│                   │
     │  9. ACCEPT + ICE  │                   │                   │
     │<──────────────────│                   │                   │
     │                   │                   │                   │
     │                  10. ICE Connectivity Check              │
     │<═══════════════════════════════════════════════════════>│
     │                   │                   │                   │
     │                  11. Direct Media (if possible)          │
     │<═════════════════════════════════════════════════════════>│
     │                   │                   │                   │
```

### 7.2 呼叫建立代码实现

```python
class CallManager:
    """呼叫管理器"""
    
    def __init__(self):
        self.active_calls = {}
        self.ice_agent = ICEAgent()
        self.media_engine = MediaEngine()
    
    async def initiate_call(self, callee_username):
        """发起呼叫"""
        # 创建呼叫会话
        call_id = self.generate_call_id()
        call_session = CallSession(call_id)
        
        # 查找被叫方
        callee_info = await self.lookup_user(callee_username)
        if not callee_info or not callee_info.online:
            raise UserOfflineError(callee_username)
        
        # 收集 ICE candidates
        local_candidates = await self.ice_agent.gather_candidates()
        
        # 生成会话密钥
        session_key = self.crypto.generate_session_key()
        
        # 构造 INVITE 消息
        invite = {
            'type': 'INVITE',
            'call_id': call_id,
            'caller': self.username,
            'callee': callee_username,
            'ice_candidates': local_candidates,
            'session_key_encrypted': self.encrypt_session_key(
                session_key, 
                callee_info.public_key
            ),
            'codecs': self.media_engine.supported_codecs
        }
        
        # 发送 INVITE
        response = await self.send_signaling(invite, callee_info)
        
        if response.type == 'ACCEPT':
            # ICE 连接建立
            remote_candidates = response.ice_candidates
            selected_pair = await self.ice_agent.establish_connection(
                remote_candidates
            )
            
            # 开始媒体流
            call_session.media_path = selected_pair
            call_session.session_key = session_key
            await self.media_engine.start_stream(call_session)
            
            self.active_calls[call_id] = call_session
            return call_session
        
        elif response.type == 'REJECT':
            raise CallRejectedError()
        
        elif response.type == 'BUSY':
            raise UserBusyError()
    
    async def handle_incoming_call(self, invite):
        """处理来电"""
        call_id = invite['call_id']
        caller = invite['caller']
        
        # 验证呼叫方身份
        if not await self.verify_caller(caller):
            return self.create_response('REJECT', call_id)
        
        # 通知 UI 有来电
        user_response = await self.ui.show_incoming_call(caller)
        
        if user_response == 'accept':
            # 收集本地 ICE candidates
            local_candidates = await self.ice_agent.gather_candidates()
            
            # 解密会话密钥
            session_key = self.crypto.decrypt_session_key(
                invite['session_key_encrypted']
            )
            
            # 发送 ACCEPT
            accept = {
                'type': 'ACCEPT',
                'call_id': call_id,
                'ice_candidates': local_candidates,
                'selected_codec': self.negotiate_codec(invite['codecs'])
            }
            
            return accept
        else:
            return {'type': 'REJECT', 'call_id': call_id}
```

### 7.3 媒体引擎

```python
class MediaEngine:
    """媒体引擎"""
    
    def __init__(self):
        self.audio_codec = None
        self.video_codec = None
        self.jitter_buffer = JitterBuffer()
        
    async def start_stream(self, call_session):
        """开始媒体流"""
        # 初始化编解码器
        self.audio_codec = self.create_codec(call_session.audio_codec)
        if call_session.video_enabled:
            self.video_codec = self.create_codec(call_session.video_codec)
        
        # 启动发送和接收循环
        asyncio.create_task(self.send_loop(call_session))
        asyncio.create_task(self.receive_loop(call_session))
    
    async def send_loop(self, call_session):
        """发送循环"""
        while call_session.active:
            # 从麦克风捕获音频
            raw_audio = await self.capture_audio()
            
            # 编码
            encoded = self.audio_codec.encode(raw_audio)
            
            # 加密
            encrypted = self.encrypt(encoded, call_session.session_key)
            
            # 打包 RTP
            rtp_packet = self.create_rtp_packet(
                encrypted,
                call_session.sequence_number,
                call_session.timestamp
            )
            
            # 发送
            await self.send_packet(rtp_packet, call_session.media_path)
            
            call_session.sequence_number += 1
            call_session.timestamp += self.audio_codec.samples_per_frame
    
    async def receive_loop(self, call_session):
        """接收循环"""
        while call_session.active:
            # 接收数据包
            packet = await self.receive_packet(call_session.media_path)
            
            # 解析 RTP
            rtp = self.parse_rtp_packet(packet)
            
            # 解密
            decrypted = self.decrypt(rtp.payload, call_session.session_key)
            
            # 添加到抖动缓冲区
            self.jitter_buffer.add(rtp.sequence, rtp.timestamp, decrypted)
            
            # 从缓冲区获取并解码
            frame = self.jitter_buffer.get_next()
            if frame:
                decoded = self.audio_codec.decode(frame)
                await self.play_audio(decoded)


class JitterBuffer:
    """抖动缓冲区"""
    
    def __init__(self, target_delay_ms=60):
        self.buffer = {}
        self.target_delay = target_delay_ms
        self.next_sequence = 0
        
    def add(self, sequence, timestamp, data):
        """添加数据包"""
        self.buffer[sequence] = {
            'timestamp': timestamp,
            'data': data,
            'arrival_time': time.now()
        }
    
    def get_next(self):
        """获取下一帧"""
        if self.next_sequence in self.buffer:
            frame = self.buffer.pop(self.next_sequence)
            self.next_sequence += 1
            return frame['data']
        else:
            # 丢包处理
            self.next_sequence += 1
            return None  # 或执行丢包隐藏 (PLC)
```

### 7.4 自适应码率控制

```python
class AdaptiveBitrateController:
    """自适应码率控制"""
    
    def __init__(self):
        self.current_bitrate = 64000  # 64 kbps 起始
        self.min_bitrate = 8000       # 8 kbps 最低
        self.max_bitrate = 256000     # 256 kbps 最高
        
        self.rtt_history = []
        self.loss_history = []
        
    def update_statistics(self, rtt_ms, packet_loss_rate):
        """更新网络统计"""
        self.rtt_history.append(rtt_ms)
        self.loss_history.append(packet_loss_rate)
        
        # 保持最近 100 个样本
        self.rtt_history = self.rtt_history[-100:]
        self.loss_history = self.loss_history[-100:]
        
        # 调整码率
        self.adjust_bitrate()
    
    def adjust_bitrate(self):
        """调整码率"""
        avg_rtt = sum(self.rtt_history) / len(self.rtt_history)
        avg_loss = sum(self.loss_history) / len(self.loss_history)
        
        if avg_loss > 0.05:  # 丢包率 > 5%
            # 降低码率
            self.current_bitrate = max(
                self.min_bitrate,
                int(self.current_bitrate * 0.8)
            )
        elif avg_loss < 0.01 and avg_rtt < 150:
            # 网络良好，尝试提高码率
            self.current_bitrate = min(
                self.max_bitrate,
                int(self.current_bitrate * 1.1)
            )
        
        return self.current_bitrate
```

---

## 8. 工程实践要点

### 8.1 Bootstrap 机制

```python
class BootstrapManager:
    """引导管理器"""
    
    def __init__(self):
        # 硬编码的 bootstrap 服务器
        self.bootstrap_servers = [
            'bootstrap1.skype.com:443',
            'bootstrap2.skype.com:443',
        ]
        
        # 本地缓存的超级节点列表
        self.host_cache_file = 'host_cache.dat'
        self.host_cache = self.load_host_cache()
    
    def load_host_cache(self):
        """加载本地缓存的超级节点"""
        try:
            with open(self.host_cache_file, 'rb') as f:
                cached = pickle.load(f)
                # 过滤过期条目
                return [
                    entry for entry in cached
                    if entry['last_seen'] > time.now() - timedelta(days=7)
                ]
        except FileNotFoundError:
            return []
    
    def save_host_cache(self):
        """保存超级节点缓存"""
        with open(self.host_cache_file, 'wb') as f:
            pickle.dump(self.host_cache, f)
    
    async def bootstrap(self):
        """启动引导流程"""
        # 1. 尝试本地缓存的超级节点
        for sn in self.host_cache:
            if await self.try_connect(sn['address']):
                return sn
        
        # 2. 本地缓存失败，使用 bootstrap 服务器
        for server in self.bootstrap_servers:
            try:
                sn_list = await self.fetch_super_nodes(server)
                for sn in sn_list:
                    if await self.try_connect(sn):
                        # 更新缓存
                        self.host_cache.append({
                            'address': sn,
                            'last_seen': time.now()
                        })
                        self.save_host_cache()
                        return sn
            except ConnectionError:
                continue
        
        raise BootstrapError("无法连接到 Skype 网络")
    
    async def try_connect(self, address):
        """尝试连接超级节点"""
        try:
            socket = await asyncio.open_connection(
                address.host, 
                address.port,
                ssl=True
            )
            # 发送握手
            await self.send_handshake(socket)
            return True
        except Exception:
            return False
```

### 8.2 连接保活与故障检测

```python
class ConnectionManager:
    """连接管理器"""
    
    def __init__(self):
        self.super_node_connection = None
        self.heartbeat_interval = 30  # 30 秒
        self.timeout_threshold = 90   # 90 秒无响应认为断开
        
    async def heartbeat_loop(self):
        """心跳循环"""
        while True:
            try:
                # 发送心跳
                response = await asyncio.wait_for(
                    self.send_heartbeat(),
                    timeout=10
                )
                
                if response:
                    self.last_response_time = time.now()
                    
            except asyncio.TimeoutError:
                # 检查是否超时
                if time.now() - self.last_response_time > self.timeout_threshold:
                    await self.handle_disconnection()
            
            await asyncio.sleep(self.heartbeat_interval)
    
    async def handle_disconnection(self):
        """处理断开连接"""
        logger.warning("超级节点连接断开，尝试重连")
        
        # 从缓存中选择其他超级节点
        for backup_sn in self.host_cache:
            if backup_sn != self.super_node_connection:
                try:
                    await self.connect_to_super_node(backup_sn)
                    logger.info(f"成功切换到超级节点: {backup_sn}")
                    return
                except ConnectionError:
                    continue
        
        # 所有备份都失败，重新 bootstrap
        await self.bootstrap()


class FailureDetector:
    """故障检测器 (Phi Accrual Failure Detector)"""
    
    def __init__(self):
        self.heartbeat_history = []
        self.phi_threshold = 8  # 故障判定阈值
        
    def record_heartbeat(self):
        """记录心跳到达"""
        now = time.now()
        if self.heartbeat_history:
            interval = now - self.heartbeat_history[-1]
            self.intervals.append(interval)
        self.heartbeat_history.append(now)
        
    def calculate_phi(self):
        """计算 Phi 值"""
        if len(self.intervals) < 2:
            return 0
            
        # 计算心跳间隔的均值和标准差
        mean = statistics.mean(self.intervals)
        std = statistics.stdev(self.intervals)
        
        # 计算自上次心跳以来的时间
        time_since_last = time.now() - self.heartbeat_history[-1]
        
        # 计算累积分布函数
        prob = self.normal_cdf(time_since_last, mean, std)
        
        # Phi = -log10(1 - prob)
        if prob >= 1:
            return float('inf')
        return -math.log10(1 - prob)
    
    def is_alive(self):
        """判断节点是否存活"""
        return self.calculate_phi() < self.phi_threshold
```

### 8.3 网络质量监控

```python
class NetworkQualityMonitor:
    """网络质量监控"""
    
    def __init__(self):
        self.metrics = {
            'rtt': [],
            'jitter': [],
            'packet_loss': [],
            'bandwidth': []
        }
        
    def measure_rtt(self, peer):
        """测量往返时延"""
        start = time.now()
        response = await self.send_ping(peer)
        end = time.now()
        
        rtt = (end - start).total_seconds() * 1000
        self.metrics['rtt'].append(rtt)
        return rtt
    
    def calculate_jitter(self):
        """计算抖动"""
        if len(self.metrics['rtt']) < 2:
            return 0
            
        # 抖动 = RTT 变化的平均值
        diffs = []
        for i in range(1, len(self.metrics['rtt'])):
            diffs.append(abs(
                self.metrics['rtt'][i] - self.metrics['rtt'][i-1]
            ))
        
        return statistics.mean(diffs)
    
    def estimate_bandwidth(self):
        """带宽估计 (使用 packet pair 技术)"""
        # 发送两个背靠背的包
        packet1_time = time.now()
        await self.send_probe_packet(size=1500)
        
        packet2_time = time.now()
        await self.send_probe_packet(size=1500)
        
        # 接收端测量到达时间差
        arrival_gap = self.receive_probe_gap()
        
        # 带宽 = 包大小 / 到达间隔
        bandwidth = 1500 * 8 / arrival_gap  # bits per second
        
        self.metrics['bandwidth'].append(bandwidth)
        return bandwidth
    
    def get_quality_score(self):
        """计算综合质量评分 (1-5)"""
        avg_rtt = statistics.mean(self.metrics['rtt'][-20:])
        avg_jitter = self.calculate_jitter()
        avg_loss = statistics.mean(self.metrics['packet_loss'][-20:])
        
        score = 5.0
        
        # RTT 惩罚
        if avg_rtt > 300:
            score -= 2.0
        elif avg_rtt > 150:
            score -= 1.0
        elif avg_rtt > 50:
            score -= 0.5
        
        # 抖动惩罚
        if avg_jitter > 50:
            score -= 1.5
        elif avg_jitter > 20:
            score -= 0.5
        
        # 丢包惩罚
        if avg_loss > 0.1:
            score -= 2.0
        elif avg_loss > 0.05:
            score -= 1.0
        elif avg_loss > 0.01:
            score -= 0.5
        
        return max(1.0, score)
```

### 8.4 日志与调试

```python
class DebugLogger:
    """调试日志系统"""
    
    def __init__(self):
        self.log_level = logging.INFO
        self.handlers = []
        
    def log_network_event(self, event_type, **kwargs):
        """记录网络事件"""
        event = {
            'timestamp': time.now().isoformat(),
            'type': event_type,
            'data': kwargs
        }
        
        if event_type in ['connection_failed', 'nat_traversal_failed']:
            self.log(logging.WARNING, event)
        else:
            self.log(logging.DEBUG, event)
    
    def log_call_quality(self, call_id, metrics):
        """记录通话质量"""
        self.log(logging.INFO, {
            'call_id': call_id,
            'rtt': metrics.rtt,
            'jitter': metrics.jitter,
            'packet_loss': metrics.packet_loss,
            'mos_score': metrics.mos_score
        })
    
    def generate_diagnostics(self):
        """生成诊断报告"""
        return {
            'client_version': VERSION,
            'nat_type': self.get_nat_type(),
            'super_node': self.current_super_node,
            'uptime': self.get_uptime(),
            'recent_calls': self.get_recent_call_stats(),
            'network_metrics': self.get_network_metrics()
        }
```

---

## 9. 架构演进与经验教训

### 9.1 P2P 架构的优势

| 优势 | 描述 |
|------|------|
| **成本效益** | 利用用户设备资源，大幅降低服务器成本 |
| **可扩展性** | 用户增长自动带来更多网络容量 |
| **抗故障性** | 无单点故障，网络自愈能力强 |
| **低延迟** | 点对点直连，减少中转跳数 |

### 9.2 P2P 架构的挑战

| 挑战 | Skype 的解决方案 |
|------|------------------|
| **NAT 穿透** | 多种穿透技术 + 中继回退 |
| **用户发现** | 分布式搜索 + 超级节点索引 |
| **安全性** | 端到端加密 + 证书认证 |
| **服务质量** | 自适应码率 + 质量监控 |
| **管理复杂性** | 难以统一管控和合规 |

### 9.3 从 P2P 到混合架构的演进

2011 年 Microsoft 收购 Skype 后，架构逐步向云端迁移：

```
P2P 时代 (2003-2011)                    云端时代 (2011-现在)
┌────────────────────┐                  ┌────────────────────┐
│   Pure P2P         │                  │  Cloud-Centric     │
│                    │                  │                    │
│  ┌──┐ ◄──► ┌──┐   │                  │    ┌─────────┐    │
│  │SN│      │SN│   │     ────►        │    │ Azure   │    │
│  └──┘      └──┘   │                  │    │ Servers │    │
│    │          │    │                  │    └────┬────┘    │
│  ┌──┐      ┌──┐   │                  │         │         │
│  │ON│      │ON│   │                  │    ┌────┴────┐    │
│  └──┘      └──┘   │                  │  ┌─┴─┐    ┌─┴─┐  │
│                    │                  │  │ C │    │ C │  │
└────────────────────┘                  │  └───┘    └───┘  │
                                        └────────────────────┘
```

**迁移原因**：
- 企业客户需要更好的管控能力
- 移动设备电量/带宽限制不适合作为超级节点
- 法规要求（合法监听、数据保留）
- 统一服务质量保障

### 9.4 关键经验教训

1. **混合架构的平衡**
   - 纯 P2P 在极端去中心化和实用性之间需要权衡
   - 少量中心化服务（登录、bootstrap）可大幅简化系统

2. **NAT 穿透是核心竞争力**
   - Skype 的 NAT 穿透成功率高达 90%+
   - 需要持续维护和更新穿透策略

3. **超级节点选举要谨慎**
   - 选举算法影响网络整体性能
   - 需要防止恶意节点成为超级节点

4. **客户端质量监控不可少**
   - 实时监控通话质量
   - 快速响应网络变化

5. **安全性必须端到端**
   - 不信任任何中间节点
   - 密钥交换和身份验证是基础

---

## 10. 参考文献

1. Baset, S. A., & Schulzrinne, H. G. (2006). **An Analysis of the Skype Peer-to-Peer Internet Telephony Protocol**. *IEEE INFOCOM*.

2. Guha, S., Daswani, N., & Jain, R. (2006). **An Experimental Study of the Skype Peer-to-Peer VoIP System**. *IPTPS*.

3. Rosenberg, J., Mahy, R., Matthews, P., & Wing, D. (2008). **Session Traversal Utilities for NAT (STUN)**. *RFC 5389*.

4. Maymounkov, P., & Mazières, D. (2002). **Kademlia: A Peer-to-Peer Information System Based on the XOR Metric**. *IPTPS*.

5. Rosenberg, J. (2010). **Interactive Connectivity Establishment (ICE)**. *RFC 5245*.

6. Matthews, P., Rosenberg, J., & Mahy, R. (2010). **Traversal Using Relays around NAT (TURN)**. *RFC 5766*.

---

## 附录 A: 术语表

| 术语 | 定义 |
|------|------|
| **Overlay Network** | 构建在底层网络之上的逻辑网络 |
| **NAT** | 网络地址转换，将私有 IP 映射到公网 IP |
| **STUN** | Session Traversal Utilities for NAT，NAT 穿透协议 |
| **TURN** | Traversal Using Relays around NAT，中继穿透协议 |
| **ICE** | Interactive Connectivity Establishment，连接建立框架 |
| **DHT** | Distributed Hash Table，分布式哈希表 |
| **Super Node** | 具有公网 IP 和足够资源的节点，承担路由职责 |
| **Hole Punching** | UDP 打洞，穿透 NAT 建立直接连接的技术 |

---

## 附录 B: 快速参考卡片

### 核心组件

```
┌─────────────────────────────────────────────────────────────────┐
│                    Skype P2P 架构速查                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  节点类型:                                                       │
│    • 普通节点 (ON) - 基础客户端                                  │
│    • 超级节点 (SN) - 路由/索引                                   │
│    • 中继节点 - NAT 穿透失败时使用                               │
│                                                                 │
│  NAT 穿透优先级:                                                 │
│    1. 直接连接 (公网 IP)                                         │
│    2. UDP 打洞                                                   │
│    3. TURN 中继                                                  │
│                                                                 │
│  加密:                                                          │
│    • 身份: RSA-2048                                             │
│    • 密钥交换: ECDH                                              │
│    • 媒体: AES-256-CTR                                          │
│                                                                 │
│  呼叫流程:                                                       │
│    INVITE → RINGING → ACCEPT → ICE → 媒体流                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

*文档版本: 1.0*  
*最后更新: 2026-01-14*
