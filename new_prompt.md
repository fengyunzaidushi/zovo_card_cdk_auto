# 在当前 Ubuntu 服务器部署 ZovoCard CDK：gptsub.org

> 这是一份交给服务器端 AI 执行的部署提示词。编写时仅做了环境调查，没有执行部署。
> 调研日期：2026-10-19（Asia/Taipei）。下面的现状是调查快照，执行时须先复核。

你是一名负责生产服务器的 Linux 运维工程师。请直接在当前服务器上，将现有官方项目 `/opt/zovo_card_cdk_auto` 部署到 `https://gptsub.org`，完成逐项验收，并保留可执行的维护、备份与回滚说明。

本提示词替代旧版部署提示词。请按当前多站共存环境操作，不要照搬官方独占 80/443 的一键脚本。

## 一、已知环境和部署决策

| 项目 | 已调查结果 |
|---|---|
| 操作位置 | AI 已在目标服务器上运行，可直接执行命令，无需 SSH |
| 系统 / 架构 | Ubuntu 24.04.1 LTS，x86_64 |
| 公网 IPv4 | `64.83.10.143`；调用公网出口查询接口得到相同结果 |
| 域名 | `gptsub.org`；用户已在 Cloudflare 配置 A 记录指向上述 IP，并开启橙云 |
| DNS 公网响应 | 调查时为 `104.21.61.183`、`172.67.212.198` 和 Cloudflare IPv6，属于橙云正常行为 |
| Docker | Engine `29.8.1`，Compose `v5.5.1`，已安装且运行正常 |
| 资源 | 总内存约 3.8 GiB、available 约 2.3 GiB、无 swap；根盘剩余约 59 GiB |
| 当前项目 | `acg-faka`、`aigptpro`、`ailinkvcc`；6 个运行容器健康，另有 1 个退出码为 0 的初始化容器 |
| 公网入口 | `acg-faka-app` 发布宿主机 TCP 80、443，容器内 Nginx 已代理多个站点 |
| 共享网络 | `acg-faka_default`，当前网段 `172.18.0.0/16`；Nginx 容器当前 IP 为 `172.18.0.4` |
| 其他端口 | 宿主机已有 `172.18.0.1:8080` 的 sub2api，以及 `8443` 等服务；SSH 实际监听 `55889` |
| 防火墙 | 调查时 UFW inactive；不代表不存在 Docker、nftables 或云防火墙规则 |
| 源码 | `/opt/zovo_card_cdk_auto`，调查时提交 `b28ea2f`，根目录 `VERSION=1.4.30` |
| 部署目录 | `/opt/cdk-recharge`，调查时尚不存在；本次用于独立部署配置、环境文件及数据 |
| 应用存储 | SQLite，不需要新建或使用现有 PostgreSQL / MySQL / Redis |

采用以下链路：

```text
浏览器 https://gptsub.org
    → Cloudflare 橙云
    → 当前服务器 443
    → acg-faka-app 中的 Nginx：新增 server_name gptsub.org
    → acg-faka_default 网络内的 zovo-cdk-web:8080
    → ZovoCard CDK 容器 / SQLite
```

**无需新增宿主机端口映射。** 不部署第二套占用 80/443 的 Caddy，不把新站发布到公网 8080 或 8443。新容器自己的 8080 与宿主机已有 8080 服务处于不同网络命名空间，不冲突。容器中的 `127.0.0.1` 指向该容器自身，不能将 Nginx 的上游错误地写成 `127.0.0.1:8080`。

现有入口的持久化结构：

- 容器：`acg-faka-app`。
- 数据卷：`acg-faka_acg_data`，挂载至容器 `/data`。
- Nginx 主配置已经包含 `include /data/nginx/*.conf;`。
- 已有配置：`ssl.conf`（`accpilot.online`）、`aigptpro.conf`（`aigptpro.online`）、`sub2api.conf`（`sub2.sub2api.online`）、`gptsub.conf`（`gptsub.sub2api.online`）。
- **`gptsub.conf` 不是新站配置，禁止覆盖。** 新文件使用 `/data/nginx/zovo-gptsub-org.conf`，新证书名称使用 `zovo-gptsub-org`。
- 容器内已有 Certbot，证书目录 `/data/ssl`；Supervisor 的 `ssl-renew` 进程正在运行，约每 12 小时调用续期脚本并检查、重载 Nginx。

