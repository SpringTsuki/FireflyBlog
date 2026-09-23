---
title: Windows AD 重要问题库
published: 2026-09-11
description: 记录值得反复复习的 AD 故障场景、排查过程与待研究课题。
tags: [AD, Practice]
category: AD
---

这篇文章作为 AD 学习过程中的“重要问题库”，保存值得反复复习的故障场景、排查思路和实验结论。已经研究的题目会整理完整答案；尚未研究的题目只记录目标和验收标准，避免在没有实验验证时提前下结论。

# 已完成的重要问题

## 一、域控时间同步源梳理与修复

定位 PDC 主时钟源、检查域内 NTP 层级漂移，排查所有 DC 的时间误差来源，修正 GPO 时间同步配置。近一周若总是域认证偶发失败，这是高嫌疑方向。

### 1.1 场景设定

> **工单现象**：今天上午，多个业务系统与域相关的认证开始"偶发失败"。用户反馈：
>
> - 部分员工登录域内 Web 系统（Kerberos 集成认证）提示"凭据无效"，重试 2–3 次有时能进；
> - 两台应用服务器上的服务账号定时任务大量报 `KRB_AP_ERR_SKEW`；
> - 文件服务器访问延迟明显升高……
> - 事件日志报告 `KRB_AP_ERR_SKEW`，Kerberos 错误码为 `0x25`。
>
> **已知**：昨晚 22:00 有一次虚拟化平台维护，其中 DC02 被迁移/重启过；今天早上有人"顺手"把 DC02 的时间手动往回调了 8 分钟"纠正显示"。

### 1.2 剖析故障

单点现象：`KRB_AP_ERR_SKEW`、`Clock skew too great`、Kerberos 错误码 `0x25`；

扩散现象：Kerberos 集成认证、使用 Kerberos 的 LDAP 绑定或 AD 复制出现失败；

迷惑项：DNS 解析正常、网络 ping 正常、账号密码确认无误 —— "什么都好，就是认证不对"。

### 1.3 开始分析

先证明"时间"是嫌疑，而不是直接怀疑账号。

```powershell
# 全 DC 时间偏差一览（在管理机上对每台 DC 轮询）
w32tm /stripchart /computer:QYNET-CORE01 /samples:3
w32tm /stripchart /computer:QYNET-CORE02 /samples:3
# 域内时间层级是否健康
w32tm /monitor /domain:qy.net
```

`w32tm /monitor` 会列出各 DC 的时间源、层级和相对偏移量。命令本身不会自动把异常项标红，需要根据偏移量和报错手动判断。

其次，确认 PDC 是谁、它的时间源是什么。

```powershell
netdom query fsmo
w32tm /query /status /verbose
w32tm /query /configuration
w32tm /query /peers
```

然后确认时间层级。AD 的时间同步采用域层级（NT5DS）：成员计算机通常向其身份验证 DC 同步；域内 DC 向域 PDC Emulator 靠拢；子域 PDC 继续沿域层级向上同步；**林根域的 PDC Emulator** 才应配置为使用批准的外部或企业权威时间源。

```powershell
# 仅在林根域 PDC 上配置经过批准的时间源
w32tm /config /manualpeerlist:"ntp1.example.net,0x8 ntp2.example.net,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync /rediscover
```

> 注意：这里特指**林根域的 PDC Emulator**。其他 DC 和域成员通常保持 NT5DS 域层级同步。实际生产环境应使用组织批准且可达的 NTP 源，不应直接照抄示例地址。

最后，修复并收敛

```powershell
# 非 PDC 的 DC 恢复为域层级同步
w32tm /config /syncfromflags:domhier /update
Restart-Service w32time
w32tm /resync /rediscover

# 验证并重新发现时间源
w32tm /query /source
w32tm /resync /rediscover
```

时间偏差收敛后，新发起的 Kerberos 请求通常可以恢复。若客户端仍持有故障期间取得的旧票据，可在评估会话影响后重新登录，或使用 `klist purge` 清理当前登录会话的票据，再进行最终验证：

