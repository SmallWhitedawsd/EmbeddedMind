# 网络优化模式

---

## 流程1: 双网卡路由优先级优化

### 触发条件
- 同时连接 WiFi 和以太网
- 需要强制特定应用走指定网卡
- 默认路由冲突导致网络不稳定

### 步骤
1. 检查当前路由表和 metric
2. 识别各网卡接口 metric
3. 调整接口 metric 控制优先级
4. 清理冲突的持久路由
5. 验证生效

### 关键命令
```powershell
# 检查当前路由表
Get-NetRoute -AddressFamily IPv4 | Where-Object { $_.DestinationPrefix -eq "0.0.0.0/0" } |
    Select-Object ifIndex, InterfaceAlias, NextHop, RouteMetric | Format-Table -AutoSize

# 检查接口 metric
Get-NetIPInterface -AddressFamily IPv4 |
    Select-Object InterfaceAlias, InterfaceMetric | Format-Table -AutoSize

# 设置接口 metric (WiFi 优先)
Set-NetIPInterface -InterfaceAlias "WLAN" -AddressFamily IPv4 -InterfaceMetric 5
Set-NetIPInterface -InterfaceAlias "以太网 2" -AddressFamily IPv4 -InterfaceMetric 999

# 通过 netsh 设置 (需管理员)
netsh interface ipv4 set interface "WLAN" metric=5
netsh interface ipv4 set interface "以太网 2" metric=50

# 清理持久路由
Get-NetRoute -PolicyStore PersistentStore -AddressFamily IPv4 |
    Where-Object { $_.DestinationPrefix -eq "0.0.0.0/0" } |
    Select-Object DestinationPrefix, NextHop, InterfaceAlias

# 删除冲突的持久路由
Remove-NetRoute -DestinationPrefix "0.0.0.0/0" -NextHop "192.168.1.1" -PolicyStore PersistentStore -Confirm:$false

# 通过注册表删除持久路由 (需管理员)
reg delete "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\PersistentRoutes" /v "0.0.0.0,0.0.0.0,192.168.0.92,1" /f
```

### 判断逻辑
- WiFi metric < 有线 metric → WiFi 优先
- 多条默认路由 → 删除低优先级路由
- 持久路由冲突 → 清理注册表 PersistentRoutes

### 来源案例
- GameViewer 远程协助 - 强制走 WiFi 避免走内网

---

## 流程2: 防火墙规则控制应用网络访问

### 触发条件
- 需要限制特定应用只能走指定网络
- 阻止应用访问局域网/特定 IP

### 步骤
1. 确认应用可执行文件路径
2. 创建阻止规则 (阻止局域网段)
3. 创建允许规则 (放行目标 IP)
4. 验证规则生效

### 关键命令
```powershell
# 阻止 GameViewer 走局域网 (需管理员)
$fwScript = @'
netsh advfirewall firewall add rule name="GV_Block_Eth_Main" dir=out program="D:\apps\GameViewer\bin\GameViewer.exe" action=block localip=192.168.0.0/16 protocol=tcp
netsh advfirewall firewall add rule name="GV_Block_Eth_Launcher" dir=out program="D:\apps\GameViewer\GameViewer.exe" action=block localip=192.168.0.0/16 protocol=tcp
netsh advfirewall firewall add rule name="GV_Block_Eth_Service" dir=out program="D:\apps\GameViewer\GameViewerService.exe" action=block localip=192.168.0.0/16 protocol=tcp
'@
[System.IO.File]::WriteAllText("$env:TEMP\block_gameviewer.ps1", $fwScript, [System.Text.Encoding]::Unicode)
Start-Process -FilePath "powershell" -ArgumentList "-NoProfile -ExecutionPolicy Bypass -File `"$env:TEMP\block_gameviewer.ps1`"" -Verb RunAs

