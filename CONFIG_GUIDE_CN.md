# Easytier 配置文件 (`easytier.toml`) 完全、详尽解析报告

本报告旨在为您提供 `easytier-core` 每一个配置项的详尽说明，包括其作用、数据格式、可选值和配置示例，希望能帮助您完全掌握 Easytier 的配置。

---

## **第一部分：核心配置项 (非 `[flags]` 部分)**

### **1. 实例设置 (Instance Settings)** - 定义节点自身属性

*   **`instance_name`** (字符串, 可选, 默认: `"default"`)
    *   为实例设置一个易于识别的名称，尤其在单机运行多实例时用于区分。
    *   **示例**: `instance_name = "work-network"`

*   **`instance_id`** (UUID字符串, 可选)
    *   实例的唯一标识符。强烈建议让程序自动生成和管理。
    *   **示例**: `instance_id = "87ede5a2-9c3d-492d-9bbe-989b9d07e742"`

*   **`hostname`** (字符串, 可选)
    *   在虚拟网络中显示的自定义主机名。若不设置，则使用系统主机名。
    *   **示例**: `hostname = "Jules-Dev-Laptop"`

*   **`netns`** (字符串, 可选, 仅Linux)
    *   将实例隔离在指定的网络命名空间中，用于高级网络隔离。
    *   **示例**: `netns = "isolated_vpn_net"`

### **2. 网络身份 (`[network_identity]`)** - 定义所属虚拟网络

*   **`network_name`** (字符串, **必选**)
    *   **核心配置**。虚拟网络的名称，只有名称完全相同的节点才能加入同一个网络。
    *   **示例**: `network_name = "my_secure_iot_network"`

*   **`network_secret`** (字符串, 可选)
    *   **核心配置**。虚拟网络的密码，用于加密和认证。若留空则网络不加密。
    *   **示例**: `network_secret = "R@ndom_P@$$w0rd_f0r_S3cur1ty"`

### **3. IP与网络接口 (IP & Network Interface)** - 配置虚拟IP和监听

*   **`ipv4`** (CIDR字符串, 推荐必选)
    *   **核心配置**。为节点在虚拟网络中设置静态IPv4地址，必须包含子网掩码。
    *   **示例**: `ipv4 = "10.144.144.10/24"`

*   **`ipv6`** (IPv6 CIDR字符串, 可选)
    *   为节点在虚拟网络中设置静态IPv6地址。
    *   **示例**: `ipv6 = "fd00:abcd::1/64"`

*   **`dhcp`** (布尔值, 可选, 默认: `false`)
    *   设置为 `true` 可尝试从网络中的DHCP服务器动态获取IP。
    *   **示例**: `dhcp = true`

*   **`listeners`** (URL数组, 推荐必选)
    *   **核心配置**。设置节点监听入站连接的地址和端口。
    *   **示例**: `listeners = [ "tcp://0.0.0.0:11010", "udp://0.0.0.0:11010" ]`

*   **`mapped_listeners`** (URL数组, 可选)
    *   当节点位于NAT后，手动指定公网可访问的地址和端口。
    *   **示例**: `mapped_listeners = [ "tcp://your_public_ip:22022" ]`

*   **`stun_servers`** (字符串数组, 可选)
    *   用于NAT穿透的STUN服务器列表，是实现P2P直连的关键。若不设置则使用内置列表。
    *   **示例**: `stun_servers = [ "stun.l.google.com:19302", "stun.qq.com:3478" ]`

*   **`stun_servers_v6`** (字符串数组, 可选)
    *   专门为IPv6网络指定的STUN服务器列表。
    *   **示例**: `stun_servers_v6 = [ "stun.6.google.com:19302" ]`

### **4. 对等点、路由和转发 (Peers, Routing & Forwarding)** - 配置网络拓扑和数据流

*   **`[[peer]]`** (对象数组, **必选**)
    *   **核心配置**。定义“引导节点”的地址，新节点通过连接它们来发现网络中的其他成员。
    *   **示例**:
        ```toml
        [[peer]]
        uri = "tcp://public.server.one:11010"
        ```

