---
name: openwrt-luci-dev
description: OpenWrt LuCI app development standards. Use for projects targeting OpenWrt 24.x/25.x主线.
---

# OpenWrt 24.x/25.x 主线开发约束

## 环境与 Shell
- Shell 为 `ash` (BusyBox)，**非 bash**。禁用 `[[ ]]`、数组、`source`（改用 `.`）。
- **文本处理**：BusyBox 的 `sed -i` 完全支持且保留软链接，**请直接使用 `sed -i`**。严禁用 `sed > tmp && mv` 重定向方式（会破坏 /etc/config 下的软链接）。
- **配置修改**：修改 UCI 配置文件（/etc/config/*）时，**必须使用 `uci` 命令**（`uci set/commit`），禁止直接用 sed 或 echo 覆写，否则守护进程无法感知变更。
- **资源限制**：避免在 Lua/JS 中处理超大字符串；避免在 LuCI 请求周期内高频调用 `os.execute()`。特别注意 25.x 中 Lua 协程内不要捕获大体积 `http.content()` 闭包。

## LuCI 开发模式（重要）
- **现代推荐**：前端使用 `/www/luci-static/resources/view/` 下的 **ES6 Class** 构建 UI（通过 `L.require("form").Form` / `L.require("grid").Grid`）。Lua 端仅负责定义路由（controller）和 RPC 声明。
- **遗留兼容**：若维护老代码使用 Lua CBI (`Map`/`TypedSection`)，语法保持不变，但官方不推荐用于新 App。
- **变量作用域**：Lua 中必须使用 `local` 声明变量。

## 前端 JS 规范 (现代)
- 模块加载：使用 `L.require("module")`，禁用全局变量。
- DOM 操作：使用 `L.dom` 或组件内置方法，**禁用** `document.getElementById` 和 jQuery。
- 通信协议：优先使用 **`L.rpc.declare()` 调用 ubus**（自动处理 token 和 session）。仅在调用非标准接口时降级使用 `L.request()`。
- 表单验证：使用 Form 组件的 `validate` 回调，返回 `true` 或错误字符串。

## 后端交互 (RPC / HTTP)
- **ubus RPC 优先**：在 `/usr/lib/lua/luci/controller/` 中声明 `entry()` 并关联 `call()` 方法，前端通过 `L.rpc.declare` 调用。
- **自定义 HTTP XHR**（备用）：若必须自建 API，后端使用 `luci.http` + `luci.jsonc`。
  - 取值：表单用 `http.formvalue()`，JSON 用 `http.content()` + `jsonc.parse()`。
  - 返回：必须 `http.prepare_content("application/json")` + `http.write_json()`，**严禁直接 `print`**。

## 调试与缓存刷新
- 修改 Lua 逻辑或 JS 资源后，执行 **`luci-reload`** 即可（清理 modulecache/indexcache 并重载索引）。
- 仅当修改了 `/usr/share/rpcd/acl.d/*.json` 时，才需额外执行 **`/etc/init.d/rpcd restart`**。
- 无需重启 `uhttpd`（`luci-reload` 会通过 ucode 热加载）。