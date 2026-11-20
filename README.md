# OpenResty Windows 构建

用 GitHub Actions 在官方推荐的 **MSYS2 MINGW64** 环境里编译 OpenResty，产出 win64 压缩包。

流程与官方 `openresty/openresty` 自己的 `.github/workflows/windows-build.yml` 完全一致：

1. `util/mirror-tarballs` —— 拼装出 `openresty-<ver>/` 完整源码树
2. 进目录跑 `util/build-win32.sh`

> **关键点**：openresty 的 **git 仓库根目录里没有 `configure`，也没有 `bundle/`**。
> 这些是 `util/mirror-tarballs` 从网络下载几十个组件后生成出来的，只在正式发布的 tar.gz 里现成存在。
> 所以从 git 直接跑 `./util/build-win32.sh` 一定会在 `./configure` 那一步报 `No such file or directory`。

## 用法

1. 把本仓库推到 GitHub（建议 private）
2. 打开 **Actions** → 左侧选 `Build OpenResty (Windows x64)` → **Run workflow**
3. `ref` 填写：
   - `master` —— 拿最新主线（当前 `util/ver` 是 1.31.9.1，含 nginx 1.31.9 的安全修复）
   - `v1.31.8.1` —— 拿已发布的正式版
4. 等 25~45 分钟（`mirror-tarballs` 会下 40+ 个组件包并生成文档索引），在运行详情页底部 **Artifacts** 下载 `openresty-win64`

## 产物结构

安装前缀保持官方脚本里的 `--prefix=`（空值 → 安装落在构建目录、路径全部相对），所以产物可绿色分发：

```
openresty-<ver>-win64.zip
└── openresty-<ver>-win64/
    ├── nginx.exe
    ├── conf/
    ├── html/
    ├── logs/
    ├── lualib/
    ├── lua/
    ├── include/
    ├── pod/
    ├── luajit.exe
    ├── lua51.dll
    ├── resty / restydoc / restydoc-index
    ├── libgcc_s_seh-1.dll / libwinpthread-1.dll / libstdc++-6.dll
    └── README.txt
```

解压后直接启动：

```cmd
cd openresty-<ver>-win64
start nginx
```

## 需要注意

- **Windows 版 OpenResty 只支持 select IO 模型，不支持 IOCP**，单 worker，并发上限 1024。官方定位是本地开发调试，不要用于生产。
- `resty` / `restydoc` 是 Perl 脚本，运行需要系统里有 Perl（StrawberryPerl / MSYS2 perl 均可）。官方包用 `pl2bat` 生成了 `.bat` 包装，本仓库没有 Strawberry Perl，所以没做这一步；只用 `nginx.exe` 不受影响。
- `mirror-tarballs` 会 `git clone` nginx 官方文档仓库来生成 `restydoc` 索引，这一步最慢。如果卡住或失败，多半是网络问题，重跑一次即可。
- 编译 `master` 时，若某个组件版本还是 rc 且未发布 tarball，`mirror-tarballs` 里对应 `get-tarball` 会 404，日志会写明是哪个组件。
- 免费额度：公开仓库不限时长；私有仓库每月 2000 分钟，单次构建约 25~45 分钟。