```powershell
w32tm /monitor /domain:qy.net       # 偏差回到毫秒级
repadmin /replsummary                     # 复制是否也恢复正常
Get-Service W32Time                       # 确认 Automatic + Running
```

### 1.4 指令速查

| 目的 | 命令 |
|---|---|
| 全域时间偏差总览 | `w32tm /monitor /domain:<域>` |
| 单机偏差抽样 | `w32tm /stripchart /computer:<DC> /samples:5` |
| 查看当前源与状态 | `w32tm /query /status /verbose` |
| 查看时间提供程序配置 | `w32tm /query /configuration` |
| 查看同步对端 | `w32tm /query /peers` |
| 林根域 PDC 配权威源 | `w32tm /config /manualpeerlist:"..." /syncfromflags:manual /reliable:yes /update` |
| 恢复域层级同步 | `w32tm /config /syncfromflags:domhier /update` |
| 强制重新同步 | `w32tm /resync /rediscover` |
| 查 FSMO 角色 | `netdom query fsmo` |
| 复制健康检查 | `repadmin /replsummary` |

`w32tm /register` 用于重新注册 Windows Time 服务，通常要配合停止服务以及必要时的 `/unregister`，属于服务修复动作，不应在所有 DC 上当作普通同步命令批量执行。


### 1.5 日志模板（可选阅读）

```markdown
# 工作日志：AD 域控时间同步故障排查与修复

日期：YYYY-MM-DD
执行人：
环境：测试域 contoso.com（DC01=PDC/GC，DC02=DC/GC，成员服务器 fs01/app01）
关联 Topic：AD 每日研究 #1 域控时间同步源梳理与修复

## 一、问题现象
- [填写：谁、什么时候、反馈什么现象，例如"09:12 起多个系统 Kerberos 认证偶发失败"]
- [填写：事件日志关键条目与 Event ID]

## 二、排查过程
1. 现象分类：先区分是"凭据问题"还是"环境问题"，[填写判断依据]
2. 时间层检查：执行 `w32tm /monitor /domain:contoso.com`，结果：
   - DC01 偏移：___ 秒
   - DC02 偏移：___ 秒
3. 定位 PDC 与时间源：`netdom query fsmo` → PDC 为 ___；
   `w32tm /query /status /verbose` → Source 为 ___，Stratum ___，
   Last Successful Sync Time ___
4. 发现异常配置：[例如 DC02 被改为 manual peer，或 PDC 指向不可达源]
5. 排除项：DNS 解析正常、网络连通正常、账号密码验证正常（已验证，非根因）

## 三、根因
[填写：一句话说明，例如"DC01 上 PDC Emulator 的外部时间源配置丢失且被手动改表，
导致域内时间层级失效，Kerberos 因超过 5 分钟容差而拒绝认证"]

## 四、修复动作
1. 在 DC01(PDC) 上重新配置外部 NTP 源并标记 reliable：`w32tm /config ...`
2. 在 DC02 等 DC 上恢复 `NT5DS` 域层级：`w32tm /config /syncfromflags:domhier /update`
3. 全量重新同步：`w32tm /resync /rediscover` + 重启 W32Time
4. 关闭虚拟化平台与 w32time 冲突的主机时间同步（VMware Tools Time Sync）
5. 核对 GPO 时间相关策略，防止被覆盖

## 五、验证结果
- `w32tm /monitor`：各 DC 偏差收敛至 ___ ms
- Kerberos 认证：复测业务系统登录 ___ 次，成功 ___ 次
- `repadmin /replsummary`：无失败项
- W32Time 服务状态：Running / Automatic（全部 DC）

## 六、复盘与改进
- 教训：[例如"认证类故障应第一时间排查时间层，而非凭据"]
- 待办：
  - [ ] 将时间健康检查加入日常巡检（每日/每周自动采集 `w32tm /monitor`）
  - [ ] 通过 GPO 固化时间源配置，禁止手工改表
  - [ ] 关闭所有虚拟化域控的宿主时间同步
  - [ ] 补充监控告警：DC 时间偏移 > 阈值 触发告警

## 七、耗时统计
- 排查：___ 分钟
- 修复：___ 分钟
- 验证与收尾：___ 分钟
- 合计：___ 分钟
```

