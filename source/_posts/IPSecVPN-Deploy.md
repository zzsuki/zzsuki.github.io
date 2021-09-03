---
title: IPSecVPN服务搭建
date: 2021-09-02 11:05:40
tags: [VPN, IPSec]
---
# IPSecVPN服务搭建

## 服务器配置

### 使用docker部署

1. 部署docker
2. 拉取自动化部署镜像：docker pull hwdsl2/ipsec-vpn-server
3. 设置环境变量文件`.vpn.env`：

   ```shell
    # 鉴别信息相关
    VPN_IPSEC_PSK=your_ipsec_pre_shared_key
    VPN_USER=your_vpn_username
    VPN_PASSWORD=your_vpn_password
    # 配置其它账号(存在多终端时建议分开配置账号)
    VPN_ADDL_USERS=additional_username_1 additional_username_2
    VPN_ADDL_PASSWORDS=additional_password_1 additional_password_2
    # VPN服务器使用的nameserver，默认使用google的dns服务即8.8.8.8
    VPN_DNS_SRV1=1.1.1.1
    VPN_DNS_SRV2=1.0.0.1
    # VPN服务器的FQDN值，会影响到IKEv2模式生成的证书(hostname -f 可查看FQDN)
    VPN_DNS_NAME=vpn.example.com
    # 指定client名称，类似一个tag一样，客户端配置以name为单位进行管理，默认vpnclient
    VPN_CLIENT_NAME=your_client_name
   ```

4. 运行IPSec VPN服务器： `docker run --name ipsec-vpn-server --env-file .vpn.env --restart always -v ikev2-vpn-data:/etc/ipsec.d -p 500:500/udp -p 4500:4500/udp -d --privileged hwdsl2/ipsec-vpn-server`
   - `-v`参数含义：自动创建volume挂载文件，使用-v时容器会自动启动IKEv2
   - 支持的协议： IPsec/L2TP、IKEv2、IPsec/XAuth ("Cisco IPsec")
   - 关于privileged：如果考虑到安全，不希望给容器提供整体的超级权限；可以通过单独给容器设置必要的权限来控制(但可能导致一些未知的错误)，将`--privileged`改为：

        ```shell
        --cap-add=NET_ADMIN \
        --device=/dev/ppp \
        --sysctl net.ipv4.ip_forward=1 \
        --sysctl net.ipv4.conf.all.accept_redirects=0 \
        --sysctl net.ipv4.conf.all.send_redirects=0 \
        --sysctl net.ipv4.conf.all.rp_filter=0 \
        --sysctl net.ipv4.conf.default.accept_redirects=0 \
        --sysctl net.ipv4.conf.default.send_redirects=0 \
        --sysctl net.ipv4.conf.default.rp_filter=0 \
        --sysctl net.ipv4.conf.eth0.send_redirects=0 \
        --sysctl net.ipv4.conf.eth0.rp_filter=0
        ```

### 服务配置常用

1. 获取VPN登录信息： `docker logs ipsec-vpn-server`
2. IKEv2配置：
   - 添加客户端：`docker exec -it ipsec-vpn-server ikev2.sh --addclient [client name]`
   - 导出已有客户端的配置：`docker exec -it ipsec-vpn-server ikev2.sh --exportclient [client name]`
   - 列出已有客户端的名称：`docker exec -it ipsec-vpn-server ikev2.sh --listclients`\
   - 显示使用信息：`docker exec -it ipsec-vpn-server ikev2.sh -h`
   - 复制配置文件到宿主机（内容可用于客户端配置）：`docker cp ipsec-vpn-server:/etc/ipsec.d/vpnclient.p12 ./`

## 客户端配置

### IPsec/L2TP VPN

#### win10

- 图形界面操作：
  1. 右键单击系统托盘中的无线/网络图标。
  2. 选择**打开网络和共享中心**。或者，如果你使用 Windows 10 版本 1709 或以上，选择**打开"网络和Internet"设置**，然后在打开的页面中单击**网络和共享中心**。
  3. 单击**设置新的连接或网络**。
  4. 选择**连接到工作区**，然后单击**下一步**。
  5. 单击**使用我的Internet连接 (VPN)**。
  6. 在**Internet地址**字段中输入`你的VPN 服务器 IP`。
  7. 在**目标名称**字段中输入任意内容。单击**创建**。
  8. 返回**网络和共享中心**。单击左侧的**更改适配器设置**。
  9. 右键单击新创建的 VPN 连接，并选择**属性**。
  10. 单击 **安全** 选项卡，从 **VPN 类型** 下拉菜单中选择 "使用 IPsec 的第 2 层隧道协议 (L2TP/IPSec)"。
  11. 单击 **允许使用这些协议**。选中 "质询握手身份验证协议 (CHAP)" 和 "Microsoft CHAP 版本 2 (MS-CHAP v2)" 复选框。
  12. 单击 **高级设置** 按钮。
  13. 单击 **使用预共享密钥作身份验证** 并在 **密钥** 字段中输入`你的 VPN IPsec PSK`。
  14. 单击 **确定** 关闭 **高级设置**。
  15. 单击 **确定** 保存 VPN 连接的详细信息。