调查时 Nginx 没有 `gptsub.org` 的虚拟主机，已有证书也没有覆盖该域名。公网 HTTPS 虽返回 200，但显示 Next.js 响应头；HTTP 显示 ACG 响应。直接连接本机并指定 `gptsub.org` 的 TLS 校验报域名不匹配。这些现象与落入默认站点一致，**不能当成新站已部署或证书正确的证据**。Cloudflare 后台的实际加密模式、Origin Rules、Workers 与缓存规则尚未核实。

## 二、执行范围和约束

1. 本提示词授权完成本项目的构建、启动、独立虚拟主机及证书配置、验证和故障回滚。常规可逆步骤说明后继续，不要每一步重复请求确认。只有实际需要用户网页操作、缺少无法推断的信息、需要超出本提示词范围，或必须中断其他项目时，才暂停相关步骤并说明具体原因。
2. 不停止、重建、删除现有项目的容器、数据卷、网络和数据库。不重启 Docker，不修改 SSH，不启停或重置防火墙，不安装宿主机 Nginx 接管端口。只允许对现有 Nginx **新增本域名配置并在检查通过后平滑 reload**。
3. 不运行原始 `deploy/docker-deploy.sh`：它会生成 Caddy 配置并启动整套 Compose，与现有入口冲突。不运行 `acg-ssl` / `acg-ssl-import` 为新域名配证书：已检查其实现，会写入现有 `/data/nginx/ssl.conf`，部分脚本使用固定证书名 `acg`，会影响旧站。
4. 保留官方业务源码、Dockerfile 和仓库自带 Compose/Caddy 文件。允许在 `/opt/cdk-recharge` 生成独立的生产 Compose、Nginx 模板、环境文件、备份和维护脚本；这属于本次部署范围。不要强制清理源码工作区、覆盖 `old_prompt.txt` / `new_prompt.md` 或其他用户改动。
5. 所有部署命令显式指定项目名 `zovo-cdk`、配置文件及环境文件，避免误操作其他项目。不要裸执行仓库默认的 `docker compose up -d`，不要执行 `down -v` 或全局 prune。
6. 不向用户索取或输出管理员密码、JWT 密钥、卡台 API Key、Session、Access Token、完整卡密、Webhook 密钥、证书私钥及已有环境文件。生成秘密时关闭命令回显，只写入权限 600 的文件。查看配置使用定向字段或 `docker compose config -q`，避免展开完整环境变量。
7. 一次性 Setup Token 是唯一允许交付给用户的安装凭据，只单独提供一次；其他秘密由用户在网页输入。不要将 Setup Token 放进普通报告、Git 或 Nginx 配置。
8. 日志与报错先脱敏再报告，保留错误码和上下文。不要原样输出应用请求日志、环境变量或包含查询参数的敏感 URL。后端 Gin 默认请求日志可能记录 URL 查询参数，真实 CDK / token 不应用于命令行探测。
9. 禁用应用内在线更新，设置 `UPDATE_ENABLED=0`。升级通过固定版本镜像与显式 Compose 操作完成，不能重新运行原一键部署脚本。
10. 不用真实卡密发码、充值、兑换或扣费来验收。本次完成基础设施和不扣费的连通验证；付费业务验证另行由用户决定。

## 三、先只读复核并记录基线

检查系统、资源、Docker、网络、端口和源码状态，记录旧站首页/健康接口状态及容器 ID。资源数字以执行时为准。宿主机无需安装 Go/Node，使用官方多阶段 Dockerfile 构建。

```bash
uname -m
cat /etc/os-release
whoami
free -h
df -h / /opt
docker version
docker compose version
docker compose ls
docker ps -a
ss -lntup
docker network inspect acg-faka_default --format '{{json .IPAM.Config}}'
docker inspect acg-faka-app --format '{{json .Mounts}} {{json .NetworkSettings.Networks}}'
docker exec acg-faka-app nginx -t
git -C /opt/zovo_card_cdk_auto status --short
git -C /opt/zovo_card_cdk_auto log -1 --format='%H %s'
getent ahostsv4 gptsub.org
curl -4 -fsS --max-time 15 https://api.ipify.org
```

公网 DNS 返回 Cloudflare IP 是正常现象，不要求等于源站 `64.83.10.143`，也不因返回 Cloudflare IPv6 就要求删除 AAAA。公开 DNS 查询不能证明 Cloudflare 内部配置的源站地址，应结合用户已提供的信息、源站测试和公网测试判断；如二者不一致，再排查 Cloudflare 后台配置。

