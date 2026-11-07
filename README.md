# Palune Releases

Palune 的**公开发行仓库**：只存放发行产物，**不包含任何源码**。

## 这是什么 / 不是什么

| | |
|---|---|
| 是 | 安装包（exe / msi / dmg / AppImage / deb / rpm）、`latest.json` 更新清单、`SHA256SUMS` 校验值、安装验证证据 |
| 不是 | 源代码、构建脚本、提交历史、Issue 跟踪 —— 这些在私有仓库 `yunyu697/palune` |

源码仓库保持私有；本仓库从创建起即为公开，只读匿名可访问。任何拿到本仓库链接的人都能下载安装包，但拿不到源码。

## 客户端更新源

```
https://github.com/yunyu697/palune-releases/releases/latest/download/latest.json
```

- 该地址**不需要任何 token**，匿名 GET 即可。客户端不内嵌任何凭据，也不使用 `Authorization` 头。
- 由于 GitHub 的 `releases/latest` 只指向「非预发布」的最新 Release，alpha 版本在本仓库中也以**普通 Release**（非 prerelease）发布，否则 `latest/download` 会 404。
- 更新清单的主链只以该地址为准；镜像只用于安装包下载，不参与清单获取与签名判断。

## 资产说明

| 资产 | 说明 |
|---|---|
| `Palune-<版本>-win-x64.exe` / `.msi` | Windows 安装包（NSIS / MSI） |
| `Palune-<版本>-macos-arm64.dmg` | macOS 安装包 |
| `Palune-<版本>-linux-x86_64.AppImage` / `.deb` / `.rpm` | Linux 安装包 |
| `latest.json` | 更新清单：`payload` 明文 + 分离的 Ed25519 签名；每个产物条带 `sha256` 与产物签名 |
| `SHA256SUMS` | 全部资产的 SHA-256，便于离线核对 |
| `windows-install-evidence.json` | Windows 安装 → 启动 → 卸载的验证结果 |
| `install-launch.json` | 启动验证结果 |

## 校验方式

1. `SHA256SUMS` 比对文件哈希；
2. `latest.json` 的 `signature.value` 用客户端内置公钥（`resources/trust/update.pub`）对 `payload` 验签，签名覆盖 `payload` 的规范化字节（键排序、无空白）；
3. 安装包本身的 `sha256` 与该产物条目的 `signature.value` 都要通过，缺一不安装。

镜像返回的内容同样要过这三步，哈希或签名不符即丢弃该通道并自动切换，防止反代投毒。

## 发布链路与回退

- 构建在私有源码仓库完成；发布 job 用具备本仓库 `contents:write` 权限的凭据（细粒度 PAT 或 GitHub App token）把产物推到本仓库的 Release。凭据只存在于私有仓库的 Actions Secret，不进代码、不进客户端。
- 发布后 CI 会匿名 GET 一次 `latest.json`：必须 200 且能取到 `version`，401/404 直接让流水线失败。
- **发布失败回退**：先 `gh release upload <tag> <file> --clobber` 补传缺失资产；若更新清单本身有误，删除本仓库该 Release 重新发布上一版本 tag 的资产，并把客户端更新源临时指向上一版本 tag 的 `latest.json`。旧 Release 不删除，保证已安装客户端仍能取到可用清单。

## 为什么客户端不能内嵌 token

私有仓库的 Release 资产需要授权才能下载；但桌面客户端一旦内嵌只读 token，二进制可被反编译，token 就等于公开，且无法按用户吊销、无法限制下载量。因此采用「源码私有 + 发行仓库公开」的拆分：需要凭据的跨仓写操作只发生在 CI，客户端只访问匿名可读的公开发行仓库。