- 命令行执行：
  
  ```cmd
    # 不保存命令行历史记录
    Set-PSReadlineOption –HistorySaveStyle SaveNothing
    # 创建 VPN 连接
    Add-VpnConnection -Name 'My IPsec VPN' -ServerAddress '你的 VPN 服务器 IP' -L2tpPsk '你的 VPN IPsec PSK' -TunnelType L2tp -EncryptionLevel Required -AuthenticationMethod Chap,MSChapv2 -Force -RememberCredential -PassThr
  ```

- **注意**
  - 首次使用IPsec/L2TP 模式连接到 VPN 时，需要修改注册表以解决 VPN 服务器 和/或 客户端与 NAT （比如家用路由器）的兼容问题：`REG ADD HKLM\SYSTEM\CurrentControlSet\Services\PolicyAgent /v AssumeUDPEncapsulationContextOnSendRule /t REG_DWORD /d 0x2 /f`
  - 某些个别的 Windows 系统配置禁用了 IPsec 加密，此时也会导致连接失败。要重新启用它，可以运行以下命令并重启：`REG ADD HKLM\SYSTEM\CurrentControlSet\Services\RasMan\Parameters /v ProhibitIpSec /t REG_DWORD /d 0x0 /f`

#### iOS

1. 进入**设置** -> **通用** -> **VPN**。
2. 单击 **添加VPN配置**...。
3. 单击 **类型** 。选择 **L2TP** 并返回。
4. 在 **描述** 字段中输入任意内容。
5. 在 **服务器** 字段中输入`你的 VPN 服务器 IP`。
6. 在 **帐户** 字段中输入`你的 VPN 用户名`。
7. 在 **密码** 字段中输入`你的 VPN 密码`。
8. 在 **密钥** 字段中输入`你的 VPN IPsec PSK`。
9. 启用 **发送所有流量** 选项。
10. 单击右上角的 **完成**。
11. 启用 VPN 连接。

#### OS X

基础配置类似，在基础配置完成后需要单独注意的包括：

- 单击 **高级** 按钮，并选中 **通过VPN连接发送所有通信** 复选框
- 单击 **TCP/IP** 选项卡，并在 **配置IPv6** 部分中选择 **仅本地链接**

### IKEv2 VPN（推荐方式）

#### Win10(GUI配置不推荐， 不如命令行快捷)

1. 导入证书文件：`certutil -f -importpfx ".p12文件的位置和名称" NoExport`；导入过程中会要求输入证书密码，密码可通过上边的配置导出进行查看；这个步骤不论手工还是CLI实现，最终只要保证在导入证书后，将客户端证书放在 "个人 -> 证书" 目录中，并且将 CA 证书放在 "受信任的根证书颁发机构 -> 证书" 目录中
2. 创建IKEv2 VPN连接,需要注意服务器地址必须和服务器端配置的一致，否则无法正常完成证书认证：

   ```cmd
    # 创建 VPN 连接（将服务器地址换成你自己的值）
    powershell -command "Add-VpnConnection -ServerAddress '你的 VPN 服务器 IP（或者域名）' -Name 'My IKEv2 VPN' -TunnelType IKEv2 -AuthenticationMethod MachineCertificate -EncryptionLevel Required -PassThru"
    # 设置 IPsec 参数
    powershell -command "Set-VpnConnectionIPsecConfiguration -ConnectionName 'My IKEv2 VPN' -AuthenticationTransformConstants GCMAES128 -CipherTransformConstants GCMAES128 -EncryptionMethod AES256 -IntegrityCheckMethod SHA256 -PfsGroup None -DHGroup Group14 -PassThru -Force"
   ```

3. 如果是手动创建的，则需要再配置下加密算法：`REG ADD HKLM\SYSTEM\CurrentControlSet\Services\RasMan\Parameters /v NegotiateDH2048_AES256 /t REG_DWORD /d 0x1 /f`

#### OS X

直接将对应客户端的`.mobileconfig`文件传送至Mac，双击后按照提示导入描述文件，会自动配置一个VPN选项，然后连接即可，证书密码可通过导出对应客户端的配置信息查看

#### iOS

与OS X一致

### 同时访问内外网的方案

原理上都是通过路由完成，VPN连接后，可以理解成约等于建立了一个虚拟网卡，虚拟网卡的网关就是VPN服务器对应的子网，连接VPN后，会导致不在路由表中的流量默认走了VPN，所以会导致无法访问内网；基于这个原理，解决方法就是去配置新的路由信息：

1. Win10: 
    ```cmd
    # 查看路由
    route print
    # 添加(永久)路由
    route -p add 192.168.0.1 mask 255.255.0.0 192.168.0.1
    route add 192.168.0.1 mask 255.255.0.0 192.168.0.1
    # 删除路由(模糊匹配，会把所有符合都删除)
    route delete [路由信息]
    ```

## 参考文档

- [docker-ipsec-vpn-server](https://github.com/hwdsl2/docker-ipsec-vpn-server/blob/master/README-zh.md#%E9%85%8D%E7%BD%AE%E5%B9%B6%E4%BD%BF%E7%94%A8-ikev2-vpn)
- [IKEv2 VPN 配置和使用指南](https://github.com/hwdsl2/setup-ipsec-vpn/blob/master/docs/ikev2-howto-zh.md)