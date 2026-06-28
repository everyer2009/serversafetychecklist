# 自建服务器安全自检清单

> 写给自己跑服务器的姐妹们——不需要成为安全专家，但这些基础坑要知道。
> 大约每隔 1-2 个月做一次全面检查，日常有改动时随时对照。
> 直接交给你的机让他读了结合你们的本地情况去做。修改完记得一定要机或者机的分身假装自己是超级大坏蛋骇客戴上黑帽子来模拟一下看方便不方便找到软柿子
（整个安全修复和检查会经历很多轮，记得做）


---

## 一、SSH 入口：最重要的门

SSH 是你进服务器的大门，这扇门的锁要最结实。

**必须做：**
- [ ] **禁用密码登录，只允许 SSH key 登录**
  - 改 `/etc/ssh/sshd_config`：`PasswordAuthentication no`
  - 改完重启：`systemctl restart sshd`
  - 改之前先确认自己的 key 能登进去，否则会把自己锁在外面
- [ ] **不要用默认 22 端口**（可选，改成非常用端口如 2222 减少扫描噪音）
- [ ] **禁止 root 直接登录**：`PermitRootLogin no`（改用普通用户 + sudo）
- [ ] **定期检查 authorized_keys**，只留自己认识的 key
  ```bash
  cat ~/.ssh/authorized_keys
  ```

**检查是否有人在尝试暴力破解：**
```bash
grep "Failed password" /var/log/auth.log | tail -20
# 或
journalctl -u sshd | grep "Failed" | tail -20
```
几十次 Failed 是正常的扫描噪音，几千次以上建议换端口或装 fail2ban。

---

## 二、防火墙：只开该开的口

原则：**默认全关，只放行需要的端口**。双层防护最稳——云厂商安全组 + 服务器本地 UFW。

**UFW 基础操作：**
```bash
ufw status verbose          # 查现状
ufw default deny incoming   # 默认拒绝所有入站
ufw allow 22/tcp            # 放行 SSH（或你改的端口）
ufw allow 80/tcp            # HTTP
ufw allow 443/tcp           # HTTPS
ufw enable
```

**检查清单：**
- [ ] `ufw status verbose` 看一眼，有没有你不认识的 ALLOW 规则
- [ ] 云厂商控制台（阿里云/腾讯云/AWS）的安全组也要对照检查
  - 安全组和 UFW 要一致，有一个没关就等于没关
- [ ] 内部服务端口（8000、8005 这类）**绝对不要对外暴露**，只允许本机访问
  - 检查服务绑定地址：`ss -tlnp` 或 `netstat -tlnp`
  - `0.0.0.0:8005` = 所有网卡可达（危险）
  - `127.0.0.1:8005` = 只有本机可达（安全）

**快速扫一眼对外开了什么：**
```bash
ss -tlnp | grep -v 127.0.0.1
# 只应该看到 nginx/sshd，其他服务不应该出现在这里
```

---

## 三、Token 与密码：最容易踩的坑

**Token 安全原则：**
- [ ] **不同服务用不同 token**，不要所有接口共用一个
  - 一个 token 泄露 = 泄露一个接口，而不是全部
- [ ] Token 要足够随机（建议 32 位以上）
  ```bash
  openssl rand -base64 32
  ```
- [ ] **不要把 token 放在 URL 里**（`?token=xxx` 会被 nginx 日志记录）
  - 改用 HTTP Header：`X-Token: xxx` 或 `Authorization: Bearer xxx`
- [ ] **不要把 token 硬编码在代码里**，放到 `.env` 文件，`.env` 加进 `.gitignore`
  - 检查：`cat .gitignore | grep .env`
- [ ] 不定期去 GitHub 搜一下自己的仓库，看有没有意外提交进去的 key

**密码强度检查：**
- [ ] 所有网页登录密码是强随机密码
- [ ] 每个服务的密码不一样
- [ ] 改密码后在安全的地方记一份（密码管理器或纸质）

---

## 四、HTTPS：传输加密

