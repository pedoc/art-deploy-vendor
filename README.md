# art-deploy-vendor

ART 独立部署包所用的第三方运行时镜像。

部署脚本按固定 URL + SHA256 下载这些文件，避免上游地址滚动或失效导致构建不可复现。

## 发布内容

| Release | 资产 | 说明 |
|---|---|---|
| `assets-7za-26.04` | `7za-26.04-x64.zip` | 7-Zip 26.04 独立版命令行（x64），供构建脚本解压组件包 |
| `assets-nginx-1.28.3-el9` | `nginx-1.28.3-el9.tar.gz` | nginx 1.28.3，静态链接 OpenSSL 3 / PCRE2 / zlib，供 Linux 部署包携带 |
| `assets-nginx-1.28.3-el7` | `nginx-1.28.3-el7.tar.gz` | 同上，构建于 glibc 2.17（CentOS 7.9），适用于 el7 / el8 |

## 说明

- 每个版本一个 release，tag 与资产名固化，内容不再变更。
- 上游来源与校验和写在各自 release 的说明里。
- 仅镜像分发所需的可再分发包，许可证随包提供。

---

## nginx-1.28.3-el9

### 为什么需要自己构建

Linux 部署包要求 **nginx 随包携带**：客户机多为内网环境，装不了软件包，不能假设系统已有 nginx。

但 libssl / libcrypto 的 ABI 在各大版本间互不兼容：

| 发行版 | OpenSSL | SONAME |
|---|---|---|
| el7 | 1.0.2 | `libssl.so.10` |
| el8 | 1.1.1 | `libssl.so.11` |
| el9 | 3.x | `libssl.so.3` |

动态链接的二进制只能跑在对应版本上。实测 nginx.org 的 el7 官方包在 el9 上直接报
`libssl.so.10: cannot open shared object file`。

因此改为源码构建 + **静态链接** OpenSSL、PCRE2、zlib，产物只依赖 glibc：

```
$ ldd nginx-1.28.3-el9/sbin/nginx
        linux-vdso.so.1
        libcrypt.so.2 => /lib64/libcrypt.so.2
        libc.so.6     => /lib64/libc.so.6
        /lib64/ld-linux-x86-64.so.2
```

### 该用哪一份

| 目标系统 | 用哪个 | 理由 |
|---|---|---|
| el9+ | `assets-nginx-1.28.3-el9` | 链接 `libcrypt.so.2`，属 glibc 自带 |
| el7 / el8 | `assets-nginx-1.28.3-el7` | 链接 `libcrypt.so.1`，el7/el8 原生提供 |

el7 那份在 el9 上也能跑（glibc 向后兼容，已实测），但它依赖 `libcrypt.so.1`——
el9 上该库由 `libxcrypt-compat` 提供，最小化安装可能缺失。因此 el9 仍建议用 el9 那份。

### 构建方法



在 **Rocky Linux 9.5**（glibc 2.34）上执行。构建机需能访问 nginx.org / openssl.org / GitHub。

```bash
dnf install -y gcc make perl-interpreter perl-core tar xz

NGINX_VER=1.28.3
OPENSSL_VER=3.0.16
PCRE2_VER=10.44
ZLIB_VER=1.3.1

mkdir -p build && cd build
curl -sLO "https://nginx.org/download/nginx-$NGINX_VER.tar.gz"
curl -sLO "https://www.openssl.org/source/openssl-$OPENSSL_VER.tar.gz"
curl -sLO "https://github.com/PCRE2Project/pcre2/releases/download/pcre2-$PCRE2_VER/pcre2-$PCRE2_VER.tar.gz"
curl -sLO "https://github.com/madler/zlib/releases/download/v$ZLIB_VER/zlib-$ZLIB_VER.tar.gz"

for f in nginx openssl pcre2 zlib; do tar xzf $f*.tar.gz; done
cd nginx-$NGINX_VER

./configure \
  --prefix=/opt/nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_realip_module \
  --with-http_stub_status_module \
  --with-threads \
  --with-openssl=../openssl-$OPENSSL_VER \
  --with-openssl-opt='no-shared no-tests' \
  --with-pcre=../pcre2-$PCRE2_VER \
  --with-pcre-jit \
  --with-zlib=../zlib-$ZLIB_VER

make -j"$(nproc)"
```

`no-shared` 是静态链接的关键；`--with-pcre` / `--with-zlib` 指向源码目录同样会静态编入。
确认 `make` 末尾的链接行包含 `libssl.a`、`libcrypto.a`、`libpcre2-8.a`、`libz.a`。

### 打包

```bash
OUT=nginx-bundle/nginx-$NGINX_VER-el9
mkdir -p "$OUT/sbin" "$OUT/conf"
cp build/nginx-$NGINX_VER/objs/nginx      "$OUT/sbin/nginx"
cp build/nginx-$NGINX_VER/conf/mime.types "$OUT/conf/mime.types"
chmod 755 "$OUT/sbin/nginx"
tar -czf nginx-$NGINX_VER-el9.tar.gz -C nginx-bundle nginx-$NGINX_VER-el9
sha256sum nginx-$NGINX_VER-el9.tar.gz
```

部署包按 `stripComponents=1` 解压到 `runtime/nginx/`，因此最终落位是
`runtime/nginx/sbin/nginx` 与 `runtime/nginx/conf/mime.types`。

### 目标系统要求在部署包侧体现

| 项 | 说明 |
|---|---|
| glibc | 构建基线 2.34，要求 **el9+** |
| el7 / el8 | 跑不了这份二进制，需在对应基线重建并新增 profile |
| mime.types | 随包放在 `conf/` 下，不引用 `/etc/nginx/mime.types`（客户机可能没装 nginx） |

### 当前版本

组件版本两份产物一致：

| 项 | 值 |
|---|---|
| nginx | 1.28.3 |
| OpenSSL | 3.0.16 |
| PCRE2 | 10.44 |
| zlib | 1.3.1 |

| 资产 | 构建环境 | 最高 glibc 符号 | SHA256 |
|---|---|---|---|
| `nginx-1.28.3-el9.tar.gz` | Rocky Linux 9.5（glibc 2.34） | GLIBC_2.34 | `8f7445ab700adb6d2163ebf50ee50fbf395f91c25fac9c31b4c87903a749d629` |
| `nginx-1.28.3-el7.tar.gz` | CentOS 7.9.2009（glibc 2.17） | GLIBC_2.17 | `1abbbc09fd76fa22596d5a38ecdbb7dfdfbf83f17bb7abf4ebd135ccab23bcea` |

el7 构建机的额外前置：`yum install perl-IPC-Cmd`。
OpenSSL 3 的 `Configure` 依赖该模块，CentOS 7 自带的 perl 5.16 未包含。

### 许可证

nginx 采用 BSD 2-Clause。OpenSSL 3 采用 Apache 2.0。PCRE2 采用 BSD。
zlib 采用 zlib License。均允许随产品分发，许可文本见各上游发行包。
