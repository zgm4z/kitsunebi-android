# Kitsunebi

A fully-featured V2Ray client for Android.

## 下载

<a href="https://play.google.com/store/apps/details?id=fun.kitsunebi.kitsunebi4android"><img src="https://play.google.com/intl/en_us/badges/images/generic/en-play-badge.png" height="100"></a>

## 负载均衡策略
Kitsunebi 使用的 Core 扩展了 v2ray-core 的功能，新增根据节点延迟值来选择最快速节点的策略，图形界面上可以添加节点组来开启，使用自定义配置的话，有以下配置项，所有时间数值单位为秒：
```json
{
    "tag": "proxy",
    "selector": [
        "primary_proxy",
        "backup_proxy"
    ],
    "strategy": "latency",
    "interval": 60, // 每次测速之间的最少时间间隔
    "totalMeasures": 3, // 每次测速中对每个 outbound 所做的请求次数
    "delay": 1, // 每个测速请求之间的时间间隔
    "timeout": 4, // 测速请求的超时时间
    "probeTarget": "tcp:www.google.com:80", // 测速请求发送的目的地
    "probeContent": "HEAD / HTTP/1.1\r\n\r\n" // 测速请求内容
}
```

## 规则集
规则集目前支持以下配置项：
- Rule
  - DOMAIN-KEYWORD
  - DOMAIN-SUFFIX
  - DOMAIN-FULL
  - IP-CIDR
  - PORT
  - GEOIP
  - FINAL
- RoutingDomainStrategy

更多关于规则集的示例及说明可以看这里：https://github.com/eycorsican/rule-sets

内置规则：
- `geosite` 规则取自：https://github.com/v2ray/domain-list-community
- `geoip` 规则为 MaxMind 的 GeoLite2，取自：https://github.com/v2ray/geoip

第三方规则：
- 兼容的第三方规则集，一般包含拦截广告、统计行为、隐私跟踪相关的规则：https://github.com/ConnersHua/Profiles

## 关于 DNS 处理
自 v1.0.0 起，默认的 DNS 处理方式为 Fake DNS，启用 Fake DNS 后，DNS 请求的流量几乎不会被传进 V2Ray，所以 V2Ray 的 `内置 DNS` 和 `DNS outbound` 配置不会起太大作用；当 Fake DNS 处于禁用状态，DNS 请求的流量会以正常 UDP 流量的形式进入 V2Ray，这时你可以使用 inbound tag 在路由中配置路由规则来识别出相应 DNS 流量，从而转发给 `DNS outbound`，从而让 V2Ray 的 `内置 DNS` 来处理（看下面配置示例）。如果使用自定义配置的同时开启 Fake DNS，则需要确保 freedom outbound 中的域名策略为 `非 AsIs`。