# 允许 GameViewer 走 WiFi
$allowCmd = 'netsh advfirewall firewall add rule name="GV_Allow_WiFi" dir=out program="D:\apps\GameViewer\bin\GameViewer.exe" action=allow localip=10.169.76.138 protocol=tcp'
Start-Process -FilePath "powershell" -ArgumentList "-NoProfile -Command `"$allowCmd`"" -Verb RunAs

# 查看规则
Get-NetFirewallRule -DisplayName "GV_*" | Select-Object DisplayName, Direction, Action, Enabled

# 清理规则
Get-NetFirewallRule -DisplayName "GV_*" | Remove-NetFirewallRule -Confirm:$false
```

### 判断逻辑
- 阻止规则优先于允许规则 (block > allow)
- 规则按名称字母顺序匹配
- 需管理员权限创建/删除规则

### 来源案例
- GameViewer - 强制走 WiFi 不走内网

---

## 流程3: ForceBindIP 强制应用绑定网卡

### 触发条件
- 应用不支持选择出口网卡
- 需要绕过系统路由表强制绑定

### 步骤
1. 获取目标网卡 IP
2. 使用 ForceBindIP 启动应用
3. 验证连接走正确网卡

### 关键命令
```powershell
# 获取 WiFi 网卡 IP
$wifiIP = (Get-NetIPAddress -InterfaceAlias "WLAN" -AddressFamily IPv4).IPAddress

# 使用 ForceBindIP 启动应用
$psi = New-Object System.Diagnostics.ProcessStartInfo
$psi.FileName = "D:\apps\ForceBindIP\ForceBindIP64.exe"
$psi.Arguments = "$wifiIP `"D:\apps\GameViewer\GameViewer.exe`""
$psi.Verb = "runas"
[System.Diagnostics.Process]::Start($psi)

# PowerShell 脚本版本
$wifiIP = (Get-NetIPAddress -InterfaceAlias "WLAN" -AddressFamily IPv4).IPAddress
Start-Process -FilePath "D:\apps\ForceBindIP\ForceBindIP64.exe" -ArgumentList @($wifiIP, "D:\apps\GameViewer\GameViewer.exe") -Verb RunAs
```

### 判断逻辑
- 应用成功启动 → 绑定成功
- 连接走 WiFi → 目标 IP 可达
- 启动失败 → 检查 ForceBindIP 版本 (32/64位)

### 来源案例
- GameViewer - 强制绑定 WiFi 网卡

---

## 流程4: 代理配置与绕过

### 触发条件
- 需要临时关闭代理测试直连
- 需要对比走代理 vs 直连的延迟

### 步骤
1. 检查当前代理设置
2. 临时关闭代理
3. 测试直连速度
4. 恢复代理

### 关键命令
```powershell
# 检查代理设置
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" |
    Select-Object ProxyEnable, ProxyServer, ProxyOverride

# 关闭代理
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" -Name ProxyEnable -Value 0

# 开启代理
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" -Name ProxyEnable -Value 1
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" -Name ProxyServer -Value "127.0.0.1:7890"

# 直连速度测试 (绕过代理)
$wc = New-Object System.Net.WebClient
$wc.Proxy = $null  # 强制不走代理
$sw = [System.Diagnostics.Stopwatch]::StartNew()
$data = $wc.DownloadData("https://speed.cloudflare.com/__down?bytes=1000000")
$sw.Stop()
$speed = $data.Length / $sw.Elapsed.TotalSeconds / 1MB
Write-Output "直连速度: {0:N2} MB/s ({1:N0} Mbps)" -f $speed, $speed * 8

# 对比走代理
$wc2 = New-Object System.Net.WebClient
$sw2 = [System.Diagnostics.Stopwatch]::StartNew()
$data2 = $wc2.DownloadData("https://speed.cloudflare.com/__down?bytes=1000000")
$sw2.Stop()
```

### 判断逻辑
- 直连速度 > 代理速度 → 代理瓶颈
- 直连无法连接 → 需要代理/检查网络
- 延迟差异大 → 代理服务器负载高

### 来源案例
- GameViewer - 对比走 Clash 代理 vs 直连延迟

---

## 流程5: 网络连接状态诊断

### 触发条件
- 应用无法连接服务器
- 需要排查网络层问题

### 步骤
1. 检查网卡状态
2. 检查默认路由
3. DNS 解析测试
4. TCP 连通性测试
5. 检查防火墙状态

### 关键命令
```powershell
# 检查网卡状态
Get-NetAdapter | Where-Object { $_.Status -eq "Up" } |
    Select-Object Name, InterfaceDescription, Status, LinkSpeed | Format-Table -AutoSize