定向查看现有 Nginx 的 `server_name`、include、证书路径和代理目标，复核新域名与文件名没有冲突。备份将涉及的配置及记录哈希，备份放权限 700 的专用目录；不要改动已有证书或复制其私钥到输出。

已有域名 `accpilot.online`、`aigptpro.online`、`sub2.sub2api.online`、`gptsub.sub2api.online` 均纳入变更前后回归。遇到原有异常先记录，不能把原有 403/跳转等一概判为本次引入。

## 四、创建独立应用部署

1. 建立 `/opt/cdk-recharge`、`/opt/cdk-recharge/data`、`/opt/cdk-recharge/nginx`。部署目录和数据目录应限制访问。若已经存在，则先辨识其用途、备份，幂等复用，禁止清空。
2. 使用当前已检查的源码提交构建；镜像标签包含版本及提交号，记录构建所用完整 commit。不要部署前未经核对就 `git pull` 更换代码。
3. 在 `/opt/cdk-recharge/.env` 中安全生成/保留 `JWT_SECRET`（`openssl rand -hex 32`）；写入唯一镜像标签 `CDK_IMAGE` 和 `CDK_TRUSTED_PROXIES`。代理信任优先使用现场查得的 Nginx 容器 IP `/32`，调查时为 `172.18.0.4/32`。Nginx 容器重建后需复核该地址并同步更新 CDK 环境，不将所有公网或所有 Docker 网段默认加入信任。
4. `.env` 权限为 600，不设置 `ADMIN_PASSWORD`，使用网页安装向导。应用自定义的 `CORS_ALLOWED_ORIGINS` 设置为 `https://gptsub.org`。
5. 创建 `/opt/cdk-recharge/compose.prod.yaml`，可按下列模板落地。独立配置不要与官方默认 Compose 合并，以免意外带入 Caddy 和端口发布：

```yaml
name: zovo-cdk
services:
  cdk:
    build:
      context: /opt/zovo_card_cdk_auto
      dockerfile: Dockerfile
    image: ${CDK_IMAGE:?set CDK_IMAGE}
    container_name: zovo-cdk-app
    restart: unless-stopped
    environment:
      SERVER_HOST: 0.0.0.0
      SERVER_PORT: "8080"
      SERVER_MODE: release
      DB_PATH: /app/data/cdk_recharge.db
      WEB_DIR: /app/web
      INSTALL_MODE: wizard
      JWT_SECRET: ${JWT_SECRET:?set JWT_SECRET}
      TRUSTED_PROXIES: ${CDK_TRUSTED_PROXIES:?set CDK_TRUSTED_PROXIES}
      CORS_ALLOWED_ORIGINS: https://gptsub.org
      UPDATE_ENABLED: "0"
    volumes:
      - /opt/cdk-recharge/data:/app/data
    expose:
      - "8080"
    networks:
      edge:
        aliases:
          - zovo-cdk-web
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://127.0.0.1:8080/health"]
      interval: 30s
      timeout: 5s
      start_period: 20s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
networks:
  edge:
    external: true
    name: acg-faka_default
```

此模板没有 `ports`，不会增加公网监听。加入共享网络也意味着与现有容器共享网络可达性；此次沿用已经使用的入口结构，不擅自重构其他项目网络。

以统一参数执行：

```bash
docker compose --env-file /opt/cdk-recharge/.env -p zovo-cdk -f /opt/cdk-recharge/compose.prod.yaml config -q
docker compose --env-file /opt/cdk-recharge/.env -p zovo-cdk -f /opt/cdk-recharge/compose.prod.yaml build cdk
docker compose --env-file /opt/cdk-recharge/.env -p zovo-cdk -f /opt/cdk-recharge/compose.prod.yaml up -d --no-deps cdk
docker compose --env-file /opt/cdk-recharge/.env -p zovo-cdk -f /opt/cdk-recharge/compose.prod.yaml ps
docker exec zovo-cdk-app curl -fsS http://127.0.0.1:8080/health
```

构建期间留意宿主机 available 内存、负载和旧站健康。发生 OOM 或构建失败时分析后调整；不要自动添加 swap、重启服务或无休止重试。官方后端构建需要 CGO，不改成 `CGO_ENABLED=0`。

