# whitelist 同步与 MITM 审阅（2026-11-17）

## 同步结果和约束

- main 从 3ca7487 快进到 fmz200/wool_scripts 的 22db02f262145931b019c9e60f4e5a53edb85cea（17 个提交）；whitelist 正常合并 main，不重写历史。
- 定制模块 MITM 的 25 项、脚本触发条件和脚本条数不变。上游小红书新增 rnote.com，与现有规则相关的该项变更因超出白名单而不采用；不导入上游全量脚本。
- 三个已审阅的纯 JSON 脚本、共 25 条引用，改为 kleeu/wool_scripts 上述完整提交 SHA 的 raw 链接。固定提交避免未来同步 main 后未经审计的代码自动生效。源文件不改动。
- 删除定制模块中历史转换残留的 `hostname - reject`；修正文档中“reject 无风险”等绝对化表述。

## 审阅范围与发现

这是源码静态审阅和有限模拟验证，不是全部模块、全部依赖或实际设备上的安全认证。

| 源文件 | 结论 | 剩余问题 |
| --- | --- | --- |
| Scripts/smzdm/smzdm_ads.js | 可读；对响应 JSON 删除/过滤字段；未见主动联网、存储、动态执行 | 缺少 JSON/字段异常保护，结构变化会失败 |
| Scripts/smzdm/Smzdm.js | 可读；响应 JSON 本地处理；未见主动联网、存储、动态执行 | 递归未防 null，多处假定 data 存在；钱包接口也在触发范围内 |
| Scripts/zhihu/zhihu.js | 可读；响应 JSON 本地处理；未见主动联网、存储、动态执行 | 第 145 行引用未定义的 item，应为 i；若命中 market_card 分支可能异常，另有字段缺失风险 |
| Scripts/xiaohongshu/xiaohongshu.js | 主逻辑可读，但不应等同低风险 | 完整响应/缓存日志，图片视频信息持久化；Env 含联网、远程执行辅助方法（存在方法不等于实际调用）；还会改 red_id/作者字段。暂不迁移或扩围 |
| Scripts/baidumap.js | 含压缩的 protobuf/MD5 库，并非仅因为压缩就恶意或“加密” | 库包含代码生成等能力，需要核对来源及调用链；本次不声称完成依赖审计 |
| app2smile / kokoryh / zirawell / Keywos 外部脚本 | 现有引用保留，未增加 | 本仓库没有相应路径的完整可直接审阅副本；本次未拉取、未审计其远端当前内容，不应扩围 |

上述三个固定版本文件的 SHA-256：

```
611b9fa8cb1022622d1df9a2952b0f0fa1a001069b5666cce7e09b87dbec8dff  Scripts/smzdm/Smzdm.js
8e8b27474b277b334b1b37b14b412330485140b0df1cb77112f66fa043c550db  Scripts/smzdm/smzdm_ads.js
544fd88207b9d7005bb5cf427a1994028f91a95a65fde3db72a501983bff99f4  Scripts/zhihu/zhihu.js
```

## 是否放宽

可以逐项评估，但不建议现在整体扩围。可读源码、自有托管、固定版本降低供应链风险；它们不能消除 HTTPS 解密范围与业务数据暴露。

1. MITM 是主机级边界，不是 URL 路径级边界。精确脚本正则只限制脚本触发，不能把该主机其它路径排除在解密之外。`%APPEND%` 也意味着最终范围还取决于主配置及其他模块。
2. 现有 `interface*.music.163.com` 是通配范围。建议根据设备日志确认需要的 interface、interface3、interface9 等精确主机后收窄；不在本次擅自替换。
3. `qianbao.smzdm.com` 是优先考虑移除的现有项，其对应脚本只做钱包页面净化，收益不明显。移除之前确认是否依赖该功能。
4. 优先评估独立广告/静态资源主机，以及不需要 JS 的精确 reject/Map Local；能用 DOMAIN REJECT 完成的拦截无需新增 MITM。共享 CDN 不应整个后缀放行。
5. 若需要小红书搜索净化，可优先评估现有 edith.xiaohongshu.com 上的精确搜索接口，先精简/审计脚本，不必先增加主机。so.xiaohongshu.com 和 rnote.com 仅作为后续候选，尚未批准或启用。
6. 对 B 站等复杂 protobuf 脚本，先获得可追溯源码、核对打包产物与依赖，检查动态执行、联网、日志、持久存储，再固定自有提交。不能只把外部 URL 换成自己的仓库名。

不建议增加银行、支付、登录、密码管理器主机或 `*.com`/整站通配符。没有本机流量和功能验证，无法证明扩围的实际收益。

官方依据：[MITM 主机列表](https://manual.nssurge.com/http/mitm.html)、[HTTP 脚本触发](https://manual.nssurge.com/scripting/http-request.html)、[脚本 API 能力](https://manual.nssurge.com/scripting/api.html)。

## 验证与上游遗留问题

- 检查 main 与获取的上游提交一致；main 为 whitelist 的祖先。
- 定制模块 25 项 MITM 字符串与旧版完全一致；所有脚本除上述 URL 替换外保持原样；未增加第三方来源。
- 三个固定脚本通过 Node 语法检查，并模拟代表性广告过滤分支；不覆盖所有响应结构。
- 原有 verifyMerge.js：Loon、QuantumultX 通过；Surge、Shadowrocket 各有 4 行差异，涉及朴朴超市及同方全球人寿。main 与上游完全一致，这些差异属于同步得到的上游状态，未为了检查通过而改写 main 或重生成其他平台。
- 尚未在 Surge 设备导入并验证；现有 B 站/网易云模板参数未定义及第三方浮动链接仍是后续维护项。