*   **`routes`** (CIDR字符串数组, 可选)
    *   向虚拟网络宣告本节点可路由到的物理局域网，实现远程访问内网。
    *   **示例**: `routes = [ "192.168.1.0/24", "10.10.0.0/16" ]`

*   **`[[proxy_network]]`** (对象数组, 可选)
    *   与 `routes` 类似，但提供更精细的控制，同样宣告本节点可以代理访问某个物理网络。
    *   **示例**:
        ```toml
        [[proxy_network]]
        cidr = "192.168.99.0/24"
        ```

*   **`exit_nodes`** (IP地址数组, 可选)
    *   将指定的节点（需开启`enable_exit_node`）设为出口网关，本机所有公网流量将通过它转发。
    *   **示例**: `exit_nodes = ["10.144.144.1"]`

*   **`[[port_forward]]`** (对象数组, 可选)
    *   创建端口转发规则，将本机端口流量转发至虚拟网络中的任一设备端口。
    *   **示例**:
        ```toml
        [[port_forward]]
        bind_addr = "0.0.0.0:8080"
        dst_addr = "10.144.144.20:80"
        proto = "tcp"
        ```

*   **`socks5_proxy`** (URL, 可选)
    *   在本机启动一个 SOCKS5 代理服务。当其他应用程序通过此代理访问网络时，`easytier-core` 会拦截这些请求，并使用它在虚拟网络中的身份（即虚拟IP）来发起这些连接。最终流量的目的地仍是应用程序请求的目标地址，但流量的源头在网络上看起来是节点的虚拟IP。
    *   **示例**: `socks5_proxy = "tcp://127.0.0.1:1080"`

*   **`[vpn_portal_config]`** (对象, 可选)
    *   启用WireGuard接入功能，允许标准WireGuard客户端连入Easytier网络。
    *   **示例**:
        ```toml
        [vpn_portal_config]
        client_cidr = "10.144.200.0/24"
        wireguard_listen = "0.0.0.0:51820"
        ```

### **5. 安全 (Security)** - 配置访问控制

*   **`[acl]`** (对象, 可选)
    *   定义详细的访问控制列表（ACL）规则，用于精细的流量过滤。
    *   **示例**:
        ```toml
        [acl]
        default_policy = "deny"
        rules = [
          { policy = "allow", dst_port = "80,443", proto = "tcp" }
        ]
        ```

*   **`tcp_whitelist` / `udp_whitelist`** (字符串数组, 可选)
    *   简化的防火墙，定义允许入站访问的TCP/UDP端口白名单（支持范围）。
    *   **示例**: `tcp_whitelist = ["80", "443", "8000-9000"]`

### **6. 日志 (Logging)** - 配置日志输出

*   **`[file_logger]` / `[console_logger]`** (对象, 可选)
    *   分别配置日志到文件和控制台的行为。
    *   **示例**:
        ```toml
        [file_logger]
        level = "info"
        file = "easytier.log"
        dir = "/var/log/easytier"

        [console_logger]
        level = "warn"
        ```

---

## **第二部分：`[flags]` 配置项完全详解**

`[flags]` 部分是用于精细化调整网络行为的开关集合。

*   **`default_protocol`** (字符串, 可选, 默认: `"tcp"`)
    *   节点间连接首选协议，`"tcp"`或`"udp"`。
*   **`disable_p2p`** (布尔值, 可选, 默认: `false`)
    *   `true`则禁用P2P直连，所有流量走中继。
*   **`p2p_only`** (布尔值, 可选, 默认: `false`)
    *   `true`则只允许P2P连接，禁止中继。
*   **`disable_udp_hole_punching`** (布尔值, 可选, 默认: `false`)
    *   `true`则禁用UDP打洞，可能导致P2P失败。
*   **`disable_sym_hole_punching`** (布尔值, 可选, 默认: `false`)
    *   `true`则禁用针对对称NAT的特殊打洞技术。
*   **`latency_first`** (布尔值, 可选, 默认: `false`)
    *   `true`则路由选择优先考虑最低延迟，而非最少跳数。
*   **`relay_all_peer_rpc`** (布尔值, 可选, 默认: `false`)
    *   `true`则将收到的RPC请求转发给所有对等节点，用于调试。
*   **`private_mode`** (布尔值, 可选, 默认: `false`)
    *   `true`则不向公共服务器报告节点信息。