在首次启动后及时提取并安全保存一次性 Setup Token 到 root 专用文件，权限 600，避免日志轮转丢失。此时先不要让用户在 HTTP 页面提交它。

## 五、为新域名配置证书及现有 Nginx

先在 `/opt/cdk-recharge/nginx` 保存配置源文件，再安装到现有容器的持久化 `/data/nginx`。每次替换前保留本文件上一版，替换后执行 `nginx -t`；失败则恢复本次变更，成功后才 `nginx -s reload`。不要重启 `acg-faka-app`。

### 5.1 首选：复用容器内 Certbot 的 webroot 签发

1. 检查运行中容器的实际续期脚本及进程状态。只读检查已有 ACME 账号是否可复用，不输出账号私钥。
2. 先安装仅监听 80 的 `server_name gptsub.org` 配置：`/.well-known/acme-challenge/` 使用 `/var/www/html` webroot，其他路径暂时返回 503。此阶段不配置指向尚不存在证书的 443 块，也不开放明文安装界面。
3. 写入一个随机、无秘密的临时 challenge 文件，分别通过本机 Host 路由和公网 `http://gptsub.org/.well-known/acme-challenge/<文件名>` 读取，核对正文完全相同；验证后删除。不得仅检查状态码。
4. 验证通过再用 `certbot certonly --webroot`，不使用 standalone、不占用新监听端口、不调用覆盖旧站的封装脚本。沿用现有 ACME 账号；如有多个账号，明确选择，避免交互挂起。示例（账号参数按现场补充）：

```bash
docker exec acg-faka-app certbot certonly \
  --webroot -w /var/www/html \
  --cert-name zovo-gptsub-org \
  -d gptsub.org \
  --non-interactive --agree-tos \
  --config-dir /data/ssl \
  --work-dir /data/ssl/work \
  --logs-dir /data/ssl/logs
```

已有可用证书时先复用，不强制重复签发。若没有可复用账号，再获取所需的非敏感注册信息，不猜测用户邮箱。

Cloudflare 若对 challenge 路径强制跳转到尚未配置的 HTTPS、发起人机验证或返回缓存内容，先定位具体规则；不要盲目重复申请，不把关闭整站橙云或安全防护当默认解法。需要用户在 Cloudflare 针对该域名的 challenge 路径调整规则时，准确说明路径和原因。只有该链路无法打通时，才改用 DNS-01 或 Cloudflare Origin CA，并先准备服务器端配置，再列出必要的用户操作；凭据和私钥不进聊天。

### 5.2 正式 HTTPS 虚拟主机要求

证书验证包含 `gptsub.org` 且有效后，将新配置换为 HTTP challenge + HTTPS 反代。HTTP 除 challenge 外跳转到 `https://gptsub.org$request_uri`；仅声明 `gptsub.org`，不覆盖旧域名，不添加 `default_server`。