**检查清单：**
- [ ] 所有对外接口走 HTTPS，没有裸 HTTP 暴露 token 或敏感数据
- [ ] 证书是否过期：
  ```bash
  echo | openssl s_client -connect 你的域名:443 2>/dev/null | openssl x509 -noout -dates
  ```
- [ ] Let's Encrypt 证书自动续签是否正常（一般 90 天，建议设提醒或检查 cron）
  ```bash
  certbot renew --dry-run
  ```
- [ ] nginx 是否关闭了老旧协议（TLS 1.0/1.1）

---

## 五、nginx 配置检查

nginx 是对外的"门面"，配置错了容易出意外。

**检查清单：**
- [ ] 没有多余的 `location` 块代理到不存在的服务（死路 = 404 不算危险，但说明配置没清理）
- [ ] 敏感接口有 token 验证（检查 `main.py` 或 `app.py` 对应接口）
- [ ] 有没有文件列目录功能（`autoindex on`）—— 应该关掉
- [ ] 上传接口有没有限制文件大小：`client_max_body_size`
- [ ] 是否有速率限制（`limit_req_zone`）防止暴力攻击
- [ ] **CORS 配置不能用通配符**（FastAPI / Express 等框架都有这个设置）                                                    
      危险写法：`allow_origins=["*"]` + `allow_origin_regex=".*"  → 任何网站都能借用户身份调你的 API                          
      正确写法：只写自己的域名 `allow_origins=["https://你的域名.com"]`                                                       
      检查方法：搜代码里的 `CORSMiddleware` 或 `cors`，看 `allow +_origins` 的值 

**查 nginx 访问日志里有没有可疑请求：**
```bash
tail -100 /var/log/nginx/access.log | grep -E "40[134]|500"
# 大量 401/403 = 有人在试 token
# 大量 404 = 有人在扫路径
```

---

## 六、服务与进程检查

**检查清单：**
- [ ] 列出所有在跑的服务，认识每一个：
  ```bash
  systemctl list-units --type=service --state=running
  ```
- [ ] 有没有你没装的陌生进程（可能是被入侵留下的）：
  ```bash
  ps aux | sort -k3 -rn | head -20   # 按 CPU 排序
  ps aux | sort -k4 -rn | head -20   # 按内存排序
  ```
- [ ] 检查定时任务（cron）里有没有陌生条目：
  ```bash
  crontab -l
  ls /etc/cron* /var/spool/cron/
  ```
- [ ] 服务器负载异常高时（CPU 持续 90%+），优先排查是否被挖矿

---

## 七、日志快速审查

日志是发现问题最直接的地方，不需要每条都看，重点找异常。

```bash
# SSH 失败登录
grep "Failed password\|Invalid user" /var/log/auth.log | wc -l

# nginx 异常请求
grep " 401 \| 403 \| 500 " /var/log/nginx/access.log | tail -30

