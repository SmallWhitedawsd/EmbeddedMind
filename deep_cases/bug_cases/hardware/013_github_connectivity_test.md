# 013: GitHub连接性问题排查

## 项目
WSC-16改造 (代码托管)

## 环境
Windows, Git, GitHub

## 现象
Testing GitHub connectivity

## 影响范围
代码推送/拉取

## 排查过程

### 假设1: 网络问题
- 验证方法：ping github.com
- 结果：网络可达

### 假设2: 代理配置
- 验证方法：检查git proxy设置
- 结果：Clash系统代理可能影响git

### 假设3: SSH/HTTPS认证
- 验证方法：检查remote URL和credential
- 结果：需确认认证方式

## 根因
GitHub连接可能受系统代理/防火墙影响

## 修改方案
```bash
# 检查连接
git ls-remote https://github.com/SmallWhitedawsd/WSC-16.git

# 如有代理问题
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890

# 或取消代理
git config --global --unset http.proxy
```

## 验证结果
连接正常后可正常push/pull

## 预防措施
- 不要关闭Clash系统代理(会影响git)
- 或用SSH方式推送

## 经验规则
- Windows git默认使用系统代理
- Clash等代理工具可能拦截git流量
- SSH不受HTTP代理影响

## 来源
ses_0978c7055ffeuQmCZRvUe5nXzD - 2026-07-16
