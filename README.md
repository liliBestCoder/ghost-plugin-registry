# Ghost Proxifier 官方插件注册表

这个仓库存放 Ghost Proxifier 插件中心的**官方注册表**：`registry.json`、它的根签名 `registry.json.sig`、根**公钥** `root-public.b64`，以及把它们发布成 GitHub Release 的工作流。格式与规则见 SDK 的 [`spec-release.md` §2](https://github.com/liliBestCoder/ghost-plugin-sdk/blob/main/docs/plugin-sdk/spec-release.md)。

**这个仓库里永远没有根私钥，工作流也从不签名。** 根私钥离线保存在维护者手里，每一版 `registry.json` 都在离线机器上签好，再把两份文件一起提交。

## 当前状态

| 项 | 值 |
|---|---|
| 最新版本 | `seq 2`（Release [`seq-2`](https://github.com/liliBestCoder/ghost-plugin-registry/releases/tag/seq-2)，`updatedAt` 2026-09-28T15:43:22Z） |
| 根签名 | `registry.json.sig`，用正式根钥离线签名，发布前本地验签、工作流再验一次 |
| 根公钥 | `root-public.b64`：`+jc4yKv3TV7hKxkBsgSjrflI2Wei8ePylSHnURxRVvhyIZEJM08lAnpoQdV+nFbcdpw6Uqm851I3/VOyyCdU/Q==` |
| 根公钥指纹 | 64 字节 X‖Y 的 SHA-256：`ebcb486c95ec4e215ab065d8b6dbc009ea0869603b49c4fbff2c41b9636a8d27` |
| 吊销 | 无（`revoked: []`） |

客户端内置的就是这把根公钥，每次都用它验证这里发布的 `registry.json`。2026-09-26 那把临时验证根钥已经退役，客户端不再信任它签过的任何东西。

### 已收录的插件

| id | 仓库 | 状态 | 付费 |
|---|---|---|---|
| `com.ghostproxifier.showcase`（插件能力示例） | [liliBestCoder/ghost-plugin-showcase](https://github.com/liliBestCoder/ghost-plugin-showcase) | `listed` | 否 |
| `com.ghostproxifier.events-viewer`（事件查看器） | [liliBestCoder/ghost-plugin-events-viewer](https://github.com/liliBestCoder/ghost-plugin-events-viewer) | `listed` | 否 |

每条的 `devKey` 是对应插件仓库里的 `dev-public.b64`。插件的发布说明必须用这把开发者钥签名，客户端才会安装。

## 客户端从哪里取

- 主源：`https://github.com/liliBestCoder/ghost-plugin-registry/releases/latest/download/registry.json`，以及同一路径下的 `registry.json.sig`
- 备用镜像：`https://ghostproxifier.com/plugins/registry.json` 与 `registry.json.sig`

⚠️ **备用镜像目前还没有部署。** 主源访问正常时不影响安装；GitHub 访问不了时，客户端也回落不到镜像。部署时，要把同一份 `registry.json` 与 `.sig` **逐字节**放上去。

## 发一版

```bash
# 1. 改 registry.json：seq 加一（必须严格大于已发布的最大 seq-<N>），
#    增删条目、改状态、吊销、抬 minVersion，并更新 updatedAt
python <sdk>/tools/plugin/gpkg.py check-registry --strict --file registry.json
# 2. 在离线机器上签名（签的是原始字节：不去 BOM、不转行尾）
pwsh -NoProfile -File <sdk>/tools/plugin/sign.ps1 -Key <离线的根私钥> -File registry.json -Force
python <sdk>/tools/plugin/gpkg.py verify-sig --pubkey root-public.b64 --file registry.json
# 3. 两份文件一起提交，推到 main
git add registry.json registry.json.sig && git commit -m "Registry seq N: ..." && git push
```

`<sdk>` 指 [liliBestCoder/ghost-plugin-sdk](https://github.com/liliBestCoder/ghost-plugin-sdk) 的一份检出。`verify-sig` 需要 Python 的 `cryptography`，`check-registry` 只用标准库。

上架一个新插件时，`devKey` 填该插件仓库 `dev-public.b64` 的内容，`repo` 填 `owner/name`。先确认那个仓库已经打过 `v<版本>` 标签，Release 里有 `.gpkg`、`ghost-plugin.json` 与 `ghost-plugin.json.sig` 三个文件。

## 工作流做什么

`.github/workflows/publish.yml` 只在 `main` 上的 `registry.json` 或 `registry.json.sig` 变化时运行，**不用任何 secret**。只改 README 不会触发它。它从公开的 SDK 仓库 `liliBestCoder/ghost-plugin-sdk` 的固定提交 `4a6fc4b7d593a7b9c1fdd905a42d81a5a5baad6a` 取工具，actions 都按提交 sha 钉住，`cryptography` 按 `.github/requirements-release.txt` 里的哈希安装。依次检查：

1. `root-public.b64` 非空；
2. `gpkg.py check-registry --strict`：与客户端解析规则逐条一致，外加客户端 1 MB 的下载上限；`--strict` 另外拒绝全零的 `devKey` 占位；
3. `gpkg.py verify-sig --pubkey root-public.b64`：验原始字节的签名；
4. 新 `seq` 必须**严格大于**已发布的最大 `seq-<N>`，否则客户端会以 `registry_rollback` 拒绝；
5. `gh release create seq-<N> registry.json registry.json.sig --latest`。

`.gitattributes` 对这三个文件关闭了行尾转换。签名签的是字节，Windows 上 `core.autocrlf` 一旦把 LF 换成 CRLF，签名就对不上了。

## 吊销与下架

- **下架**：把条目的 `status` 改成 `delisted`。已安装的用户不受影响，新用户装不了。
- **吊销**：发现安全问题时，除了让插件作者发修复版，还要把该条目的 `minVersion` 抬到修复版本，必要时在 `revoked[]` 里列出受影响的版本（`spec-release.md` §2.3）。客户端会停用被吊销的版本。
- 每一次改动都要走上面「发一版」的完整流程：`seq` 加一，离线签名。

## 许可

目前没有选定许可证，保留所有权利。
