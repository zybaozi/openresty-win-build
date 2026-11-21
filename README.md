# OpenResty 自动构建（Windows / 银河麒麟 V10 Server）

两个独立 workflow，各自在 Actions 页面单独触发：

| workflow | 文件 | 产出 |
|---|---|---|
| Build OpenResty (Windows x64) | `.github/workflows/build-windows.yml` | `openresty-<ver>-win64.zip` |
| Build OpenResty (Kylin V10 Server / el8) | `.github/workflows/build-kylin-el8.yml` | `openresty-<ver>-el8-x86_64.tar.gz`（可选 aarch64） |

## 共同关键点：必须先拼装源码树

openresty 的 **git 仓库根目录里没有 `configure`，也没有 `bundle/`**，只有 `util/`、`patches/`、`specs/` 这些东西。它们是由 `util/mirror-tarballs` 从网络下载 40+ 个组件后生成出来的，只现成存在于官方发布的 `tar.gz` 里。

所以两个 workflow 都是同一个套路（与官方自己的 CI 一致）：

```
bash util/mirror-tarballs          # 生成 openresty-<ver>/ 完整源码树
cd openresty-$(./util/ver)
编译
```

直接 `./util/build-win32.sh` 或 `./configure` 都会报 `No such file or directory`。

## 怎么跑

推到 GitHub 后进 **Actions**，左边选对应 workflow，点 **Run workflow**：

- `ref`：`master`（当前 1.31.9.1，含 nginx 1.31.9 安全修复）或 `v1.31.8.1`（已发布正式版）
- Windows 的 `jobs`：并行数，runner 4 核填 `4`
- 麒麟的 `arm64`：客户是飞腾/鲲鹏就选 `yes`，Intel/AMD 保持 `no`

跑完在运行详情页底部 **Artifacts** 下载。

耗时参考（`mirror-tarballs` 下 40+ 组件 + 生成文档索引最慢）：

- Windows：25~45 分钟
- 麒麟 x86_64：15~25 分钟

## Windows 产物

```
openresty-<ver>-win64.zip
└── openresty-<ver>-win64/
    ├── nginx.exe
    ├── conf/  html/  logs/  lualib/  lua/  include/  pod/
    ├── luajit.exe  lua51.dll
    ├── resty / restydoc / restydoc-index
    └── libgcc_s_seh-1.dll / libwinpthread-1.dll / libstdc++-6.dll
```

保持官方 `--prefix=`（空值）安装，路径全相对，解压即可用：

```cmd
cd openresty-<ver>-win64
start nginx
```

## 麒麟产物

```
openresty-<ver>-el8-x86_64.tar.gz
└── usr/local/openresty/...     # 安装前缀 /usr/local/openresty
```

部署（先在客户机上留档，再切）：

```bash
/usr/local/openresty/nginx/sbin/nginx -V 2>&1 | tee /root/nginx-V-before.txt
systemctl stop openresty
cp -a /usr/local/openresty /usr/local/openresty.bak.$(date +%Y%m%d%H%M)
tar xzf openresty-<ver>-el8-x86_64.tar.gz -C /tmp/stage
# 保留客户原有配置
cp -a /usr/local/openresty/nginx/conf/. /tmp/stage/usr/local/openresty/nginx/conf/
mv /usr/local/openresty /usr/local/openresty.old
mv /tmp/stage/usr/local/openresty /usr/local/openresty
systemctl start openresty
/usr/local/openresty/nginx/sbin/nginx -v
```

Artifacts 里还附了 `nginx-V-<ver>-x86_64.txt`，就是本次编译的 `nginx -V` 参数清单。

### 目标机依赖

Linux 版 OpenResty **用系统库编译**（不像 Windows 那样把 pcre/openssl/zlib 一起编进去），所以二进制依赖：

```
libpcre2-8.so.0   libssl.so.1.1   libcrypto.so.1.1   libz.so.1
```

麒麟 V10 底座是 el8，这几个都自带。拷过去先确认一下：

```bash
ldd /usr/local/openresty/nginx/sbin/nginx | grep -v openresty
```

workflow 的构建日志里也会打印这段（`=== 目标机必须具备的系统库 ===`），部署前对一眼。缺 pcre2 就 `dnf install -y pcre2`。

## 为什么用 rockylinux:8 容器

银河麒麟 V10（SP1/SP2/SP3）底座是 CentOS 8 / RHEL 8 生态，**glibc 2.28**。
如果用 `ubuntu-latest` 裸编（glibc 2.39），二进制拷过去会直接报 `GLIBC_2.34 not found`。

workflow 里跑了这个校验，输出必须 ≤ 2.28：

```bash
objdump -T nginx | grep -o 'GLIBC_[0-9.]*' | sort -Vu | tail -3
```

另外用的是 `make install DESTDIR=/tmp/stage`，暂存目录打包，编译进去的 prefix 依然是 `/usr/local/openresty`。

## 可能踩的坑

- **`the HTTP rewrite module requires the PCRE library`**：编译环境缺 PCRE 开发库。Linux 版 OpenResty 用系统库，必须装 `pcre2-devel pcre-devel zlib-devel openssl-devel`（已在 workflow 的 dnf 清单里）。
- **`mirror-tarballs` 卡在 `git clone nginx.org`**：网络问题，重跑。只影响 `restydoc` 文档索引，不影响编译。
- **某个组件 `get-tarball` 404**：master 引用了还没发布的 rc 版本，日志会写明组件名；先跑 `v1.31.8.1` 验证流程本身没问题。
- **`unix2dos` / `pod2text` 缺失**：workflow 里已放兜底 shim，不会中断构建。
- **aarch64 job 起不来**：`ubuntu-24.04-arm` runner 在部分私有仓库（免费个人账号）不可用，会报 no runner matching labels。这种情况把 `arm64` 保持 `no`，或改成在 `ubuntu-latest` 上用 QEMU 模拟（慢 3~5 倍）。
- **Windows 版限制**：只支持 select IO 模型，不支持 IOCP，单 worker，并发上限 1024。官方定位是本地开发调试，别用于生产。
