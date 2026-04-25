# 1.前期准备
- 1.一台VPS
(PS：国内VPS特点延迟低，国外的宽带大，根据自己情况选择)
- 2.安装V2rayN
# 2.原理说明
UDP 53 是 DNS 查询常用端口，部分老旧的校园网系统连接后其实已经给你分配了网络IP，不过当你访问其他网站的时候会被拦截。但是为了能让你能够访问认证网站肯定不会拦截UDP53端口。那我们可以在VPS上部署UDP协议的代理，并且监听53端口，将本地的流量全部转换为UDP53的形式，校园网看见是UDP53就不会拦截。
## 补充
- 本人学校的校园网系统比较老旧甚至没有HTTPS，因此这个方案能够跑通，不同学校的校园网不同所有不保证通用性，但是可以用于思路借鉴
- 除了53端口外，其实还有很多特殊端口，比如67端口。如果你知道你访问的认证网站是什么端口也可以尝试。
# 3.检测可用性
- 在命令行窗口中用 `nslookup` 测试 DNS解析是否能用(在连接校园网但是未认证的状态)
  <img width="535" height="204" alt="PixPin_2026-04-25_14-25-58" src="https://github.com/user-attachments/assets/8588699b-c8b1-4a43-9230-a9c5a46bdb8b" />
  正常情况下应该会返回IP地址，如果显示如图上的情况，一般就说明可以绕过校园网认证
# 4.释放53端口
- 一般服务器的53端口被服务器的dns解析服务systemd-resolved占用，我们需要释放53端口
## 停用systemd-resolved 服务
- ```sudo -i```
  获得ROOT权限
- ```systemctl stop systemd-resolved```
## 编辑resolved.conf 文件
- ``` vi /etc/systemd/resolved.conf```
- 进入编辑后输入`i`插入和修改按下列注释修改文件，如果是国内的服务器，将dns=1.1.1.1改成223.5.5.5，国外的就1.1.1.1，修改完成后按Esc键，再输入英文的冒号: 输入` wq!` 回车保存文件
 ```
DNS=1.1.1.1  #取消注释，增加dns
#FallbackDNS=
#Domains=
#LLMNR=no
#MulticastDNS=no
#DNSSEC=no
#Cache=yes
DNSStubListener=no  #取消注释，把yes改为no
```
- ```ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf```配置文件
- 输入` lsof -i:53 `和 `netstat -ap | grep 53` 查看53端口占用情况，如下可以看到没有程序在使用53端口
  <img width="781" height="338" alt="PixPin_2026-04-25_14-43-43" src="https://github.com/user-attachments/assets/d1f7691e-8a70-4ec2-8b03-67a118caf991" />
# 5.开放UDP53防火墙
- `iptables -I INPUT -p UDP --dport 53 -j ACCEPT`
- 同时去你VPS的控制台上开放对应的端口，以Azure为例
  <img width="2550" height="1435" alt="PixPin_2026-04-25_14-46-57" src="https://github.com/user-attachments/assets/9c5321f0-b7e3-4d5f-b7a4-431aa552e04a" />
# 6.安装Hysteria 2
- ```bash <(curl -fsSL https://get.hy2.sh/)```
- 官方脚本支持 systemd，并会生成示例配置，但还需要手动改配置
```
nano /etc/hysteria/config.yaml
```
- 填入：
```
  listen: :53

auth:
  type: password
  password: 你的强密码

tls:
  type: self-signed
  ```
- 如果安装过程中提示是否需要设置域名选择NO,如果可以在安装过程中选择监听的端口请选择53
# 7.V2rayN配置

## 服务器
- 在V2ray中添加HY2服务器
  <img width="304" height="673" alt="PixPin_2026-04-25_14-52-31" src="https://github.com/user-attachments/assets/1ddd9bda-16dd-41c0-ab7b-24c99b093f17" />
- 地址:VPS的IP地址
- 端口：53
- 密码：你自己设置的密码
- 跳过证书验证:true
- core类型：sing_box(如果不能选择可以先忽略)
 <img width="1093" height="843" alt="PixPin_2026-04-25_14-54-22" src="https://github.com/user-attachments/assets/16681060-ba44-4cec-aa68-9301b67facc0" />
 
## 客户端
 - 在V2rayN中开启UDP，FAKE IP以及严格路由
   <img width="552" height="265" alt="PixPin_2026-04-25_14-59-29" src="https://github.com/user-attachments/assets/fe9b74c3-9a14-4b70-8421-f953ee29be7f" />
   <img width="733" height="455" alt="PixPin_2026-04-25_14-59-50" src="https://github.com/user-attachments/assets/7dd49d1d-8d75-4d24-9f2c-deab7e1d62e5" />
   <img width="1207" height="794" alt="PixPin_2026-04-25_15-00-07" src="https://github.com/user-attachments/assets/d6b65755-5ac1-4f1c-a5fb-b1bd097659ce" />
##  编辑DNS设置
```
{
  "servers": [
    {
      "tag": "remote",
      "address": "https://1.1.1.1/dns-query",
      "detour": "proxy"
    },
    {
      "tag": "local",
      "address": "223.5.5.5"
    }
  ],
  "rules": [
    {
      "outbound": "any",
      "server": "remote"
    }
  ]
}
```
<img width="739" height="740" alt="PixPin_2026-04-25_15-03-05" src="https://github.com/user-attachments/assets/95da7c34-9928-4250-8fa0-d349b265e8ca" />

## 编辑规则
-在路由设置中添加一条全局规则
<img width="1112" height="380" alt="PixPin_2026-04-25_15-05-48" src="https://github.com/user-attachments/assets/f2ce47b2-76c5-4c3b-99d7-71f2e8f6f55e" />

- outboundTag:direct
- port: 53
- ip: 填入你VPS的IP
  <img width="1099" height="518" alt="PixPin_2026-04-25_15-07-50" src="https://github.com/user-attachments/assets/4a1552f5-15a8-4f41-ba22-2accd6a5fb26" />

# 测试
- 连接你的校园网但是不认证
- 选择全局路由（一定是你添加了规则的那个全局路由），连接系统代理或者开启TUN模式
  <img width="771" height="46" alt="PixPin_2026-04-25_15-08-23" src="https://github.com/user-attachments/assets/aee02606-a488-4501-9717-c032733a0288" />
- 访问任意网站测试

## PS
- 经过测试，TUN模式以及系统代理可能存在冲突，也可能不存在。有的时候开启TUN模式无法访问网站，关闭TUN模型启用系统代理不能登录QQ和微信。但是又因为前面添加了规则好像又无所谓。总之，出现网络异常可以把这两个开关排列组合

# END








  


