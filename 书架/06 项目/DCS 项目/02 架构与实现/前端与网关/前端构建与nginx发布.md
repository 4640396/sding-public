---
title: "前端构建与nginx发布"
type: runbook
status: verified
created: 2026-07-11
updated: 2026-07-25
sensitivity: internal
source:
project: DCS
---
# 前端构建与 nginx 发布

## global 前端脚本

仓库：`$env:WORKS_ROOT\dcs\dcs-global-vehicle-client`

开发：

```powershell
npm run dev:global-vehicle
npm run dev:global-vehicle-junit
npm run dev:global-vehicle-tests
npm run dev:global-vehicle-regression
```

构建：

```powershell
npm run build:test-global-vehicle
npm run build:junit-global-vehicle
npm run build:tests-global-vehicle
npm run build:uat-global-vehicle
npm run build:prod-global-vehicle
```

## SMMX 前端脚本

仓库：`$env:WORKS_ROOT\dcs\dcs-smmx-vehicle-client`

开发：

```powershell
npm run dev:smmx-vehicle
npm run dev:smmx-vehicle-junit
npm run dev:smmx-vehicle-regression
```

构建：

```powershell
npm run build:test-smmx-vehicle
npm run build:junit-smmx-vehicle
npm run build:uat-smmx-vehicle
npm run build:prod-smmx-vehicle
```

## nginx 路由

根目录：

```nginx
root /app/html;
```

核心前端路径：

```nginx
location /global-vehicle/ {
  try_files $uri $uri/ /global-vehicle/index.html;
}

location /global-vehicle-tests/ {
  try_files $uri $uri/ /global-vehicle-tests/index.html;
}

location /global-vehicle-junit/ {
  try_files $uri $uri/ /global-vehicle-junit/index.html;
}

location /smmx-vehicle/ {
  try_files $uri $uri/ /smmx-vehicle/index.html;
}
```

## API 代理

```nginx
location /api/ {
  rewrite ^/api/(.*)$ /$1 break;
  proxy_pass http://172.29.10.9:8080;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
}
```

排查思路：

- 页面能打开但接口失败：先看前端请求是否是 `/api/xxx`，再看 nginx 是否 rewrite 到后端。
- 刷新页面 404：看对应 location 是否配置了 `try_files ... /项目/index.html`。
- 静态资源 404：看构建产物目录和 nginx location 名称是否一致，比如 `global-vehicle` 与 `smmx-vehicle`。
- HTTPS 问题：443 server 使用 `sit-dcs.smil.com`，证书路径为 `/etc/nginx/ssl/smil.com.crt` 和 `/etc/nginx/ssl/smil.com.key`。





