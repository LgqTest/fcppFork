# fcpp 项目本地 CI 快速使用指南

**版本**: 2.0.0
**日期**: 2026-01-16
**状态**: ✅ 已验证通过

---

## 快速开始

### 1. 触发 CI 测试

```bash
cd /home/lgq/conan_dev_github_self_runner/fcppFork

# 触发 CI（使用 :fire: emoji）
git commit --allow-empty -m ":fire: 触发本地 CI 测试"
git push origin main
```

### 2. 查看 CI 运行

访问 GitHub Actions 页面：
```
https://github.com/你的用户名/fcppFork/actions
```

### 3. 验证构建结果

```bash
# 搜索上传的包
~/.local/bin/conan search "fcpp/*" --remote=local-server

# 查看包详情
~/.local/bin/conan get "fcpp/1.0.0@" --remote=local-server
```

---

## CI 触发规则

| Trigger | 说明 | 示例 |
|---------|------|------|
| `:fire:` | 触发本地 Conan Server + GitHub Runner CI | `:fire: 本地测试` |
| `:beer:` | 触发完整自动化测试 | `:beer: 完整测试` |

---

## 关键配置文件

### CI 配置
- `.github/workflows/ci-local-conan-server-test.yml` - 本地 CI 配置
- `test_package/conanfile.py` - 测试包配置
- `test_package/main.cpp` - 测试代码

### 环境要求
- ✅ Conan Server 运行中（端口 9300）
- ✅ GitHub Self-Hosted Runner 运行中
- ✅ sudo 免密配置完成

---

## 常见命令

### 检查服务状态
```bash
# Conan Server
bash /home/lgq/conan/scripts/status_server.sh

# GitHub Runner
bash /home/lgq/conan_github_self_runner/scripts/check_all.sh
```

### Conan 操作
```bash
# 搜索本地服务器上的包
~/.local/bin/conan search "*" --remote=local-server

# 下载包
~/.local/bin/conan download "fcpp/1.0.0" --remote=local-server

# 清理缓存
~/.local/bin/conan remove "fcpp/*" --confirm
```

---

## 故障排除

### CI 未触发
- ✅ 检查 commit message 是否包含 `:fire:`
- ✅ 确认已推送到 main 分支

### Runner 不可用
- ✅ 检查 Runner 服务状态
- ✅ 查看 Runner 标签配置

### Conan Server 连接失败
- ✅ 检查服务是否运行
- ✅ 测试端口：`curl http://localhost:9300/v1/ping`

---

## 完整文档

详细的部署和配置指南请参考：
```
/home/lgq/conan_github_self_runner/conan_github_self_runner_guide_all_in_one.md
```

---

**维护者**: lgq
**最后更新**: 2026-01-16
