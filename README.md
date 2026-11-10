# hound-lzcapp

[Hound-Media-Server/hound](https://github.com/Hound-Media-Server/hound) 的懒猫微服打包，
仅发布到**喵喵（私有）商店**，镜像模式（不复制镜像、只改运行时地址）。

**Hound** 是一体化媒体服务器：找片、看片、记录追剧进度、管理影音库，
支持本地影音库、下载源和 IPTV 直播，另有 Android / Android TV / iOS / tvOS 客户端。

## 打包内容

| 项目 | 值 |
|------|----|
| 包名 | `community.lazycat.app.hound` |
| 上游镜像 | `houndmediaserver/hound` + `library/postgres`（均为镜像模式） |
| 运行时加速 | `docker.1ms.run`（Action 默认 Docker Hub 加速地址） |
| 版本来源 | 上游 `-beta`  release 标签，去掉后缀作为包版本（`0.4.3-beta` → `0.4.3`） |
| 容器端口 | 2323（hound）、5432（postgres，仅内部） |
| 持久化 | `/lzcapp/var/data`（影音库/下载/缓存）、`/lzcapp/var/db`（数据库）、`/lzcapp/run/mnt/home/<影视库目录>`（只读） |
| 健康检查 | hound：`wget http://127.0.0.1:2323/`；postgres：`pg_isready` |

## 上游镜像标签

Hound 只把 GitHub Release 推到 Docker Hub，所以带 `-beta` 的标签是稳定的一条线；
仓库里同时存在 `-staging`、`-demo`、`webgpu` 等开发标签和会漂移的 `latest`。
`tag_regex` 只收 `^\d+\.\d+\.\d+-beta$`，`version_regex` 去掉后缀，
镜像引用里保留原标签（`.../hound:0.4.3-beta`）。

Postgres 作为第二个镜像单独跟踪 `18.x-alpine` 系列，跟上游 compose 的主版本一致，
不参与包版本计算（`version_source` 只指向 hound）。

## 数据目录：绕开上游带空格的路径

上游把库和下载目录写死成相对工作目录的 `Hound Data`（带空格），
直接绑到容器里带空格的路径不好维护，所以：

```yaml
setup_script: rm -rf "/app/Hound Data" && ln -s /data "/app/Hound Data"
binds:
  - /lzcapp/var/data:/data
```

容器 rootfs 每次启动都是全新的，脚本先清同名目录再建软链，重复执行结果一致。

外部影音库走懒猫的文稿目录（`document.private` 权限），只读挂到 `/extlib`，
并把 `EXTERNAL_LIBRARY_MOVIES_PATH` / `EXTERNAL_LIBRARY_SHOWS_PATH`
指到 `/extlib/Movies` 和 `/extlib/Shows`（同样避免空格）。

## 部署参数

- **TMDB API Key**：Hound 只用 TMDB 做元数据，不填基本没法用，强烈建议填写。
- **MDBList API Key**：可选，导入 MDBList 片单。
- **影视库目录**：文稿目录下的子目录名，默认 `Media`，只读挂载。
- **外部影音库扫描**：默认关闭。

数据库口令与 `HOUND_SECRET` 由 `stable_secret` 自动生成，不需要用户填写，
同一微服每次渲染结果一致，因此重建容器不会丢凭据。

## 访问方式说明

Hound 有自己的账号体系，但**首次启动会写入上游固定的默认口令**
（用户名 `admin`，口令是仓库里公开的默认值），所以本包**不开 `public_path`**，
先用懒猫平台入口登录兜一层，用户在应用内改掉口令。

需要让手机 / TV 客户端直连 API 时，请先改掉默认口令，再自行把
`lzc-manifest.yml` 里的 `public_path` 放开重新打包。

## 版本与发布

- 每天定时检查上游 `-beta` 新标签。
- 新版本会更新 `package.yml` 的 `version` 与 `lzc-manifest.yml` 的镜像地址，
  构建 LPK、创建带版本号的 Release 资产，再提交到喵喵商店。
- 官方商店关闭（`stores.official.enabled: false`），不会触发官方审核。

## 版权

上游 [Hound-Media-Server/hound](https://github.com/Hound-Media-Server/hound)，
AGPL-3.0（全文见 `content/licenses/LICENSE`）。本仓库只做懒猫打包，不修改上游代码；
按 AGPL 的源码提供义务，用户可通过上面的上游仓库地址获取对应版本的完整源码。

本包尚未在懒猫实机安装验证。