443 块至少包含下列配置，并保留 HTTP 和 HTTPS 的 challenge 路径：

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name gptsub.org;

    ssl_certificate /data/ssl/live/zovo-gptsub-org/fullchain.pem;
    ssl_certificate_key /data/ssl/live/zovo-gptsub-org/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    include /data/nginx/includes/zovo-gptsub-org-cloudflare.inc;
    resolver 127.0.0.11 valid=10s ipv6=off;
    resolver_timeout 5s;

    access_log off;
    error_log /dev/stderr crit;
    client_max_body_size 2m;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type text/plain;
        try_files $uri =404;
    }

    location ~ /\. { return 404; }
    location ~* ^/(data|backup|backups)(/|$) { return 404; }

    location / {
        set $zovo_cdk_backend zovo-cdk-web:8080;
        proxy_pass http://$zovo_cdk_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;
        proxy_set_header Connection "";
        proxy_connect_timeout 5s;
        proxy_read_timeout 180s;
        proxy_send_timeout 180s;
        proxy_next_upstream off;
        proxy_cache off;
        proxy_buffering off;
        proxy_hide_header Cache-Control;
        add_header Cache-Control "no-store" always;
    }
}
```

`zovo-gptsub-org-cloudflare.inc` 在本虚拟主机范围内配置官方 Cloudflare IPv4/IPv6 网段的 `set_real_ip_from`、`real_ip_header CF-Connecting-IP`、`real_ip_recursive on`。从官方地址列表获取并验证内容格式后生成，避免空列表、直接执行下载文本或信任任意来源头。单独保存本域名 include，不修改旧站共用文件。

这样 Nginx 先从可信 Cloudflare 连接恢复访客 IP，再覆盖传给 Go 的转发头，避免所有访客共用一个限流 IP。HTTPS 转发头必须保留，以使管理员 Cookie 带 `Secure`。动态 Docker DNS 解析用于支持后续 CDK 容器换 IP；不要硬编码新应用容器 IP。

先检查旧站基线，再执行：

```bash
docker exec acg-faka-app nginx -t
docker exec acg-faka-app nginx -s reload
```

证书签发成功后，确认 `/data/ssl/renewal/zovo-gptsub-org.conf` 存在、使用正确 webroot，现有 `ssl-renew` 进程能扫描到它；针对新证书执行一次 `certbot renew --cert-name zovo-gptsub-org --dry-run`（同样指定现有 config/work/logs 目录），不要改动旧证书续期定义。报告续期是否验证成功。

## 六、Cloudflare 设置与分层验收

源站证书正确后，Cloudflare 此域名应采用 **Full (strict)**，不能以 Flexible 作为最终配置。无法直接查看后台时明确列为“待用户确认”，不凭 HTTP 200 推断已开启 strict。若该 Zone 还托管其他源站，避免未经检查改变其全局加密设置。

检查是否有 Workers、Origin Rules、端口重写、Host 重写或“Cache Everything”规则影响新站。至少让 `/api/*`、`/ops*` 不被缓存，Webhook 与正常 API 请求不应被交互式验证码拦截；如必须调整 WAF，仅针对实际受影响的本域名路径处理。服务器的 `no-store` 不替代对 Cloudflare 强制缓存规则的核对。

逐层验证，不用真实卡密、Session 或管理员密码：

```bash
# 应用内部
docker exec zovo-cdk-app curl -fsS http://127.0.0.1:8080/health
docker exec zovo-cdk-app curl -fsS http://127.0.0.1:8080/api/v1/setup/status

# 现有反代容器到新应用（先检查容器内 curl 可用）
docker exec acg-faka-app curl -fsS --max-time 15 http://zovo-cdk-web:8080/health

# 绕过 Cloudflare，但保留域名、SNI 与证书验证
curl -fsS --max-time 15 --resolve gptsub.org:443:64.83.10.143 https://gptsub.org/health

# 经过 Cloudflare 的真实入口
curl -fsS --max-time 20 https://gptsub.org/health
curl -fsS --max-time 20 https://gptsub.org/api/v1/setup/status
curl -fsS --max-time 20 https://gptsub.org/api/v1/public/site
```

必须核对 JSON 内容、Content-Type、页面及静态资源是否属于本项目，不能仅检查 200；检查首页、`/ops/setup`、`/ops/login`、`/recharge` 的 SPA 回退和资源加载。HTTP 应跳转 HTTPS，ACME 路径仍可访问。匿名访问管理 API 应拒绝，检查敏感路径被拒绝。

若最终使用 Cloudflare Origin CA，普通系统信任库的直连 curl 可能不信任它，应提供正确 CA 验证并检查域名，不将 `curl -k` 的成功算作 TLS 验收。

部署后复查旧站、容器 ID/启动时间、监听端口及 Nginx 配置哈希：应只增加 CDK 容器和本域名配置，旧容器无需重建。发生回归立即撤回本次新配置并平滑 reload，再定位原因。

## 七、交付安装入口并做上游连通检查

1. HTTPS 正常后，单独交付 Setup Token，告诉用户打开 `https://gptsub.org/ops/setup` 创建管理员。管理员密码由用户自己保存。首次 `setup/status` 应为 `installed:false`；如果已经是 true，先辨识是否已有安装，禁止清库重装。
2. 用户完成后，再验证 `https://gptsub.org/api/v1/setup/status` 返回 `installed:true`。完成安装会使 Setup Token 失效；删除本次保存的明文临时 token 文件。用户尚未操作时，继续完成独立的文档、备份准备等工作，最终如实标注待办。
3. 交付 `https://gptsub.org/ops/integration`，由用户配置卡台 API Key。白名单应填写服务器实际出站 IP，调查时是 `64.83.10.143`，不能填写域名解析出来的 Cloudflare IP。部署后也应从 CDK 容器验证 IPv4 出口；如上游域名同时支持 IPv6，核实实际出站路径，不能只用强制 IPv4 的结果断言全部调用都会走 IPv4。
4. 回调地址为 `https://gptsub.org/api/v1/webhooks/cardplatform`。用户在卡台和本站设置一致的 Webhook 签名密钥，禁止为验收跳过验签。可由用户使用卡台提供的测试功能验证回调，不制造付费订单。
5. 用户配置完成后，检查后台连通测试和脱敏后的 `plan-sync` 日志，再查询公开套餐接口。默认 GPT 套餐接口的 `source` 必须为 `cardplatform_live` 且有可售套餐，才能证明实时上游查询成功。`docs_default` / `docs_default_fallback` 即使返回 200 也不是通过；X 产品分支返回结构不同，不强行要求相同 source 字段。
6. 401 优先核对 Key 有效性；403 同时检查 IP 白名单、权限与上游响应；429 按上游提示退避；超时分开检查 DNS、TLS、网络、上游和 Cloudflare 超时。不要仅按一个 HTTP 状态码武断确定原因。

等待用户网页操作不等于整个部署失败。应区分“基础设施已部署”“管理员待初始化”“卡台待配置”“业务未做扣费验收”。

## 八、备份、升级与回滚交付

在 `/opt/cdk-recharge/OPERATIONS.md` 写清实际路径、Compose 命令、镜像标签、commit、证书归属、依赖的外部网络及反代容器。

- **备份范围**：SQLite 数据目录、`.env`、生产 Compose、本域名 Nginx 配置和 include；证书恢复需考虑 Certbot live 的链接与 archive、renewal 目录。备份含敏感信息，目录权限 700、文件 600，不进 Git。
- **一致性**：禁止直接复制运行中的 SQLite 主文件当完整备份。优先使用 SQLite backup API；若没有已验证可用的在线方案，使用短暂停止仅 `zovo-cdk` 的冷备。停服前说明影响；新站首次尚无用户流量时可完成初始化备份，已投入使用且未约定窗口时再询问。脚本用退出清理保证归档失败也恢复该容器，不停止 Nginx 或旧站。
- **验证备份**：检查归档列表、大小、权限和摘要，在隔离临时目录验证 SQLite `integrity_check`，不把验证恢复写回生产数据目录。
- **升级**：先备份和记录旧镜像 ID；在明确版本的干净源码副本中构建新标签，修改 `CDK_IMAGE`，只执行本项目 `up -d --no-deps cdk`。保留 JWT、数据与旧镜像。不要自动修改当前工作区中的用户文件，不用后台在线更新或原始一键脚本。
- **回滚**：新站接入失败时，仅撤回本次新增 Nginx 配置，检查后 reload，并停止新 CDK 容器；保留数据用于排查。已有新站升级失败时回到旧镜像；若涉及不兼容数据库迁移，停止 CDK 后恢复对应备份，不承诺只换镜像必然足够。
- **依赖说明**：新站入口依赖 `acg-faka-app`、其持久化 Nginx/证书配置及 `acg-faka_default` 网络。将来维护该入口时要一起回归此站；容器 IP 变化还要更新 `CDK_TRUSTED_PROXIES`。

最终报告应包含：部署结果、网址、所用镜像/commit、实际网络链路、健康与 TLS/续期验证、旧站回归结果、文件与备份位置、待用户完成的具体步骤。未配置 API Key 或未做真实兑换时明确指出，不虚称业务全流程验收通过。

## 九、查证参考

以下为本提示词涉及机制的官方资料；执行时优先结合当前服务器实况和当前源码：

- [Cloudflare 橙云与 DNS 返回值](https://developers.cloudflare.com/dns/proxy-status/)
- [Cloudflare Full (strict) 的源站证书要求](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/)
- [Cloudflare Origin CA 的适用范围与直连限制](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)
- [Cloudflare 可代理端口](https://developers.cloudflare.com/fundamentals/reference/network-ports/)
- [Cloudflare CF-Connecting-IP 请求头](https://developers.cloudflare.com/fundamentals/reference/http-headers/)
- [Cloudflare 官方 IP 范围](https://www.cloudflare.com/ips/)
- [Docker Compose 使用现有 external 网络](https://docs.docker.com/compose/how-tos/networking/#use-an-existing-network)
- [Nginx proxy_pass 与动态解析](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)
- [Let's Encrypt HTTP-01 / DNS-01 验证](https://letsencrypt.org/docs/challenge-types/)
