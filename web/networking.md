---
date updated: 2026-11-06T18:17
date created: 2026-10-25T14:34
---

# 网络服务

## socket

socket 是网络编程底层 API。先掌握 TCP 服务端和客户端，再学习超时、并发和协议设计。

```python
import socket


with socket.create_connection(("example.com", 80), timeout=3) as sock:
    sock.sendall(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")
    data = sock.recv(1024)
```

服务端要处理的问题：

- 连接超时和读写超时。
- 粘包和拆包，本质是应用层协议边界。
- 并发模型：线程、进程、asyncio、事件循环。
- 资源关闭：`with`、`try/finally`、信号处理。

## httpx

`httpx` 支持同步和异步 API，适合现代 HTTP 客户端。

```python
import httpx


with httpx.Client(timeout=5) as client:
    response = client.get("https://example.com")
    response.raise_for_status()
```

异步：

```python
import httpx


async def fetch(url: str) -> str:
    async with httpx.AsyncClient(timeout=5) as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.text
```

## Nginx

Nginx 常用于静态文件、反向代理、TLS 终止和负载均衡。

```nginx
server {
    listen 80;
    server_name example.com;

    location /api/ {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

注意：

- `location /api/` 配合 `proxy_pass http://host/;` 会改写路径，斜杠语义要测试。
- 80 端口被占用时先查 `lsof -i :80` 或 `ss -ltnp`，不要盲目杀进程。
- 所谓僵尸进程不能直接 kill，应该找到父进程和服务管理方式，例如 systemd unit。

## Caddy

Caddy 默认自动 HTTPS，配置更简洁，适合中小服务和内部工具。

```caddyfile
example.com {
    reverse_proxy 127.0.0.1:8000
}
```

## Consul 与 ZooKeeper

Consul 适合服务发现、健康检查和 KV 配置；ZooKeeper 更偏一致性协调、选主和分布式锁。不能简单说谁替代谁，要看需求：

- 服务发现和健康检查：优先评估 Consul。
- Kafka、HBase 等生态依赖：可能仍需要 ZooKeeper 或其替代机制。
- 配置中心：也可评估 Nacos、etcd、云厂商配置服务。

Nacos 同时提供服务注册、发现和配置管理。应用先注册服务名、地址与端口，调用方再获取可用实例；负载均衡通常由客户端或网关完成，不是 Nacos 自动转发业务请求。

接入服务治理工具时，确认健康检查、故障摘除、配置版本、权限与回滚方式。服务注册中心不负责替你部署应用，也不自动完成业务监控。

## Paramiko

Paramiko 用于 SSH 自动化，适合执行远程命令、上传下载文件。生产环境更推荐最小权限账户、密钥登录和审计日志。

```python
import paramiko


client = paramiko.SSHClient()
client.load_system_host_keys()
client.connect("server.example.com", username="deploy")
stdin, stdout, stderr = client.exec_command("uname -a")
print(stdout.read().decode())
client.close()
```

## psutil

psutil 用于进程、CPU、内存、磁盘和网络监控。

```python
import psutil

print(psutil.cpu_percent(interval=1))
print(psutil.virtual_memory().percent)
```

## 参考

- Python socket：<https://docs.python.org/3/library/socket.html>
- HTTPX 文档：<https://www.python-httpx.org/>
- Nginx proxy 文档：<https://nginx.org/en/docs/http/ngx_http_proxy_module.html>
- Caddy 文档：<https://caddyserver.com/docs/>
