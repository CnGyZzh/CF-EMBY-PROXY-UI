# `测试/worker.js` 主流程图

```mermaid
graph TD
    A[开始：Cloudflare Worker 收到请求或定时事件] --> B{入口类型}

    B -- HTTP fetch --> C{fetch(request, env, ctx)<br>路径与方法分发}
    B -- scheduled --> Z[scheduled(event, env, ctx)]

    C -- GET / --> D[renderLandingPage<br>返回首页]
    C -- GET /admin --> E[返回 UI_HTML 管理面板]
    C -- OPTIONS /admin 或 /api --> F[返回 CORS 预检响应]
    C -- POST /api/auth/login --> G[Auth.handleLogin]
    C -- POST /admin --> H{Auth.verifyRequest<br>JWT/Cookie 校验}
    C -- 命中节点名前缀 --> I[Database.getNode<br>读取节点配置]
    C -- 其他路径 --> Y[返回 404 Not Found]

    G --> G1{环境与请求是否合法}
    G1 -- 否 --> G2[返回 400 或 503]
    G1 -- 是 --> G3{密码正确且未达到锁定阈值}
    G3 -- 否 --> G4[写入失败次数<br>返回 401 或 429]
    G3 -- 是 --> G5[生成 JWT Cookie<br>返回登录成功]

    H -- 否 --> H1[返回 401 未授权]
    H -- 是 --> J[Database.handleApi]
    J --> J1{action 是否命中 ApiHandlers}
    J1 -- 否 --> J2[返回 Invalid Action]
    J1 -- 是 --> J3[执行管理动作<br>仪表盘/配置/节点/日志/Telegram/CF 清缓存]
    J3 --> J4[normalizeJsonApiResponse<br>返回 JSON]

    I --> I1{节点存在且 secret 校验通过}
    I1 -- 否 --> Y
    I1 -- 是 --> K[Proxy.handle]

    K --> K1[读取运行时配置<br>识别代理路径与请求特征]
    K1 --> K2{CORS/IP/地区/限流 是否通过}
    K2 -- 否 --> K3[返回 403 或 429]
    K2 -- 是 --> K4[清洗 Header<br>补齐 Emby 授权头]
    K4 --> K5[识别预热/图片/字幕/Manifest/分段/大文件/WebSocket]
    K5 --> K6[构建上游 fetch 选项<br>确定重试目标]
    K6 --> K7[请求源站<br>处理 403 降级与 5xx 重试]
    K7 --> K8{上游发生重定向}
    K8 -- 是 --> K9[跟随或改写 Location<br>必要时改为直连跳转]
    K8 -- 否 --> K10[继续处理响应]
    K9 --> K10
    K10 --> K11[改写响应头/CORS/缓存策略]
    K11 --> K12[Logger.record<br>记录访问日志]
    K12 --> K13{命中 Range 预热探测}
    K13 -- 是 --> K14[ctx.waitUntil 异步预取后续 Range]
    K13 -- 否 --> K15[返回最终响应]
    K14 --> K15

    Z --> Z1{KV 可用}
    Z1 -- 否 --> Z5[结束]
    Z1 -- 是 --> Z2[读取配置]
    Z2 --> Z3{D1 可用}
    Z3 -- 是 --> Z4[清理过期 proxy_logs]
    Z3 -- 否 --> Z6{已配置 Telegram}
    Z4 --> Z6
    Z6 -- 是 --> Z7[Database.sendDailyTelegramReport]
    Z6 -- 否 --> Z5
    Z7 --> Z5[结束]

    E --> H
    D --> C
```
