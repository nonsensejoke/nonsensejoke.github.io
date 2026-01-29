# 在 SSH 主机上创建并使用 Git Bare 仓库教程

## 一、什么是 bare 仓库？

**bare 仓库（裸仓库）**：

- ❌ 没有工作区（没有源码文件可直接编辑）
- ✅ 只保存 Git 历史、分支、tag 等
- ✅ 非常适合作为远程仓库（central repo）

通常命名为：

```
xxx.git
```

---

## 二、前提条件

### 本地
- 已安装 Git
- 有一个已经初始化的 Git 仓库

### 远程 SSH 主机
- 可 SSH 登录（密码或 SSH key）
- 已安装 Git

---

## 三、在 SSH 主机上创建 bare 仓库

### 1️⃣ 登录远程主机

```bash
ssh user@your.remote.host
```

（如果 SSH 用的是非 22 端口）：

```bash
ssh -p 2222 user@your.remote.host
```

---

### 2️⃣ 创建并初始化 bare 仓库

```bash
mkdir -p ~/git/myproject.git
cd ~/git/myproject.git
git init --bare
```

---

## 四、本地仓库配置远程 SSH bare 仓库

### 添加远程仓库（指定 SSH 端口）

```bash
git remote add origin ssh://user@your.remote.host:2222/home/user/git/myproject.git
```

或使用 SSH config（推荐）：

```text
Host myserver
    HostName your.remote.host
    Port 2222
    User user
```

```bash
git remote add origin myserver:/home/user/git/myproject.git
```

---

## 五、推送代码到远程 bare 仓库

```bash
git push -u origin main
```

---

## 六、bare 仓库里查看文件

列出文件：

```bash
git --git-dir=~/git/myproject.git ls-tree -r main
```

查看文件内容：

```bash
git --git-dir=~/git/myproject.git show main:README.md
```

---

## 七、clone 出工作区

```bash
git clone ~/git/myproject.git ~/checkout/myproject
```
