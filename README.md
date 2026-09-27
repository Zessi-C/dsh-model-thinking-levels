# dsh-model-thinking-levels

给 DSH 的自定义模型 API 路由补上「思考强度」设置。

> Unofficial DSH plugin. Per-model thinking levels for custom `llm-pi-ai` routes, with gateway-side capability probing. MIT.

## 功能

- 在「设置 → 模型」的自定义 API 卡片里勾选每个模型支持的思考档位，写入该路由的 `reasoningEfforts`；
- 打开面板时**自动探测网关接受哪些档位**（每个模型一次极小请求），并据此预填。网关没声明的档位仍然可选，但画成虚线，悬停提示「会被归并到相邻档位、勾了不生效」；
- 未声明的模型预填成网关的档位，通常点一次「保存」就完事；把档位全部取消勾选则写入 `reasoningEfforts: false`（显式非推理模型）。

## 安装

```bash
dsh plugin --profile <profile> add github:Zessi-C/dsh-model-thinking-levels
```

也可以从本地目录装：`dsh plugin --profile <profile> add /absolute/path/to/dsh-model-thinking-levels`。

装完**重启 DSH**（Host 半是进程内模块，只刷新页面不生效）。

## 用法

设置 → 模型 → 你的自定义 API 卡片，里面会出现一行 `▸ 思考强度`：

1. 点开这一行。每个模型一行，右侧写着网关接受的档位，例如 `网关接受：低 / 中 / 高`；
2. 勾选档位 → **保存**；
3. 回对话页，输入框上出现 Effort 行。

想让某个模型不显示 Effort 行：把它的档位全部取消勾选再保存。

## 命令行探测

面板的自动探测用的是同一个办法：给 `reasoning_effort` 传一个非法值，表驱动的网关会把
自己的档位表写进 400 报错里（`valid levels: low, medium, high`）。也可以单独跑：

```bash
node tools/probe-levels.mjs <baseURL> <model> [<model>...]
```

**一个坑**：CLIProxyAPI 对配置里手写的自定义 provider 模型，若模型条目没写 `thinking`，
会**默认**按 `low, medium, high` 校验——这是网关的默认值，不是模型的能力。
探测到恰好这三个档位时先怀疑是默认值，去上游 `/v1/models` 或网关配置里确认。
在网关侧显式声明即可修正：

```yaml
    models:
      - name: upstream-model-id
        alias: exposed-model-id
        thinking:
          levels: [high]
```

## 不装插件时的等效写法

```yaml
- id: llm-pi-ai
  config:
    providers:
      my-gateway:
        baseURL: https://gateway.example.com/v1
        apiKeyEnv: MY_GATEWAY_KEY
        models:
          - id: model-a
            reasoningEfforts:      # 只声明网关真正接受的档位
              off:                 # 空值 = 支持关闭，且不发送思考字段
              low: low
              medium: medium
              high: high
```

## 卸载

```bash
dsh plugin --profile <profile> remove @local/model-thinking-levels
```

## License

MIT