## 二、全局编录(GC)覆盖审计与站点复制拓扑核查

检查每台 DC 是否承担 GC，确认站点间是否仍有可用复制路径，并结合 `repadmin /replsummary` 生成健康报告。

量化：找出复制失败的 DC 对，修复并复测。

GC：Global Catalog，中文叫全局编录。它算是一个角色，但更准确地说，它是 DC 承担的一种功能角色——不是所有 DC 都必须承担，但承担了的 DC 就叫 GC 服务器。

为了核查 GC 的状态，把它分成三步：检查、诊断、修复。

### 2.1 GC 覆盖审计

先确认每台 DC 是否承担 GC 角色。这里使用 Powershell：

```powershell
# 简易列出 GC
Get-ADDomainController -Filter * | Select-Object Name, Site, IsGlobalCatalog

# 详细列出 GC
Get-ADDomainController -Filter * | Where-Object {$_.IsGlobalCatalog -eq $true}

# 当然、dsquery 也可以
dsquery server -isgc
```

### 2.2 检查复制拓扑

```powershell
# 查看所有 DC 的入站复制状态
repadmin /showrepl *

# 查看指定 DC 的连接对象
repadmin /showconn <DC Name>

# 使用 PowerShell 查询站点、站点链接和连接对象
Get-ADReplicationSite -Filter *
Get-ADReplicationSiteLink -Filter *
Get-ADReplicationConnection -Filter *
```

也可以在“Active Directory 站点和服务”（`dssite.msc`）中检查 Sites、Subnets、Servers、NTDS Settings 以及 Inter-Site Transports。删除一条站点链接不一定立即让跨站点复制完全中断，因为 KCC 可能经其他可用链接重新计算路径；真正需要确认的是目标站点是否仍存在可达的复制路径，以及 KCC 是否成功生成连接对象。

### 2.3 生成复制健康报告

```powershell
# 这是核心命令，它会列出每台 DC 的最大复制偏差（delta）和失败计数，快速定位有问题的 DC
repadmin /replsummary

# 如果要更详细的 CSV 报告，使用管道导出即可
repadmin /showrepl * /csv > repl.csv

# 只看有错误的
repadmin /replsummary /errorsonly
```

### 2.4 定位并修复失败的 DC

一旦 replsummary 指出某台 DC 有问题，用 showrepl 深入即可。

```powershell
repadmin /showrepl <DCName> /verbose
```

输出里会列出每个入站伙伴的上次成功时间、上次失败时间、最近的 Win32 错误码。常见复制错误码速查如下表：

| 错误码 | 含义 | 首查方向 |
|---|---|---|
| **1722** | RPC 服务器不可用 | 网络/防火墙、DC 是否在线、DNS 解析 |
| **8524** | DSA 操作/DNS 失败 | DNS SRV/CNAME 记录缺失或过期 |
| **-2146893022 / 0x80090322** | 目标主体名称不正确 | SPN、计算机账户密码、DNS 指向、Kerberos 与复制一致性 |
| **8453** | 复制访问被拒绝 | 权限、复制授权、令牌、机器账户与安全通道 |
| **1256** | 远程系统不可用 | 伙伴离线，如果之前有过其他错误，先查那个更早的 |
| **8606** | 缺少足够属性，无法创建对象 | 重点检查残留对象和复制顺序，不要盲目强制同步 |
| **8614** | 距离上次复制已超过墓碑生存期 | DC 可能进入复制隔离，应先评估恢复或重建方案 |

找到原因后着手进行修复，最后再次使用 repadmin /replsummary 进行校验是否修复完成即可。

## 三、用户已经加入组，但仍然没有权限

这是 AD 权限排查中的高频问题。不要只确认“用户是否在组里”，需要同时检查四层状态：

