# Ghost Proxifier 插件注册表（模板）

这个目录是将来 `liliBestCoder/ghost-plugin-registry` 的全部内容：官方注册表 `registry.json`、它的根签名 `registry.json.sig`、根**公钥** `root-public.b64`，以及把它们发布成 GitHub Release 的工作流。格式与规则见 `docs/plugin-sdk/spec-release.md` §2。

**这个仓库里永远没有根私钥，工作流也从不签名。** 根私钥离线保存在维护者手里；每一版 `registry.json` 都在离线机器上签好，两份文件一起提交。

## 模板里的占位

| 文件 | 现在 | 维护者要做的 |
|---|---|---|
| `root-public.b64` | **空文件** | 填入根公钥（`pwsh -File tools/plugin/sign.ps1 -Key <根私钥> -PublicKey` 打印的 88 字符）。空着时 `publish.yml` 第一步就以一句明确的话失败 |
| `registry.json` | `seq: 1`，一条 `com.ghostproxifier.events-viewer`，`devKey` 是 64 个零字节的 base64 | 换成示例插件仓库真实的 `dev-public.b64` 内容，再离线签名 |
| `registry.json.sig` | 不存在 | 离线签名的产物 |

`devKey` 的占位值语法合法（`check-registry` 通过），但不是曲线上的点：用它的注册表发布出去，这个插件的每一次安装都会以 `release_bad_sig` 失败——占位就该这样失败，而不是碰巧能用。

## 发一版

```bash
# 1. 改 registry.json：seq 加一（必须严格大于已发布的最大 seq-<N>），增删条目、吊销、抬 minVersion
python <sdk>/tools/plugin/gpkg.py check-registry --file registry.json
# 2. 离线签名（原始字节，不去 BOM、不转行尾）
pwsh -NoProfile -File <sdk>/tools/plugin/sign.ps1 -Key <离线的根私钥> -File registry.json -Force
python <sdk>/tools/plugin/gpkg.py verify-sig --pubkey root-public.b64 --file registry.json
# 3. 两份文件一起提交、推到 main
git add registry.json registry.json.sig && git commit -m "registry seq N" && git push
```

`.github/workflows/publish.yml` 在 `main` 上 `registry.json` 或 `registry.json.sig` 变化时运行，**零 secret**。它先守门：`SDK_REPO`（**公开**的 SDK 仓库，这个工作流的 `GITHUB_TOKEN` 读不了私有仓库）与 `SDK_REF`（那个仓库一次提交的 40 位 sha，须含 `check-registry` 与 `verify-sig`）还是模板占位符就失败——填法见 SDK 的 `docs/plugin-sdk/maintainer.md` §0 与 §2.3。actions 按提交 sha 钉住，`cryptography` 按 `.github/requirements-release.txt` 的哈希装。然后：

1. `root-public.b64` 非空；
2. `gpkg.py check-registry --strict`——`plugin_docs.h` `ParseRegistry` 的 Python 镜像，外加客户端 1 MB 的下载上限；`--strict` 另拒全零的 `devKey` 占位（`placeholder_dev_key`），所以模板原样发不出去；
3. `gpkg.py verify-sig --pubkey root-public.b64`——验原始字节；
4. 新 `seq` 必须**严格大于**已发布的最大 `seq-<N>`（否则客户端以 `registry_rollback` 拒绝）；
5. `gh release create seq-<N> registry.json registry.json.sig --latest`。

`.gitattributes` 对这三个文件关闭了行尾转换：签名覆盖的是字节，Windows 上 `core.autocrlf` 把 LF 换成 CRLF，签名就对不上了。

同一份 `registry.json` 与 `.sig` 还要**逐字节**放到备用镜像 `https://ghostproxifier.com/plugins/`（维护者清单见 SDK 的 `docs/plugin-sdk/maintainer.md`）。

吊销与 `minVersion` 是发版流程的一部分：修掉一个安全问题，除了插件作者发新版，还要把该条目的 `minVersion` 抬到那个版本，必要时在 `revoked[]` 里列出受影响的版本（`spec-release.md` §2.3）。
