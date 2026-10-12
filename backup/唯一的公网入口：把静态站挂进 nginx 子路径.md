我有一台服务器，上面跑着一个 WordPress 站点。我想把另一件东西——一个 Vite 构建的静态站点——挂到同一个域名下，用一个子路径 `/line/` 访问。

听起来是五分钟的事。实际上我改了四遍配置，重启了三次 nginx，最后发现问题根本不在 nginx 上。

先把结论放在前面：**这台机器是 NAT 共享公网 IP，只有一个端口能从外面进来。所有配置都必须围绕这个事实来写。**

## 一、先搞清楚外面能看见哪些端口

服务器环境：Debian 12 + 宝塔面板 12.0.0 + nginx 1.28.3。

我以为登录用的 SSH 端口（49683）就是服务商分给我的那一组端口之一，其他端口应该也能通。测完发现完全不是这样：

| 公网端口 | 实际落到 | 说明 |
| --- | --- | --- |
| 49683 | 内网 22 | SSH |
| **57914** | **内网 80** | 唯一的公网 Web 入口 |
| 80 / 443 / 888 / 8888 / 29436 / 39000-40000 | 无映射 | 公网完全不通 |
| 8080 / 2052 / 8443 / 22 / 3306 | **别人的机器** | 同一公网 IP 下的其他租户 |

最后一行是最要命的。这台机器不能监听 8080——公网上 8080 指向的是同 IP 下另一个租户。我如果按直觉写「应用监听 8080，防火墙放行」，外面看到的会是别人的服务。

**探测方法**：起一组监听器，每个端口返回带端口号的标记，然后从公网逐个访问，看谁回应。

```python
#!/usr/bin/env python3
# 每个端口回一个带端口号的标记，用来确认公网端口到底落到内网哪个端口
import socket, sys, threading

PORTS = [int(x) for x in sys.argv[1:]] or [29436, 39000, 39001, 39002, 8888, 888]

def serve(port):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    try:
        s.bind(("0.0.0.0", port))
    except Exception as e:
        print("bind %d failed: %s" % (port, e), flush=True)
        return
    s.listen(16)
    print("listening %d" % port, flush=True)
    while True:
        c, a = s.accept()
        body = ("MARKER-PORT-%d\n" % port).encode()
        c.sendall(b"HTTP/1.0 200 OK\r\nContent-Type: text/plain\r\nContent-Length: "
                  + str(len(body)).encode() + b"\r\nConnection: close\r\n\r\n" + body)
        c.close()

for p in PORTS:
    threading.Thread(target=serve, args=(p,), daemon=True).start()
threading.Event().wait()
```

这个方法比端口扫描快得多，因为**不用猜**：你监听哪个端口，标记里就写着哪个端口号，从外面收到什么标记，就知道映射关系是什么。

这台机器还测出一条额外结论：**没有公网 IPv6，也无法主动连到 Cloudflare 或 GitHub**（出网只通阿里云、npmmirror 这类国内源）。所以 Cloudflare Tunnel、CDN 回源这些方案在这里一律用不了。

## 二、于是只能挂在子路径上

唯一的入口是「公网 57914 → 内网 80」，而内网 80 上的默认站点是我原来的 WordPress。我不想为了一个新页面把博客挪走，所以选择把作品挂在 `/line/` 子路径下，WordPress 完全不受影响。

对应地，构建产物的 `base` 必须跟着改：

```bash
cd /www/wwwroot/unbroken-line
pnpm exec vite build --base=/line/ --outDir dist-line
```

`--outDir dist-line` 是故意的：根路径版本（`base=/`，输出到 `dist/`）保留着，以后如果拿到独立端口，直接切过去就行。

## 三、宝塔面板下的扩展配置：只能写 location

宝塔的 nginx 站点配置支持 `extension` 目录，每个站点一个子目录，里面的 `.conf` 会被 `include` 进该站点的 `server {}` 里：

```
/www/server/panel/vhost/nginx/extension/<站点名>/*.conf
```

**关键限制：这些文件里只能写 `location`，不能写 `server {}`。** 我第一版不信邪，在里面写了一个 `server {}`，nginx 直接报 duplicate default server 起不来。

最终配置：

```nginx
# 挂载在现有站点下的子路径 /line/
# 构建产物 base=/line/，目录 /www/wwwroot/unbroken-line/dist-line/
# 公网入口： http://<公网IP>:57914/line/

location = /line {
    # 用 $http_host 保留端口，否则会跳到公网不通的 80 端口
    return 301 $scheme://$http_host/line/;
}

location ^~ /line/assets/ {
    alias /www/wwwroot/unbroken-line/dist-line/assets/;
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable";
    access_log off;
}

location ^~ /line/fonts/ {
    alias /www/wwwroot/unbroken-line/dist-line/fonts/;
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
    access_log off;
}

location ^~ /line/ {
    alias /www/wwwroot/unbroken-line/dist-line/;
    index index.html;
    autoindex off;
    access_log off;
}
```

三个细节值得单独说：