*   **`dev_name`** (字符串, 可选, 默认: `""`)
    *   指定虚拟网卡名称。
*   **`enable_ipv6`** (布尔值, 可选, 默认: `true`)
    *   `false`则禁用IPv6。
*   **`mtu`** (整数, 可选, 默认: `1380`)
    *   设置虚拟网卡的最大传输单元。
*   **`no_tun`** (布尔值, 可选, 默认: `false`)
    *   `true`则不创建虚拟网卡，以纯代理/中继模式运行。
*   **`use_smoltcp`** (布尔值, 可选, 默认: `false`)
    *   `true`则使用用户态TCP/IP协议栈。
*   **`bind_device`** (布尔值, 可选, 默认: `true`)
    *   `true`则将网络套接字绑定到指定的物理网卡。
*   **`proxy_forward_by_system`** (布尔值, 可选, 默认: `false`)
    *   `true`则使用操作系统路由和NAT功能进行代理转发。
*   **`accept_dns`** (布尔值, 可选, 默认: `false`)
    *   `true`则响应来自对等节点的DNS请求。
*   **`tld_dns_zone`** (字符串, 可选, 默认: `".et"`)
    *   设置内部DNS的顶级域名。
*   **`enable_encryption`** (布尔值, 可选, 默认: `true`)
    *   `false`则禁用节点间流量加密。
*   **`encryption_algorithm`** (字符串, 可选, 默认: `"aes-gcm"`)
    *   加密算法。**完整可选项**: `"aes-gcm"`, `"aes-256-gcm"`, `"xor"`, `"chacha20"`, `"openssl-aes-gcm"`, `"openssl-chacha20"`, `"openssl-aes-256-gcm"` (后四项需在编译时开启相应特性)。
*   **`data_compress_algo`** (字符串或整数, 可选, 默认: `"none"`)
    *   数据压缩算法。可选项: `"none"` (或 `0`), `"zstd"` (或 `1`), `"lz4"` (或 `2`)。
*   **KCP 协议相关选项** (高级)
    *   **`enable_kcp_proxy`** (布尔值, 可选, 默认: `false`): 启用 KCP 协议作为节点间代理和数据传输的一种方式。KCP 是一种基于 UDP 的可靠传输协议，在高丢包、高延迟的网络环境下可能比 TCP 表现更好。
    *   **`disable_kcp_input`** (布尔值, 可选, 默认: `false`): 禁止节点接受 KCP 协议的入站连接。设置后，本节点将无法作为 KCP 连接的目标。
    *   **`disable_relay_kcp`** (布尔值, 可选, 默认: `false`): 当本节点作为中继服务器时，禁止使用 KCP 协议来转发流量。
    *   **`enable_relay_foreign_network_kcp`** (布尔值, 可选, 默认: `false`): 仅当本节点为其他外部网络（`network_name` 不同）提供中继服务时，才允许使用 KCP 协议。
*   **QUIC 协议相关选项** (高级)
    *   **`enable_quic_proxy`** (布尔值, 可选, 默认: `false`): 启用 QUIC 协议作为节点间代理和数据传输的一种方式。QUIC 是一个现代化的、基于 UDP 的加密传输协议，旨在减少连接和传输延迟。
    *   **`disable_quic_input`** (布尔值, 可选, 默认: `false`): 禁止节点接受 QUIC 协议的入站连接。
    *   **`quic_listen_port`** (整数, 可选, 默认: `0`): 指定 QUIC 协议监听的 UDP 端口。默认值为 `0`，表示不监听。您需要设置一个具体的端口（如 `11010`）来启用 QUIC 监听。
*   **`enable_exit_node`** (布尔值, 可选, 默认: `false`)
    *   `true`则允许本节点作为其他节点的出口网关。
*   **`multi_thread`** (布尔值, 可选, 默认: `true`)
    *   `true`则启用多线程处理数据，提升性能。
*   **`multi_thread_count`** (整数, 可选, 默认: `2`)
    *   多线程模式下的工作线程数。
*   **`foreign_relay_bps_limit`** (整数, 可选, 默认: 无限制)
    *   限制为不相关网络提供中继服务的最大带宽（Bps）。