1. **AD 成员关系**：组是否为 Security、作用域是否正确、嵌套关系能否展开、变更是否已复制。
2. **登录令牌**：用户当前令牌是否已经包含新组，必要时注销并重新登录。
3. **共享权限**：SMB Share ACL 是否允许访问。
4. **文件系统权限**：NTFS ACL 是否允许访问，是否存在优先级更高的显式拒绝。

```powershell
Get-ADGroup -Identity "目标组" -Properties GroupCategory,GroupScope

Get-ADGroupMember -Identity "目标组" -Recursive

whoami /groups

# 查看客户端现有的票据和 SMB 连接
klist
net use

# 在文件服务器上查询共享 ACL
Get-SmbShareAccess -Name "ShareName"

# 查询目录的 NTFS ACL
Get-Acl "D:\Shares\ShareName" |
    Select-Object -ExpandProperty Access

repadmin /replsummary
```

`Get-SmbSession` 和 `Get-SmbOpenFile` 查询的是会话与打开文件，不是共享 ACL。`gpupdate /force` 和 `klist purge` 都不会重新生成用户的桌面登录令牌。用户在登录后才被加入组时，最可靠的验证方法仍然是注销并重新登录；旧 SMB 连接还应在关闭文件后重新建立。

## 四、AD 复制错误 1722：RPC 服务器不可用

错误 1722 表示 RPC 通信无法建立，不等于目标 DC 一定关机。推荐沿着以下顺序排查：

```text
DNS 解析
    ↓
基础网络与路由
    ↓
TCP 135（RPC Endpoint Mapper）
    ↓
RPC 动态端口与防火墙
    ↓
NTDS、RPC、Netlogon 等服务
    ↓
DCDiag、Repadmin 与事件日志
```

```powershell
Resolve-DnsName QYNET-CORE02.qy.net
Test-NetConnection QYNET-CORE02 -Port 135
Test-NetConnection QYNET-CORE02 -Port 389
Test-NetConnection QYNET-CORE02 -Port 445

Get-Service -ComputerName QYNET-CORE02 `
    -Name RpcSs,RpcEptMapper,Netlogon,NTDS,KDC,DNS

dcdiag /s:QYNET-CORE02 /test:Connectivity /v
dcdiag /s:QYNET-CORE02 /test:DNS /DnsBasic /v
dcdiag /s:QYNET-CORE02 /test:Replications /v
repadmin /showrepl QYNET-CORE02 /verbose
```

Ping 使用 ICMP，不存在“Ping 端口”。Ping 失败可能只是 ICMP 被阻止；Ping 成功也不能证明 TCP 135 和协商出的 RPC 动态端口可用。完成修复后必须重新执行 `repadmin /replsummary` 和 `/showrepl`，并核对 System、Directory Service、DNS Server 与 DFS Replication 日志。

## 五、计算机能够登录域，但没有应用预期 GPO

能够使用域账户登录，只能证明至少有一台 DC 完成了认证，不能证明计算机位于正确 OU、GPO 链接有效、SYSVOL 可访问或策略处理成功。

推荐检查链：

```text
计算机对象与 OU
    ↓
GPO 链接、继承和安全筛选
    ↓
客户端 DNS 与 DC Locator
    ↓
DC Advertising、复制与 SYSVOL
    ↓
gpupdate 与 gpresult
```

```powershell
Get-ADComputer -Identity PC-LAB-01 |
    Select-Object Name,DistinguishedName

Get-GPInheritance -Target "OU=Lab,OU=Computer,OU=Organization,DC=qy,DC=net"

nltest /dsgetdc:qy.net
Test-Path "\\qy.net\SYSVOL"
Test-Path "\\qy.net\NETLOGON"

gpupdate /force
gpresult /Scope Computer /R
gpresult /Scope Computer /H "C:\Temp\Computer-GPO.html"
```

Day1 的完整答题与批改过程见：[Windows AD Practice Day1](/posts/springtsuki/notes/windowsad/practice/922/)。

## 六、服务从 Kerberos 回退到 NTLM

业务能够打开不代表正在使用 Kerberos。应用可能因为 SPN 缺失、重复、绑定错误、访问名称不匹配或客户端配置而回退到 NTLM。

推荐检查链：

```text
访问名称与 DNS
    ↓
