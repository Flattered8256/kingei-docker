# KinGei Docker

全栈容器：页面、API、Garage 共用宿主机 **3000**。

## 启动

在仓库根目录：

```
cp .env.example .env
```

改 `.env` 里的 `JWT_SECRET`，然后：

```
docker compose up --build
```

打开 http://127.0.0.1:3000

Windows 上 `localhost` 常解析到 `::1`，若本机还有别的进程绑在 IPv6 的 3000，浏览器会打到那个进程而不是容器。用 `127.0.0.1` 可固定走到 Compose 映射。

局域网其它机器访问时，把 `S3_PUBLIC_ENDPOINT` 改成 `http://<主机IP>:3000` 后重启 api。

把映射改成例如 `8080:80` 时，同步改 `S3_PUBLIC_ENDPOINT=http://127.0.0.1:8080`。

## 端口

只有 `nginx` 映射 `3000:80`。`api`、`garage` 不对外。页面由 nginx 托管 `nuxt generate` 的静态文件，`/api` 和 `/kingei` 由同一进程反代。

数据在 named volume：`api-data`（SQLite）、`garage-meta`、`garage-data`。

## PostgreSQL

同一套 API，镜像用 `--features postgres` 编译。覆盖文件会加 Postgres 容器并把 `DATABASE_URL` 指过去：

```
docker compose -f docker-compose.yml -f docker-compose.postgres.yml up --build
```

仍从宿主机 **3000** 访问。本机 `cargo run` 默认 sqlite；验 Postgres 时：

```
cargo run --no-default-features --features postgres
```

并把 `kingei-rust/.env` 的 `DATABASE_URL` 改成 `postgres://kingei:kingei@127.0.0.1:5432/kingei`（需先把 Postgres 端口映射出来，或只用上面的 compose overlay）。

## 云对象存储

API 走 S3 兼容协议，本机 Garage 和云上 R2 / OSS / COS / MinIO 用同一套变量。云上不跑 Garage 容器，桶事先建好：

```
S3_ENDPOINT=https://<对象存储API>
S3_PUBLIC_ENDPOINT=https://<对象存储API>
S3_ACCESS_KEY=...
S3_SECRET_KEY=...
S3_BUCKET=...
S3_REGION=us-east-1
S3_MANAGE_BUCKET=false
```

`S3_MANAGE_BUCKET=false` 时启动只 `HeadBucket`，不建桶、不改 CORS。浏览器直传预签名 URL，CORS 在控制台配。官方 AWS S3 再加 `S3_PATH_STYLE=false`。

## 本机开发

容器与本机 `cargo run` + `pnpm dev` 可并存。本机仍用 `kingei-rust/.env`（无 `S3_PUBLIC_ENDPOINT` 时预签名指向 `S3_ENDPOINT`，一般是 `127.0.0.1:3900`）。不要同时占用宿主机 IPv4 的 3000。
