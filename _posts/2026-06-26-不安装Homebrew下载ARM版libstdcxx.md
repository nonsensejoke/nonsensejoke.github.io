---
layout: default
title: "不安装 Homebrew 下载 ARM 版 libstdc++.6.dylib"
date: 2026-06-26 12:00:00 +0800
categories: [macOS, 工具]
tags: [macos, arm64, gcc, homebrew, dylib]
description: "在 Apple Silicon Mac 上不安装 Homebrew，直接从 GHCR 下载 GCC bottle 提取 libstdc++.6.dylib 的完整方法。"
---

适用场景：你需要在 **macOS Apple Silicon / arm64** 上拿到 `libstdc++.6.dylib`，但不想完整安装 Homebrew。不适用：iOS 真机、Linux、Intel Mac。iOS/Xcode 项目更推荐迁移到 `libc++`。

---

## 0. 先确认你的系统和架构

```bash
uname -m
sw_vers -productVersion
```

`uname -m` 应该输出 `arm64`。系统版本与 Homebrew bottle 标签对应关系：

| macOS 版本 | Homebrew bottle 标签 |
|---|---|
| macOS 26 Tahoe | `arm64_tahoe` |
| macOS 15 Sequoia | `arm64_sequoia` |
| macOS 14 Sonoma | `arm64_sonoma` |

---

## 1. 选择对应的 bottle URL

### macOS 26 Tahoe / Apple Silicon

```bash
OS_TAG="arm64_tahoe"
VERSION="16.1.0"
URL="https://ghcr.io/v2/homebrew/core/gcc/blobs/sha256:39faa08c413c043b9f44a6be5beb260d5acf7fd0bf035688c2579d3be48a8463"
SHA256="39faa08c413c043b9f44a6be5beb260d5acf7fd0bf035688c2579d3be48a8463"
```

### macOS 15 Sequoia / Apple Silicon

```bash
OS_TAG="arm64_sequoia"
VERSION="16.1.0"
URL="https://ghcr.io/v2/homebrew/core/gcc/blobs/sha256:083223989b47d242c7c42366abaf3e8509f7bdde84a370cb7381c9267a199847"
SHA256="083223989b47d242c7c42366abaf3e8509f7bdde84a370cb7381c9267a199847"
```

### macOS 14 Sonoma / Apple Silicon

```bash
OS_TAG="arm64_sonoma"
VERSION="16.1.0"
URL="https://ghcr.io/v2/homebrew/core/gcc/blobs/sha256:9d4be966090f2585d27a717c96208077059e43c5ab5b510f1525b0efc2a21bb7"
SHA256="9d4be966090f2585d27a717c96208077059e43c5ab5b510f1525b0efc2a21bb7"
```

---

## 2. 下载官方 bottle

选好上面其中一组变量后，执行：

```bash
TOKEN="$(curl -fsSL 'https://ghcr.io/token?scope=repository:homebrew/core/gcc:pull' \
  | sed -n 's/.*"token":"\([^"]*\)".*/\1/p')"

curl -fL \
  -H "Authorization: Bearer $TOKEN" \
  "$URL" \
  -o "gcc-${VERSION}-${OS_TAG}.bottle.tar.gz"
```

校验 SHA256：

```bash
echo "$SHA256  gcc-${VERSION}-${OS_TAG}.bottle.tar.gz" | shasum -a 256 -c -
```

看到以下结果说明文件完整：

```text
gcc-16.1.0-arm64_sequoia.bottle.tar.gz: OK
```

---

## 3. 解压并找到 `libstdc++.6.dylib`

```bash
mkdir -p "$HOME/Downloads/gcc-bottle"

tar -xzf "gcc-${VERSION}-${OS_TAG}.bottle.tar.gz" \
  -C "$HOME/Downloads/gcc-bottle"

find "$HOME/Downloads/gcc-bottle" \
  -name 'libstdc++.6*.dylib' \
  -print
```

通常会找到类似路径：

```text
/Users/你的用户名/Downloads/gcc-bottle/gcc/16.1.0/lib/gcc/current/libstdc++.6.dylib
```

确认是 ARM 版：

```bash
LIB="$(find "$HOME/Downloads/gcc-bottle" -name 'libstdc++.6.dylib' | head -n 1)"
file "$LIB"
lipo -archs "$LIB"
```

输出里看到 `arm64` 就确认无误。

---

## 4. 推荐：按 Homebrew 原本路径放置

`gcc` 的 Apple Silicon bottle Cellar 路径是 `/opt/homebrew/Cellar`，按此放置可以避免路径引用问题：