KDC 与 Kerberos SRV
    ↓
主动请求目标服务票据
    ↓
SPN 唯一所有者
    ↓
服务实际运行身份
    ↓
客户端票据与 KDC 事件
    ↓
修复后重新认证并验证协议
```

```powershell
Resolve-DnsName testweb.qy.net
Resolve-DnsName -Type SRV "_kerberos._tcp.qy.net"
nltest /dsgetdc:qy.net /KDC /force

klist get HTTP/testweb.qy.net

setspn -F -Q HTTP/testweb.qy.net
setspn -X -F
setspn -L QY\svc_testweb

w32tm /query /status
klist
```

修复后，应同时确认客户端出现目标服务票据、DC 记录成功的 4769、对应访问不再持续产生 4776，并且 SPN 属于服务实际使用的账户。`klist get` 成功只证明 KDC 能签发票据，不能单独证明服务端可以解密、应用已经使用票据或 ACL 已授权。

Day2 的完整答题与批改过程见：[Windows AD Practice Day2](/posts/springtsuki/notes/windowsad/practice/923/)。

# 待研究课题

以下课题暂时只记录研究目标和验收标准。完成实验、取得实际输出并验证结论后，再补充正式答案。

## 课题一：域安全基线 GPO 差异审计

对照 Microsoft Security Baselines 或组织采用的安全基线，导出当前域 GPO 设置并生成差异清单，逐项评估兼容性与加固收益。

**验收标准**：输出差异项数量、建议加固点、例外项及回退方案。

## 课题二：RODC 或新增、下架域控的全流程演练

在实验环境演练提升新 DC、验证 DNS 与 GC、转移必要角色、安全降级旧 DC，以及异常降级后的元数据清理。

**验收标准**：形成提升、验证、降级、元数据清理和回退检查清单，并保留关键命令结果。

## 课题三：废弃账号与特权账号治理

结合最后登录、密码修改、账户状态、组成员关系和资产负责人确认，识别长期闲置账号以及 DA、EA、BA 等高权限组中的非必要成员。

**验收标准**：记录候选账号、人工复核结果、禁用观察期、最终处置和特权组收敛数量。

## 课题四：受控灾难恢复演练

选择“AD 回收站还原误删对象”或“系统状态备份恢复域控”进行受控演练，明确权威与非权威恢复边界。

**验收标准**：记录 RTO、RPO、恢复步骤、验证项、失败点和完整演练报告。

## 课题五：Kerberos 与 NTLM 使用情况观测

基于高级审核日志统计 Kerberos TGT、服务票据及 NTLM 凭据验证事件，识别仍依赖 NTLM 的客户端、服务和访问路径。

**验收标准**：统计认证协议占比，列出 NTLM 来源、业务负责人、迁移难点和可执行的收紧计划。

## 课题六：DNS 记录清理与条件转发健康检查

检查条件转发器、存根区域、SRV、CNAME、A 与 PTR 记录，识别陈旧或冲突记录；所有删除操作必须先生成候选报告并确认业务归属。

**验收标准**：记录清理候选、审批结果、实际清理数量、转发延迟和清理后的客户端解析验证。

## 课题七：细粒度密码策略（PSO）

在确认域功能级别和业务需求后，为高权限人员、服务账户和普通用户设计不同的密码与锁定策略，并验证 Resultant PSO。

**验收标准**：保留改造前后配置、优先级设计、目标用户、冲突处理和生效验证样例。

## 课题八：域与林信任关系安全审计

梳理信任方向、传递性、身份验证范围、SID History 与 SID Filtering 等配置，评估攻击者利用信任关系横向移动的风险。

**验收标准**：形成信任清单、风险项、业务依赖、整改建议和验证方法。

---

每日训练入口：[Day1](/posts/springtsuki/notes/windowsad/practice/922/) · [Day2](/posts/springtsuki/notes/windowsad/practice/923/)
