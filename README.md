# Emby Reverse Proxy for Nginx Proxy Manager

在 [Nginx Proxy Manager (NPM)](https://nginxproxymanager.com/) 中反向代理 Emby 服务器的配置范例，核心目标：**隐藏反代特征、支持流媒体拖拽、防止被后端识别**。

## 适用场景

| 场景 | 说明 |
|------|------|
| 前后端不分离 | Emby 主站和推流在同一台服务器上，无独立推流节点 |
| 前后端分离 | Emby 主站与推流节点分离，后端返回 302 重定向到独立的推流域名 |
| LilyEmby | 需要对响应体做域名替换的特殊场景（如 JSON/XML 中包含硬编码的源站域名） |

## 快速开始

### 1. 新建 Proxy Host — Details 页

填写你的域名、后端 Emby 地址和端口，**务必开启 Websockets Support**。

![Details 页配置](screenshots/detail-page.png)

### 2. SSL 页

选择 SSL 证书，**至少开启 Force SSL**。

![SSL 页配置](screenshots/ssl-page.png)

### 3. Advanced 页

根据你的实际情况，从下面三种配置中选择一种，将内容粘贴到 **Custom Nginx Configuration** 文本框中，**修改其中的示例域名为你自己的域名**，然后保存。

## 配置说明

### 方案一：前后端不分离（[no-separation.txt](advanced-examples/no-separation.txt)）

> **适用于**：Emby 主站和推流在同一台服务器上，无独立推流节点。

最简单的配置。只需一个 `location /` 块，核心做三件事：

1. **伪装 Host** — 将 `Host` 和 `proxy_ssl_name` 设为后端实际地址，避免 SNI 不匹配和后端检测
2. **清除代理特征头** — 置空 `X-Real-IP`、`X-Forwarded-For`、`Via` 等所有暴露反代身份的请求头
3. **流媒体支持** — 透传 `Range` / `If-Range` 头以支持视频拖拽和断点续传，关闭 `proxy_buffering` 降低延迟

<details>
<summary>点击展开完整配置</summary>

```nginx
location / {
    # ---------------- 基础反代目标 ----------------
    proxy_pass $forward_scheme://$server:$port;

    # ---------------- 伪装后端 (防 Emby 识破) ----------------
    proxy_set_header Host $server;
    proxy_ssl_server_name on;
    proxy_ssl_name $server;

    # 覆盖并清空 NPM 默认自带的代理特征头
    proxy_set_header X-Real-IP "";
    proxy_set_header X-Forwarded-For "";
    proxy_set_header X-Forwarded-Proto "";
    proxy_set_header X-Forwarded-Host "";
    proxy_set_header X-Forwarded-Port "";
    proxy_set_header Via "";

    # ---------------- 流媒体支持 ----------------
    proxy_set_header Range $http_range;
    proxy_set_header If-Range $http_if_range;

    proxy_connect_timeout 60s;
    proxy_send_timeout 180s;
    proxy_read_timeout 180s;
    proxy_buffering off;

    # ---------------- 伪装客户端 (防客户端识破) ----------------
    proxy_hide_header X-Powered-By;
    proxy_hide_header X-Frame-Options;
    proxy_hide_header X-Content-Type-Options;
    more_clear_headers 'Server';
}
```

</details>

---

### 方案二：前后端分离（[separation.txt](advanced-examples/separation.txt)）

> **适用于**：Emby 后端返回 302 重定向到独立推流域名（如 `stream1.example.com`、`stream2.example.com`）的部署架构。

在方案一的基础上增加了 **redirect 拦截 + 推流节点代理**：

1. **`proxy_redirect` 拦截重定向** — 将后端返回的推流域名重写为本地路径（如 `/s1/`、`/s2/`、`/s3/`）
2. **独立 `location` 块** — 为每个推流节点定义独立的代理入口，`rewrite` 去掉路径前缀后转发到真实推流节点
3. **每个节点独立伪装** — 推流节点的 `Host`、`Referer`、代理特征头各自独立处理

使用前需要修改以下占位符：

| 占位符 | 替换为 |
|--------|--------|
| `stream1.example.com` | 推流节点 1 的真实域名 |
| `stream2.example.com` | 推流节点 2 的真实域名 |
| `stream3.example.com` | 推流节点 3 的真实域名 |
| `yourdomain.com` | 你自己的反代域名 |
| `example-emby.com` | Emby 主站的源域名 |

> **提示**：推流节点数量可以按需增减，复制 `/s1/` 块并修改编号和域名即可。

<details>
<summary>点击展开完整配置</summary>

```nginx
# 核心入口：反代 Emby 主程序
location / {
    proxy_pass $forward_scheme://$server:$port;

    proxy_set_header Host $server;
    proxy_ssl_name $server;
    proxy_ssl_server_name on;

    proxy_set_header Range $http_range;
    proxy_set_header If-Range $http_if_range;

    # 将推流源重定向到伪装路径
    proxy_redirect http://stream1.example.com https://yourdomain.com/s1/;
    proxy_redirect http://stream2.example.com https://yourdomain.com/s2/;
    proxy_redirect http://stream3.example.com https://yourdomain.com/s3/;

    proxy_set_header X-Real-IP "";
    proxy_set_header X-Forwarded-For "";
    proxy_set_header X-Forwarded-Proto "";
    proxy_set_header X-Forwarded-Host "";
    proxy_set_header Forwarded "";
    proxy_set_header Via "";

    more_clear_headers 'Server';
    proxy_hide_header X-Powered-By;
}

# 推流节点 1
location /s1/ {
    rewrite ^/s1(/.*)$ $1 break;
    proxy_pass http://stream.example.com;

    proxy_set_header Referer "https://example-emby.com/web/index.html";
    proxy_set_header Host $proxy_host;
    proxy_set_header Range $http_range;
    proxy_set_header If-Range $http_if_range;

    proxy_buffering off;
    proxy_connect_timeout 60s;
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;

    proxy_set_header X-Real-IP "";
    proxy_set_header X-Forwarded-For "";
    proxy_set_header X-Forwarded-Proto "";
    proxy_set_header X-Forwarded-Host "";
    proxy_set_header Forwarded "";
    proxy_set_header Via "";

    more_clear_headers 'Server';
    proxy_hide_header X-Powered-By;
}

# 推流节点 2、3 结构相同，修改域名即可（完整版见源文件）
```

</details>

---

### 方案三：LilyEmby 特殊版（[lily-special.txt](advanced-examples/lily-special.txt)）

> **适用于**：后端在 JSON / XML 响应体中硬编码了源站域名，仅靠 `proxy_redirect` 无法覆盖所有场景。

在方案二的基础上引入了 **`sub_filter` 响应体替换**：

1. **强制禁用压缩** — `proxy_set_header Accept-Encoding ""` 使上游返回明文，否则 `sub_filter` 无法工作
2. **扩展替换范围** — `sub_filter_types` 覆盖 `application/json`、`text/xml`、`text/plain`
3. **全局替换** — `sub_filter_once off` 替换所有匹配项
4. **域名映射** — 将响应体中的源站域名和推流 CDN 域名全部替换为反代域名

使用前需要修改以下占位符：

| 占位符 | 替换为 |
|--------|--------|
| `www.example.com` | Emby 主站的源域名 |
| `stream.example.com` | 推流节点的真实域名 |
| `lily.yourdomain.com` | 你自己的反代域名 |
| `example-emby.com` | 用于伪装 Referer 的 Emby 域名 |

<details>
<summary>点击展开完整配置</summary>

```nginx
# 核心入口：反代 Emby 主站
location / {
    proxy_pass $forward_scheme://$server:$port;

    proxy_set_header Host $server;
    proxy_ssl_name $server;
    proxy_ssl_server_name on;

    proxy_set_header Range $http_range;
    proxy_set_header If-Range $http_if_range;

    # 处理 302 重定向
    proxy_redirect https://stream.example.com https://lily.yourdomain.com/s1;
    proxy_redirect https://www.example.com https://lily.yourdomain.com;

    # 响应体替换（核心）
    proxy_set_header Accept-Encoding "";
    sub_filter_types application/json text/xml text/plain;
    sub_filter_once off;
    sub_filter 'https://www.example.com' 'https://lily.yourdomain.com';
    sub_filter 'https://stream.example.com' 'https://lily.yourdomain.com/s1';

    proxy_set_header X-Real-IP "";
    proxy_set_header X-Forwarded-For "";
    proxy_set_header X-Forwarded-Proto "";
    proxy_set_header X-Forwarded-Host "";
    proxy_set_header Forwarded "";
    proxy_set_header Via "";

    more_clear_headers 'Server';
    proxy_hide_header X-Powered-By;
}

# 推流节点代理
location /s1/ {
    rewrite ^/s1(/.*)$ $1 break;
    proxy_pass https://stream.example.com;

    proxy_ssl_server_name on;
    proxy_ssl_name stream.example.com;

    proxy_set_header Range $http_range;
    proxy_set_header If-Range $http_if_range;
    proxy_set_header Host $proxy_host;
    proxy_set_header Referer "https://example-emby.com/web/index.html";

    proxy_buffering off;
    proxy_connect_timeout 60s;
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;

    proxy_set_header X-Real-IP "";
    proxy_set_header X-Forwarded-For "";
    proxy_set_header X-Forwarded-Proto "";
    proxy_set_header X-Forwarded-Host "";
    proxy_set_header Forwarded "";
    proxy_set_header Via "";

    more_clear_headers 'Server';
    proxy_hide_header X-Powered-By;
}
```

</details>

## 三种方案对比

| 特性 | 不分离 | 分离 | LilyEmby |
|------|:------:|:----:|:--------:|
| 配置复杂度 | 低 | 中 | 高 |
| 独立推流节点 | - | 多节点 | 单/多节点 |
| 302 重定向拦截 | - | `proxy_redirect` | `proxy_redirect` |
| 响应体域名替换 | - | - | `sub_filter` |
| 隐藏代理特征 | 请求头 + 响应头 | 请求头 + 响应头 | 请求头 + 响应头 + 响应体 |

**选择建议**：

- 只有一台 Emby 服务器、无推流分离 → **方案一**
- 后端有独立推流节点、302 跳转到推流域名 → **方案二**
- 后端在 API 响应中硬编码了源站域名 → **方案三**

## 关键指令速查

| 指令 | 作用 | 为什么需要 |
|------|------|-----------|
| `proxy_set_header Host $server` | 伪装 Host 为后端地址 | 防止后端检测到反代域名 |
| `proxy_ssl_name` / `proxy_ssl_server_name on` | 设置 SNI | HTTPS 后端必须，否则证书校验失败 |
| `proxy_set_header X-Real-IP ""` | 清空代理特征头 | NPM 默认会加这些头，暴露反代身份 |
| `proxy_set_header Range $http_range` | 透传 Range 头 | 视频拖拽/断点续传必需 |
| `proxy_buffering off` | 关闭响应缓冲 | Emby 官方推荐，降低播放延迟 |
| `more_clear_headers 'Server'` | 清除 Server 响应头 | 隐藏 Nginx 身份（依赖 NPM 内置 OpenResty） |
| `proxy_redirect` | 重写 302 重定向目标 | 拦截推流域名跳转 |
| `sub_filter` | 替换响应体内容 | 处理 JSON/XML 中硬编码的源站域名 |

## 注意事项

- `more_clear_headers` 指令依赖 OpenResty 的 `headers-more-nginx-module` 模块，NPM 默认已内置，无需额外安装
- `sub_filter` 和 gzip 压缩互斥 — 使用方案三时必须通过 `Accept-Encoding ""` 禁用上游压缩
- 所有配置中的 `$forward_scheme`、`$server`、`$port` 是 NPM 内置变量，会自动替换为 Details 页填写的值，**不需要手动修改**
- 超时时间可按需调整，推流节点建议设置更长的 `proxy_read_timeout`（默认 300s）以应对大文件播放