# 系统错误
journalctl -p err --since "1 week ago" | tail -50
```

**发现大量 401/403：**
- 来自同一 IP → 可以用 UFW 封掉：`ufw deny from xxx.xxx.xxx.xxx`
- 来自不同 IP → 说明 token/接口路径可能泄露了，考虑换 token

---

## 八、数据备份

安全不只是防入侵，还要防丢失。

- [ ] 重要数据（SQLite 数据库、配置文件、记忆文件）定期备份到本地或异地
- [ ] 备份文件不放在同一台服务器上
- [ ] 定期测试恢复（备份文件损坏是最常见的悲剧）

---

## 九、快速体检脚本

把以下命令存下来，每次检查跑一遍：

```bash
echo "=== UFW 状态 ===" && ufw status verbose
echo "=== 对外监听的端口 ===" && ss -tlnp | grep -v 127.0.0.1
echo "=== 最近 SSH 失败 ===" && grep "Failed" /var/log/auth.log 2>/dev/null | wc -l && echo "次失败登录"
echo "=== 磁盘使用 ===" && df -h /
echo "=== 内存使用 ===" && free -h
echo "=== 证书到期时间 ===" && echo | openssl s_client -connect 你的域名:443 2>/dev/null | openssl x509 -noout -enddate
```

---

## 十、出事了怎么办

**发现可疑入侵：**
1. 立刻快照/备份现场（云控制台创建快照）
2. 在云控制台安全组直接封所有入站（包括 SSH）——先断网再调查
3. 换所有 token 和密码
4. 检查 `/etc/passwd`、`~/.ssh/authorized_keys`、crontab 有无陌生条目
5. 从可信备份重建，不要复用被污染的环境

**常用排查命令：**
```bash
last -20                           # 最近登录记录
who                                # 当前在线用户
find / -newer /tmp -type f 2>/dev/null | head -20   # 最近被修改的文件
```

---

> 安全不是一次性的事，是习惯。改动了什么、发现了什么，记下来，下次检查时对比。
> 草木皆兵不是坏事，是负责任。

---

## 附：Stack-chan 硬件设备专项检查

如果你也是stack要走服务器而不是直接连你的电脑，请对照：

> 适用于用 ESP32 系列硬件（Stack-chan、M5Stack 等）与服务器通信的情况。
> 硬件设备有一些纯软件项目没有的特殊风险——token 烧进固件、无法随时更新、在局域网里裸跑——需要单独注意。

### 硬件端（固件）

**Token 安全：**
- [ ] **不要把 token 放在 URL 里**
  - 固件里常见的写法：`POST https://your-server.com/api/xxx?token=abcdef`
  - 这个 URL 会完整出现在 nginx 日志里，换 token 后旧值还在日志里躺着
  - 正确做法：改用 HTTP Header 传 token
    ```cpp
    // config.h 里只存 token 值，不拼进 URL
    
    // audio_bridge.cpp 里加 Header
    http.addHeader("X-Token", TOKEN);
    // 而不是 url += "?token=" + TOKEN
    ```

- [ ] **WebSocket 鉴权用首帧，不用 URL param**
  - `ws://server/api/bridge?token=xxx` → token 明文在 URL 和日志里
  - 改成连接建立后发第一帧：`{"type":"auth","token":"xxx"}`
  - 服务器端验证首帧，验证失败直接断连

- [ ] **config.h 里的 token 和服务器侧的 token 名称不要一样**
  - 固件是可以被提取和反编译的，token 名称越不明显越好
  - 比如服务器用 `STACK_AUTH_TOKEN`，固件里叫 `DEVICE_KEY`，不要都叫 `TOKEN`

**WiFi 凭证：**
- [ ] **固件里的 WiFi 密码不要提交到 GitHub**
  - 如果用平台 build（PlatformIO / Arduino IDE），确认 `config.h` 在 `.gitignore` 里
  - 不小心提交了：立刻改 WiFi 密码，git history 里的旧密码就当泄露处理

**固件更新：**
- [ ] 换 token 后记得重新烧录固件，旧 token 在硬件里会一直有效直到刷掉
- [ ] 每次换服务器 token，同步更新固件 config.h 并重烧

---

### 服务器端（接收硬件请求的接口）

- [ ] **限制文件大小**：硬件上传音频（WAV/MP3）的接口要设 `client_max_body_size`，防止被塞大文件

- [ ] **加速率限制**：硬件接口不需要太高的频率，每分钟 10-20 次就够
  - 防止设备被控制后狂轰接口，或者 token 泄露后被外部滥用

- [ ] **验证来源合理性**（可选，进阶）：
  - 硬件通常是局域网内设备，可以在服务器端检查 `X-Forwarded-For` 或请求来源 IP
  - 来自奇怪 IP 的 Stack-chan 请求可以直接拒绝

---

### 局域网安全

Stack-chan 绑着一个**固定局域网 IP**（一般是路由器分配的静态地址），这意味着：

- 任何连上你家 WiFi 的设备，都能直接访问它的 HTTP 接口（如果它对外开了的话）
- Stack-chan 本身一般没有鉴权，谁都能发命令给它

**建议：**
- [ ] 检查 Stack-chan 固件里有没有对外暴露管理接口（`/settings`、`/ota` 等），有的话确认有密码保护
- [ ] 路由器开启 AP 隔离（客户端之间不能互相通信），可以防止陌生设备访问 Stack-chan
- [ ] 不把 Stack-chan 的 IP 和接口路径透露给不信任的人

---

*最后更新：2026-06*
