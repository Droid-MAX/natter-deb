# Natter
帮助 Full cone NAT (NAT 1) 用户打开公网 TCP 端口。  
<p align="right">
交流群组：<a href="https://jq.qq.com/?_wv=1027&k=EYXohGpC">Q 657590400</a> | <a href="https://t.me/+VS5sjOWGgzsyYjY1">TG</a>
</p>

**当前版本：** v0.9.0  
**注意：目前的 Natter 仍处于开发初期阶段。** Natter 的参数列表和行为将来可能会发生改变。

## 使用例
首次使用，检查当前网络 NAT 情况：
```
python natter.py --check-nat
```
![](.img/img01_1.png)
如果没有告警，那么您的网络一切正常。如有报错，请参考下文“错误解读”部分。

在本地 3456 号 TCP 端口上实行 TCP 打洞，并开启测试用 HTTP 服务：
```
python natter.py -t 3456
```
![](.img/img01_2.png)

使用外部网络访问该公网地址 `http://203.0.113.10:14500/`，如果看见 `It works!` 字样，则为打洞成功：
![](.img/img02.png)

打洞测试成功后，可以去掉 `-t` 选项，然后将 3456 端口转发至您想要的目标地址上。

## 转发方法
您可以这样设置端口转发：

在 OpenWRT 网页端中「网络」 - 「防火墙」 - 「端口转发」，填写以下信息：

 协议 | 外部端口 | 内部 IP 地址  | 内部端口
------|----------|---------------|----------
 TCP  | 3456     | 192.168.1.100 | 443

此时，我们在 OpenWRT 上使用 Natter 在 `3456` 号端口进行打洞，即可向外网暴露 `192.168.1.100:443` 。
```
python natter.py 3456
```

## 使用配置文件
如果您不想手动设置端口转发，可以交由 Natter 处理。同时，使用配置文件，Natter 可以提供更多有用的功能。
```
python natter.py -c ./natter-config.json
```
配置文件的说明如下：
```javascript
// 注意：JSON 配置文件不支持代码注释，此处为说明配置用途。
{
    "logging": {
        "level": "info",                        // 日志等级：可选值："debug"、"info"、"warning"、"error"
        "log_file": "./natter.log"              // 将日志输出到指定文件，不需要请留空：""
    },
    "status_report": {
        // 当外部IP/端口发生改变时，会执行下方命令。
        // 大括号 {...} 为占位符，命令执行时会被实际值替换。
        // 不需要请留空：""
        "hook": "bash ./natter-hook.sh '{protocol}' '{inner_ip}' '{inner_port}' '{outer_ip}' '{outer_port}'",
        "status_file": "./natter-status.json"   // 将实时端口映射状态储存至指定文件，不需要请留空：""
    },
    "open_port": {
        // 此处设置 Natter 打洞IP:端口。（仅打洞）
        // 此处地址为 Natter 绑定（监听）的地址，Natter 仅对这些地址打洞，您需要手动设置端口转发。
        // 注意：使用默认出口IP，请使用 0.0.0.0 ，而不是 127.0.0.1 。
        "tcp": [
            "0.0.0.0:3456",
            "0.0.0.0:3457"
        ],
        "udp": [
            "0.0.0.0:3456",
            "0.0.0.0:3457"
        ]
    },
    "forward_port": {
        // 此处设置需要 Natter 开放至公网的 IP:端口。（打洞 + 内置转发）
        // Natter 会全自动打洞、转发，您无需做任何干预。
        // 注意：使用本机IP，请使用 127.0.0.1，而不是 0.0.0.0 。
        "tcp": [
            "127.0.0.1:80",
            "192.168.1.100:443"
        ],
        "udp": [
            "127.0.0.1:53",
            "192.168.1.100:51820"
        ]
    },
    "stun_server": {
        // 此处设置公共 STUN 服务器。
        // TCP 服务器请确保 TCP/3478 端口开放可用；
        // UDP 服务器请确保 UDP/3478 端口开放可用。
        "tcp": [
            "stun.stunprotocol.org",
            "stun.voip.blackberry.com"
        ],
        "udp": [
            "stun.miwifi.com",
            "stun.qq.com"
        ]
    },
    "keep_alive": "www.qq.com"  // 此处设置 HTTP Keep-Alive 服务器。请确保该服务器 80 端口开放，且支持 HTTP Keep-Alive。
}
```

## 原理图
![](.img/img03.png)


## 方案

- **推荐方案：**  
    光猫设置桥接模式，在路由器系统如 OpenWRT 上直接运行 Natter（仅经过一层 NAT）

- **可行方案：**  
    在子网中的主机上运行 Natter，在光猫或路由器上对其开启 DMZ 功能，或对需要开放的端口设置端口转发。（经过多层NAT）


## 大概率会失败的情形
- **不满足基本条件：**  
    经过测试，我的网络不是 NAT 1；

- **多层非可控 NAT：**  
    光猫处于路由模式，我无法关闭光猫的防火墙，并对其设置 DMZ 主机或改桥接；

- **运营商设置了防火墙：**  
    我在外部网络使用 `nmap` 对出口 IP 地址进行 TCP 全端口扫描，发现均为 `filtered` 。


## 更新日志
### v0.9.0
1. 打洞前不再强制检查 NAT 类型；
2. 新增 UDP 功能；
3. 新增内置端口转发功能；
4. 新增实时状态推送/更新功能；
5. 新增配置文件支持。


## 错误解读 & 解决方法

```
This OS or Python does not support reusing ports!
```
此操作系统或者 Python 不支持端口重用。  
**解决方法：** 推荐使用内核版本 4.0+ 的 Linux 系统。

```
No public STUN server is avaliable. Please check your Internet connection.
```
没有可用的公共 STUN 服务器。请检查您的网络连接。  
**解决方法：** 检查网络，检查防火墙是否阻止 TCP/UDP 端口 3478。如果是脚本内置列表中的服务器不可用，请提 issue。

```
You cannot perform TCP hole punching in a symmetric NAT network.
```
您无法在一个对称型 NAT 网络中实行 TCP 打洞。  
**解决方法：** 此网络无法打洞。检查您的网络拓扑。检查您是否处于多层 NAT 下。

> Q：Natter 给出了外部地址，但我没有办法访问。

**解决方法：** 检查您是否正确配置了端口转发。检查您的目标服务是否正常启动。尝试关闭光猫的防火墙。如果处于多层 NAT 下，请在光猫或路由器下设置 DMZ 主机或者端口转发。如果上述方案不能解决改问题，则是运营商设置了防火墙，此网络无法开放 TCP 端口。