```bash
sudo mkdir -p /opt/homebrew/Cellar /opt/homebrew/opt

sudo tar -xzf "gcc-${VERSION}-${OS_TAG}.bottle.tar.gz" \
  -C /opt/homebrew/Cellar

sudo ln -sfn ../Cellar/gcc/${VERSION} /opt/homebrew/opt/gcc

ls -l /opt/homebrew/opt/gcc/lib/gcc/current/libstdc++.6.dylib
```

之后的稳定路径：

```text
/opt/homebrew/opt/gcc/lib/gcc/current/libstdc++.6.dylib
```

---

## 5. 让旧程序找到这个库

先查看程序依赖：

```bash
otool -L /path/to/your_binary | grep libstdc++
```

如果它依赖 `/usr/lib/libstdc++.6.dylib`，用 `install_name_tool` 改掉引用路径：

```bash
install_name_tool -change \
  /usr/lib/libstdc++.6.dylib \
  /opt/homebrew/opt/gcc/lib/gcc/current/libstdc++.6.dylib \
  /path/to/your_binary
```

改完再检查：

```bash
otool -L /path/to/your_binary | grep libstdc++
```

---

## 6. 不要复制到 `/usr/lib`

现代 macOS 有 System Integrity Protection，会保护 `/System`、`/usr`、`/bin`、`/sbin` 等位置。正确做法是将库放在 `/opt/homebrew/opt/gcc/...`，通过 `install_name_tool` 修改旧程序的引用路径。

---

## 7. macOS 13 Ventura / macOS 12 Monterey 的备选方案

当前 `gcc` bottle 主要覆盖 Tahoe / Sequoia / Sonoma，较旧系统可以使用 `gcc@11` bottle。

### macOS 13 Ventura / Apple Silicon

```bash
OS_TAG="arm64_ventura"
VERSION="11.5.0"
URL="https://ghcr.io/v2/homebrew/core/gcc/11/blobs/sha256:b0936cf63d97795bfe8e17a9e4a4f75be0fd3f9407e467d5d02df1fba5caa499"
SHA256="b0936cf63d97795bfe8e17a9e4a4f75be0fd3f9407e467d5d02df1fba5caa499"
```

```bash
TOKEN="$(curl -fsSL 'https://ghcr.io/token?scope=repository:homebrew/core/gcc/11:pull' \
  | sed -n 's/.*"token":"\([^"]*\)".*/\1/p')"

curl -fL \
  -H "Authorization: Bearer $TOKEN" \
  "$URL" \
  -o "gcc-${VERSION}-${OS_TAG}.bottle.tar.gz"

echo "$SHA256  gcc-${VERSION}-${OS_TAG}.bottle.tar.gz" | shasum -a 256 -c -
```

### macOS 12 Monterey / Apple Silicon

```bash
OS_TAG="arm64_monterey"
VERSION="11.5.0"
URL="https://ghcr.io/v2/homebrew/core/gcc/11/blobs/sha256:85d430ddf596edf1f14639ca164d7370a46456c017008723392fcf9cd1f91bfa"
SHA256="85d430ddf596edf1f14639ca164d7370a46456c017008723392fcf9cd1f91bfa"
```

```bash
TOKEN="$(curl -fsSL 'https://ghcr.io/token?scope=repository:homebrew/core/gcc/11:pull' \
  | sed -n 's/.*"token":"\([^"]*\)".*/\1/p')"

curl -fL \
  -H "Authorization: Bearer $TOKEN" \
  "$URL" \
  -o "gcc-${VERSION}-${OS_TAG}.bottle.tar.gz"

echo "$SHA256  gcc-${VERSION}-${OS_TAG}.bottle.tar.gz" | shasum -a 256 -c -
```

解压后按 Homebrew 路径放置：

```bash
sudo mkdir -p /opt/homebrew/Cellar /opt/homebrew/opt
sudo tar -xzf "gcc-${VERSION}-${OS_TAG}.bottle.tar.gz" -C /opt/homebrew/Cellar
sudo ln -sfn ../Cellar/gcc@11/${VERSION} /opt/homebrew/opt/gcc@11
find /opt/homebrew/opt/gcc@11 -name 'libstdc++.6*.dylib' -print
```

---

## 参考

- [Homebrew gcc formula](https://formulae.brew.sh/formula/gcc)
- [Homebrew gcc Formula JSON API](https://formulae.brew.sh/api/formula/gcc.json)
- [Homebrew Bottles 文档](https://docs.brew.sh/Bottles)
- [Apple System Integrity Protection 说明](https://support.apple.com/en-us/102149)
- [Homebrew gcc@11 formula](https://formulae.brew.sh/formula/gcc@11)

---

**完**
