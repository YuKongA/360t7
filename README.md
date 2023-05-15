# Build OpenWrt for 360T7

### 360T7 折腾札记

#### 起因

---

OpenWrt 主线已经合并了对 MT7981 的支持，所以打算自己尝试编译一份。

#### 问题

---

因为咱也是刚接触 OpenWrt 不久，所知甚少，如果这些都是常识，请见谅：(

##### 其一

由于是我们是自行编译的 OpenWrt, 所以通常来说最先会遇到以下问题：

```
* satisfy_dependencies_for: Cannot satisfy the following dependencies for iptables-mod-extra:
 *      kernel (= 5.15.110-1-868e8f7ccd7266e72a24338f895c798b)
 * pkg_hash_check_unresolved: cannot find dependency kernel (= 5.15.110-1-868e8f7ccd7266e72a24338f895c798b) for kmod-tun
```
安装 komd 时需要验证当前系统内核的 MD5 是否与 SHAPSHOT 官方版本的一致。因为我们是自行编译，校验自然是通过不了的，也就导致我们安装不了。

关于这一问题，搜索得来的结果是 [解决Openwrt自编译版本内核不兼容问题](https://blog.csdn.net/weixin_36182568/article/details/116602462) ，对于该内容的可行性我并不清楚，因为该操作需要在编译之前完成，对于使用 GitHub workfolws 编译的我来说又要等一个多小时，所以我在操作完成等待编译完成的期间继续寻找其他解决办法。

很幸运的是在这期间我确实找到了其他的解决办法，灵感来自 [该贴#78楼](https://www.right.com.cn/forum/thread-8287712-6-2.html) 。根据此条回复，我们可以从 [该链接](https://downloads.openwrt.org/snapshots/targets/mediatek/filogic/packages) 下载到一个最新 SHAPSHOT 构建以内核版本命名的 ipk 软件包，以下是折腾的时候可以下到的版本：`kernel_5.15.111-1-868e8f7ccd7266e72a24338f895c798b_aarch64_cortex-a53.ipk`。

如果你编译的内核版本小于它，就可以直接安装这个 ipk ，将安装时的验证结果伪装成官方的 `5.15.111-1-868e8f7ccd7266e72a24338f895c798b` ，这样就可以正常安装 kmod 了。如果你编译的内核版本大于等于它，那么该方法无法适用于你，会提示无法降级。<s>我编译的时候刚巧 revert 了合并 5.15.111 的那个提交。</s>

操作完上述步骤之后，很荣幸我遇到了和 [该贴#80楼](https://www.right.com.cn/forum/thread-8287712-6-2.html) 回复的内容一样的问题。我们知道 kmod 其实就是 kernel module，也就是内核模块。既然 tun 启动不了，那么很大概率就是内核模块没成功加载，此时正常来讲我们只需要 force load 它，一般来说就行。

但我想知道为什么模块没有正常加载，在我的一番查找之后发现，自行安装的 kmod 在 openwrt 中是作为 overlay 运作的，位于 `/overlay/upper/lib/modules/`。这时候 `tun.ko` 位于`/overlay/upper/lib/modules/5.15.110/tun.ko`，其他 kmod 模块则位于 `/overlay/upper/lib/modules/5.15.111/` 目录下。那么咱的想法很明显，只需要把  `tun.ko` 移动到 `5.15.111` 目录下问题也就迎刃而解了，经过实测，事实也正是如此。

这样以来，自编译主线 OpenWrt 无法安装和启动 kmod 的问题就完全解决了。

##### 其二

由于所处环境原因，路由器需要设置 NAT6，在这之前我用的那些老版本 OpenWrt，都是 `firewall + iptables` 的组合，而主线现在换到了 `firewall4 + nftables`，如此一来之前用的 NAT6 设置方案就失效了。

在经过一段时间的摸索和查找资料之后，找到了 [小米4A千兆版V2刷自己编译的OpenWRT以及IPV6设置（包括中继与NAT6）](https://www.bilibili.com/read/cv23234832) ，按照里面的步骤成功设置了 NAT6。

不过完全按照其中步骤的话在我这边有点小问题，以下是我的操作步骤：

- 1.将 接口 >> LAN >> DHCPv6 服务器 >> IPv6 设置 里的 `RA 服务` 和 `DHCPv6 服务` 改为**服务器模式**，`NDP 代理` 改为**已禁用**（即安装 OpenWrt 时的默认状态），
- 2.将 接口 >> WAN >> 常规设置 里的 `请求指定长度的 IPv6 前缀` 改为**已禁用**，
- 3.将接口 >> 全局网络选项 里的 `IPv6 ULA 前缀` 的第一个字母 f 改成 d（例：`fdb5:4e54:d3bb::/48` -> `ddb5:4e54:d3bb::/48`），
- 4.使用终端输入 `vi /etc/hotplug.d/iface/99-ipv6` 写入以下内容保存，

```
#!/bin/sh

line=0
while [ $line -eq 0 ] ; do
    sleep 10
    line=`route -A inet6 | grep xxxx | awk 'END{print NR}'`
done
ip -6 r add default via `ip -6 route | grep "default from" | awk 'NR==1{print $5,$6,$7}'`

```

<b> 其中 xxxx 替换为你公网 IPv6 地址的前 8 位；具体看实际情况，如果经常有断电等情况，也可以只用前 4 位运营商前缀。</b>

- 5.使用终端输入 `vi /etc/sysctl.conf` ，添加下方内容保存，

```
net.ipv6.conf.wan.accept_ra=2
```

- 6.完成以上步骤后重启即可。

<b> 使用 nftables 请建议尽量移除 iptables 以保障兼容性，安装依赖相关功能的软件包时也注意使用 nftables 方案，例如 OpenClash。</b>

##### 其他

对于 OpenWrt 软件包较少的问题，可以自己添加一些软件源来补全，
```
src/gz immortalwrt_luci https://mirrors.pku.edu.cn/immortalwrt/snapshots/packages/aarch64_generic/luci
src/gz immortalwrt_packages https://mirrors.pku.edu.cn/immortalwrt/snapshots/packages/aarch64_generic/packages
src/gz immortalwrt_luci_a53 https://mirrors.pku.edu.cn/immortalwrt/snapshots/packages/aarch64_cortex-a53/luci
src/gz immortalwrt_packages_a53 https://mirrors.pku.edu.cn/immortalwrt/snapshots/packages/aarch64_cortex-a53/packages
```
或者直接下载对应 ipk 按照步骤进行安装，这都不是什么问题。

#### 最后

---

当前开源驱动下的 MT7981 编译的成品对我来说是可用状态，没发现什么异常的地方。
所处环境的宽带也只有 100Mbps，自然也是没有什么瓶颈存在了。

#### 感谢

---

以上全程非常感谢 @AngelaCooljx 的鼎力相助。