# 检查默认路由
Get-NetRoute -AddressFamily IPv4 | Where-Object { $_.DestinationPrefix -eq "0.0.0.0/0" } |
    Select-Object NextHop, InterfaceAlias, RouteMetric | Format-Table -AutoSize

# DNS 解析测试
Resolve-DnsName -Name "gameviewer.com" -ErrorAction SilentlyContinue

# TCP 连通性测试
Test-NetConnection -ComputerName "117.147.201.71" -Port 443 -WarningAction SilentlyContinue |
    Select-Object ComputerName, RemotePort, TcpTestSucceeded, InterfaceAlias

# 检查丢包率
$results = Test-Connection -ComputerName "27.36.121.21" -Count 20 -ErrorAction SilentlyContinue
$loss = ($results | Where-Object { $_.StatusCode -ne 0 }).Count
$avgLatency = ($results | Where-Object { $_.StatusCode -eq 0 } | Measure-Object -Property ResponseTime -Average).Average
Write-Output "丢包率: $loss/20  平均延迟: ${avgLatency}ms"

# 检查防火墙状态
netsh advfirewall show currentprofile | Select-String "状态"

# 检查进程网络连接
Get-NetTCPConnection -State Established | Where-Object { $_.RemotePort -eq 443 } |
    Group-Object OwningProcess | ForEach-Object {
        $proc = Get-Process -Id $_.Group[0].OwningProcess -ErrorAction SilentlyContinue
        [PSCustomObject]@{ Process = $proc.ProcessName; Count = $_.Count }
    } | Format-Table -AutoSize
```

### 判断逻辑
- 网卡 Down → 检查驱动/物理连接
- 无默认路由 → 检查 DHCP/静态 IP
- DNS 失败 → 更换 DNS 服务器
- TCP 测试失败 → 检查防火墙/路由
- 丢包率高 → 网络质量差/线路问题

### 来源案例
- GameViewer - 网络连接问题排查
- UU 远程协助 - 连接质量检测

---

## 流程6: 持久路由管理

### 触发条件
- 系统重启后路由被覆盖
- 多网关冲突

### 步骤
1. 检查持久路由
2. 删除冲突路由
3. 添加正确的持久路由
4. 验证重启后生效

### 关键命令
```powershell
# 检查持久路由
Get-NetRoute -PolicyStore PersistentStore -AddressFamily IPv4 |
    Where-Object { $_.DestinationPrefix -eq "0.0.0.0/0" } |
    Select-Object DestinationPrefix, NextHop, InterfaceAlias

# 通过注册表查看
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\PersistentRoutes"

# 删除持久路由 (PowerShell)
Remove-NetRoute -DestinationPrefix "0.0.0.0/0" -NextHop "192.168.1.1" -PolicyStore PersistentStore -Confirm:$false

# 删除持久路由 (reg.exe)
reg delete "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\PersistentRoutes" /v "0.0.0.0,0.0.0.0,192.168.1.1,1" /f

# 添加持久路由
New-NetRoute -DestinationPrefix "10.0.0.0/8" -NextHop "192.168.1.1" -PolicyStore PersistentStore

# 通过 route.exe 添加
route -p add 10.0.0.0 mask 255.0.0.0 192.168.1.1 metric 1
```

### 判断逻辑
- 持久路由存在 → 重启后保持
- 多条默认路由 → 按 metric 选路
- 路由冲突 → 删除低优先级

### 来源案例
- GameViewer - 清理冲突的持久路由