为什么启用了 Fake DNS 后，freedom outbound 一定要用 `非 AsIs` 策略呢？如果你不熟悉 `Fake DNS` 怎么工作，可以看看 [这篇文章](https://medium.com/@TachyonDevel/%E6%BC%AB%E8%B0%88%E5%90%84%E7%A7%8D%E9%BB%91%E7%A7%91%E6%8A%80%E5%BC%8F-dns-%E6%8A%80%E6%9C%AF%E5%9C%A8%E4%BB%A3%E7%90%86%E7%8E%AF%E5%A2%83%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8-62c50e58cbd0)。
启用 `Fake DNS` 后，本地的系统 DNS 缓存是被染污了的，如果 freedom outbound 用了 AsIs，对于那些非代理的域名请求，到了 freedom outbound 的时候，如果用系统 DNS 去解析（`AsIs` 策略），得到的 DNS 结果将会是被染污了不可用的 IP，会导致直连的请求发不出去。为了避免这个问题，方法就是让 freedom outbound 不使用系统的 DNS，也即不使用 `AsIs`，转而使用 V2Ray 的 `内置 DNS`（`UseIP` 策略）。

Fake DNS 跟 V2Ray 的 `流量探测` 在效果上非常相似，目的同样是要拿到请求的域名，但工作原理上有较大差异：
- Fake DNS
  - 适用于任何请求的流量
  - 对于走代理的请求，本地不会实际发出任何 DNS 请求流量（远程 DNS 解析）
  - 会染污本地 DNS 缓存，VPN 关掉后的短暂时间内，可能会导致网络请求异常
- `流量探测`
  - 只适用 http/tls 流量
  - 不能控制本地是否发出 DNS 请求流量（由 V2Ray 的 DNS 功能模块控制）

## 配置示例

```json
{
    "dns": {
        "clientIp": "115.239.211.92",
        "hosts": {
            "localhost": "127.0.0.1"
        },
        "servers": [
            "114.114.114.114",
            {
                "address": "8.8.8.8",
                "domains": [
                    "google",
                    "android",
                    "fbcdn",
                    "facebook",
                    "domain:fb.com",
                    "instagram",
                    "whatsapp",
                    "akamai",
                    "domain:line-scdn.net",
                    "domain:line.me",
                    "domain:naver.jp"
                ],
                "port": 53
            }
        ]
    },
    "log": {
        "loglevel": "warning"
    },
    "outbounds": [
        {
            "protocol": "vmess",
            "settings": {
                "vnext": [
                    {
                        "address": "1.2.3.4",
                        "port": 10086,
                        "users": [
                            {
                                "id": "0e8575fb-a71f-455b-877f-b74e19d3f495"
                            }
                        ]
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp"
            },
            "tag": "proxy"
        },
        {
            "protocol": "freedom",
            "settings": {
                "domainStrategy": "UseIP"
            },
            "streamSettings": {},
            "tag": "direct"
        },
        {
            "protocol": "blackhole",
            "settings": {},
            "tag": "block"
        },
	{
	    "protocol": "dns",
	    "tag": "dns-out"
	}
    ],
    "policy": {
        "levels": {
            "0": {
                "bufferSize": 4096,
                "connIdle": 30,
                "downlinkOnly": 0,
                "handshake": 4,
                "uplinkOnly": 0
            }
        }
    },
    "routing": {
        "domainStrategy": "IPIfNonMatch",
        "rules": [
            {
                "inboundTag": ["tun2socks"],
                "network": "udp",
                "port": 53,
                "outboundTag": "dns-out",
                "type": "field"
            },
            {
                "domain": [
                    "domain:setup.icloud.com"
                ],
                "outboundTag": "proxy",
                "type": "field"
            },
            {
                "ip": [
                    "8.8.8.8/32",
                    "8.8.4.4/32",
                    "1.1.1.1/32",
                    "1.0.0.1/32",
                    "9.9.9.9/32",
                    "149.112.112.112/32",
                    "208.67.222.222/32",
                    "208.67.220.220/32"
                ],
                "outboundTag": "proxy",
                "type": "field"
            },
            {
                "ip": [
                    "geoip:cn",
                    "geoip:private"
                ],
                "outboundTag": "direct",
                "type": "field"
            },
            {
                "outboundTag": "direct",
                "port": "123",
                "type": "field"
            },
            {
                "domain": [
                    "domain:pstatp.com",
                    "domain:snssdk.com",
                    "domain:toutiao.com",
                    "domain:ixigua.com",
                    "domain:apple.com",
                    "domain:crashlytics.com",
                    "domain:icloud.com",
                    "cctv",
                    "umeng",
                    "domain:weico.cc",
                    "domain:jd.com",
                    "domain:360buy.com",
                    "domain:360buyimg.com",
                    "domain:douyu.tv",
                    "domain:douyu.com",
                    "domain:douyucdn.cn",
                    "geosite:cn"
                ],
                "outboundTag": "direct",
                "type": "field"
            },
            {
                "ip": [
                    "149.154.167.0/24",
                    "149.154.175.0/24",
                    "91.108.56.0/24",
                    "125.209.222.0/24"
                ],
                "outboundTag": "proxy",
                "type": "field"
            },
            {
                "domain": [
                    "twitter",
                    "domain:twimg.com",
                    "domain:t.co",
                    "google",
                    "domain:ggpht.com",
                    "domain:gstatic.com",
                    "domain:youtube.com",
                    "domain:ytimg.com",
                    "pixiv",
                    "domain:pximg.net",
                    "tumblr",
                    "instagram",
                    "domain:line-scdn.net",
                    "domain:line.me",
                    "domain:naver.jp",
                    "domain:facebook.com",
                    "domain:fbcdn.net",
                    "pinterest",
                    "github",
                    "dropbox",
                    "netflix",
                    "domain:medium.com",
                    "domain:fivecdm.com"
                ],
                "outboundTag": "proxy",
                "type": "field"
            }
        ],
        "strategy": "rules"
    }
}
```