**1. `$http_host` 而不是 `$host`。**
`/line` 少了结尾斜杠时要跳转到 `/line/`。我第一版写的是 `return 301 $scheme://$host/line/;`——`$host` 是纯主机名，不带端口，于是浏览器被跳到 `http://<IP>/line/`，也就是 **80 端口**。而这台机器公网 80 是不通的，页面直接打不开。

用 `$http_host` 才保留住 `:57914`。这个 bug 的隐蔽之处在于：如果你在本机用 `curl http://127.0.0.1/line` 测，它是**正常的**——因为本机 80 确实通。只有从公网访问才会暴露。所以我最后写了专门的验收脚本，全部带 `Host` 头模拟公网请求：

```bash
curl -s -o /dev/null -D- -H 'Host: <公网IP>:57914' http://127.0.0.1:80/line \
  | grep -iE '^(HTTP|Location)'
# 期望：301 且 Location 里带 :57914
```

**2. `alias` 必须写在 `location ^~ /line/` 里，而不是 `root`。**
`alias` 会把 location 匹配到的前缀**替换**掉；`root` 是**拼接**。用 `root /www/wwwroot/unbroken-line/dist-line;` 会去找 `dist-line/line/index.html`，404。

**3. `^~` 前缀匹配的优先级。**
`assets/` 和 `fonts/` 用了更长的前缀，所以它们会先命中，拿到长缓存；其余请求落到 `/line/` 那条。

## 四、这台机器的另外两个坑

**npm 源不通。** 服务器国内直连 `registry.npmjs.org` 超时，装依赖必须指定镜像：

```bash
npm i -g pnpm --registry=https://registry.npmmirror.com
pnpm install --registry=https://registry.npmmirror.com
```

**磁盘 I/O 极慢。** apt 解包一个 10MB 的包要七分钟以上。在服务器上跑构建会等到怀疑人生，所以最终流程改成：**本地构建好，只上传 `dist-line/` 目录。** 服务器上只放产物，不装 Node 工具链。

## 五、验收

配置改完不需要重启任何服务（nginx 直接读文件），但改配置本身要先检查再平滑重载：

```bash
/www/server/nginx/sbin/nginx -t && /www/server/nginx/sbin/nginx -s reload
```

验收脚本跑三件事：作品页能开、静态资源能开、**WordPress 没被搞坏**。

```powershell
Write-Output "===== 公网验收 ====="
$r = Invoke-WebRequest -Uri 'http://<公网IP>:57914/line/' -UseBasicParsing -TimeoutSec 25 `
     -Headers @{ 'User-Agent' = 'Mozilla/5.0' }
Write-Output ("首页  OK  {0}  {1} bytes" -f $r.StatusCode, $r.RawContentLength)

# 哈希文件名的资源，验证 alias 和缓存头
foreach ($u in @(
  'http://<公网IP>:57914/line/assets/index-Bz4Gf8fi.js',
  'http://<公网IP>:57914/line/assets/style-CwnDITju.css',
  'http://<公网IP>:57914/line/fonts/newsreader-normal-latin.woff2'
)) {
  $h = Invoke-WebRequest -Uri $u -Method Head -UseBasicParsing -TimeoutSec 25 `
       -Headers @{ 'User-Agent' = 'Mozilla/5.0' }
  Write-Output ("OK  {0}  {1}  {2}" -f $h.StatusCode, $h.Headers['Content-Type'], $u)
}

Write-Output "===== 你的 WordPress ====="
$b = Invoke-WebRequest -Uri 'http://<公网IP>:57914/' -UseBasicParsing -TimeoutSec 25 `
     -Headers @{ 'User-Agent' = 'Mozilla/5.0' }
Write-Output ("博客首页  OK  {0}" -f $b.StatusCode)
```

最后一行是我最在意的。加子路径这件事的风险从来不是「新页面打不开」——那个一眼就能发现。风险是**把原来好好的站点弄坏了，而你没注意**。

## 六、怎么回滚

写在部署文档里的回滚步骤，全部实测过：

```bash
# 1) 去掉子路径挂载（WordPress 立刻恢复原样）
rm -f /www/server/panel/vhost/nginx/extension/*/unbroken-line.conf
/www/server/nginx/sbin/nginx -t && /www/server/nginx/sbin/nginx -s reload

# 2) 去掉内网独立站点
rm -f /www/server/panel/vhost/nginx/unbroken-line.conf
/www/server/nginx/sbin/nginx -s reload

# 3) 删代码（可选）
rm -rf /www/wwwroot/unbroken-line
```

---

## 回头看

真正的教训只有一条：**先花时间搞清楚网络拓扑，再动手写配置。**

我在 nginx 配置上反复折腾的三轮，回溯起来都源于一开始的假设——「这台机器有很多端口可以用」。这个假设错了，之后的每一个决定都建在沙子上。花二十分钟写个端口探测脚本，能省掉后面两小时的瞎猜。

至于 TLS：这台机器出网到不了 Let's Encrypt，所以目前是 HTTP。想上 HTTPS，得等域名真正解析到这台机器、并且有一个能用的 ACME 通道，那是另一件事了。
