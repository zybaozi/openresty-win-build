# OpenResty Windows 构建

用 GitHub Actions 在官方推荐的 **MSYS2 MINGW64** 环境里编译 OpenResty，产出 win64 压缩包。

全程复用官方 `util/build-win32.sh`，编译参数与官方 Windows 构建一致（`--platform=msys`、`--with-select-module` 等）。

## 用法

1. 把本仓库推到 GitHub（建议 private）
2. 打开 **Actions** → 左侧选 `Build OpenResty (Windows x64)` → **Run workflow**
3. `ref` 填写：
   - `master` —— 拿最新主线（当前 `util/ver` 是 1.31.9.1，含 nginx 1.31.9 的安全修复）
   - `v1.31.8.1` —— 拿已发布的正式版
4. 等 15~25 分钟，在运行详情页底部 **Artifacts** 下载 `openresty-win64`

## 产物结构

```
openresty-<ver>-win64.zip
└── openresty/
    ├── nginx/sbin/nginx.exe
    ├── nginx/conf/
    ├── nginx/logs/
    ├── lualib/
    ├── luajit/
    ├── bin/           (resty 等脚本，需系统有 perl)
    └── *.dll          (MinGW 运行时，已一并打包)
```

解压后直接用，启动：

```cmd
cd openresty\nginx
start nginx
```

## 需要注意

- **Windows 版 OpenResty 只支持 select IO 模型，不支持 IOCP**，单 worker，并发上限 1024。官方定位是本地开发调试，不要用于生产。
- `resty` 命令行需要系统里有 Perl（StrawberryPerl / MSYS2 perl / Git 自带 perl 均可）。
- 编译 `master` 时，若某个组件版本还是 rc 且未镜像到 openresty.org，`./configure` 会下载失败。届时把 `util/build-win32.sh` 或 `bundle/` 里对应组件版本退一格即可。
- 免费额度：公开仓库不限时长；私有仓库每月 2000 分钟，单次构建约 15~25 分钟。
