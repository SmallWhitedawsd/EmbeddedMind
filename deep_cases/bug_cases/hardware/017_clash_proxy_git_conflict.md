# 017: Clash系统代理导致Git操作失败

## 项目
多项目 (Git版本管理)

## 环境
Windows, Git, Clash代理

## 现象
Git push/pull失败或超时

## 影响范围
代码同步

## 排查过程

### 假设1: 网络不通
- 验证方法：ping github.com
- 结果：网络可达

### 假设2: Clash代理拦截git流量
- 验证方法：关闭Clash后重试
- 结果：关闭Clash后git正常

## 根因
Clash系统代理默认拦截git.exe的流量，导致GitHub连接失败

## 修改方案
```bash
# 方案1: 不要关闭Clash系统代理(保持开启)
# 方案2: git单独配置代理
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890

# 方案3: 用SSH替代HTTPS(不受HTTP代理影响)
git remote set-url origin git@github.com:user/repo.git
```

## 验证结果
保持Clash开启，git操作正常

## 预防措施
- 不要关闭Clash系统代理
- 或git配置SSH方式

## 经验规则
- Clash代理≠git代理
- Windows git使用系统代理设置
- SSH方式不受HTTP代理影响

## 来源
ses_150d61a0affenekmtXcDx74eJQ - 2026-06-17
