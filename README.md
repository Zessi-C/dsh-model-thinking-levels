# @local/model-thinking-levels

在「设置 → 模型」的自定义模型 API 卡片里，为每个模型声明思考强度（Effort 档位）。
**默认已按通用档位预填，通常只需要点一次「保存」。**

## 它解决什么问题

DSH 判断一个模型要不要显示 Effort 行，只看该模型自己有没有 `reasoningEfforts` 元数据：

- 没有这个字段 → 没有任何推理元数据 → 输入框上不出现 Effort 行；
- `false` → 显式声明为「非推理模型」，同样没有 Effort 行；
- 一个字典（如 `{off: null, low: low, medium: medium, high: high}`）→ 出现 Effort 行。

自定义模型 API 表单只写 `id / name / contextWindow / maxTokens`，从不写这个字段，
所以手填的路由天然没有 Effort 行——这不是表单的 bug，而是表单的职责边界。

官方页面没有暴露这个字段，但它**声明了一个扩展位**：`settings.models.provider-card`。
这是一个 keyed 插槽，按设置命名空间（`llm-pi-ai`）分发，手填路由同样会渲染。
本插件就注册进这个插槽，读写则走 `ui-settings` 提供的 `configForms` 服务——
**和官方模型页用的是同一个共享表单**：读是响应式快照（不额外发请求），
写会折叠回页面自己的镜像，两边永远一致。

好处：不碰 app.asar，不动官方源码，DSH 升级不丢；写入的就是 YAML 里那份真实配置，
和手改 `cordis.patch.yml` 完全等价（表单后续保存也不会覆盖它——页面只写它自己看得见的字段）。

## 目录

| 文件 | 作用 |
| --- | --- |
| `package.json` | 声明 bundle（`dsh.bundle.patch`）与 Client 半（`dsh.client` + `exports["./client"]`） |
| `cordis.patch.yml` | Host 半：一行条目 |
| `index.js` | Host 半：探测通道（读路由、解析凭证、问网关、解析档位表） |
| `client.js` | Client 半：插槽注册、自动探测、预填、勾选、读写共享表单 |
| `tools/probe-levels.mjs` | 独立命令行探测工具，排查用，不参与插件运行 |

## 安装

插件必须装进 profile（会写 profile 的 `package.json` / 依赖并跑 pnpm），
所以走官方 GUI 流程，不要手改 profile 文件：

1. 打开 DSH 侧边栏 → **Plugins**
2. 点 **Add plugin**
3. 在输入框里填这个目录的**绝对路径**（即你 clone 下来的位置）：

   ```
   /absolute/path/to/dsh-model-thinking-levels
   ```

4. 点 **Install**（Host 会先读 `package.json` 做校验）
5. 装完点 **Enable now**

命令行等价写法：

```bash
dsh plugin --profile desktop add /absolute/path/to/dsh-model-thinking-levels
```

## 位置说明（为什么不在编辑器里面）

官方模型页只声明了两个扩展位，**都在卡片级**：

| 插槽 | 位置 |
| --- | --- |
| `settings.models.provider-card`（keyed） | 卡片内、`编辑` 编辑器**之前** |
| `settings.models.footer`（list） | 整个分区底部 |

`ProviderEditor` 内部没有任何 `renderSlot` 调用，所以「放进编辑器里面」在官方扩展点下**做不到**。
官方插件开发规范也明确禁止另一条路：

> Do not read another plugin's DOM, stylesheet, or component source to estimate placement;
> choose a slot that already allocates space.

即不许靠读 DOM 判断编辑器是否展开、或把节点搬进去。那种做法在 DSH 升级后必然失效。

所以这里取可行范围内最接近的做法：**同一张卡片内、紧贴编辑器上方的一个一行折叠块**。
默认收起时只占一行（和宿主自己的 `customized` 折叠行同款），不会给卡片添乱；
**只要还有模型没声明档位就自动展开**，所以新加的自定义路由依然是「点一次保存」；
保存后保持展开以便看到「已保存」并继续微调，点标题行可随时收起。

## 用法

设置 → 模型，找到 `cpa`（自定义模型 API）那张卡片，卡片里会出现一行
`▸ 思考强度  未声明`（或 `关闭 / 低 / 中 / 高`）：

- 点这一行展开/收起。展开后每个模型一行，**七个档位始终都在**（关闭 / 最小 / 低 / 中 / 高 / 极高 / 最大），
  面板不会在探测回来时把选项抽走；
- 每行右侧写着**这个模型**的网关声明，例如 `网关声明：低 / 中 / 高`——
  声明本来就是逐模型的，同一路由上两个模型可以不同，所以按行显示而不是给一个总体结论；
- 网关没声明的档位**照常可选，但画成虚线并变淡**，悬停说明
  「网关未声明此档位：多半会被归并到相邻档位，勾了也不会真正生效」；
- **面板打开时会自动探测网关**（每个模型一次极小的请求），
  把「未声明」的模型直接预填成网关声明的档位，此时显示「有未保存的改动」，
  点一次**保存**即全部写入。探测结果在本次会话内缓存，不会反复请求；
- 想给某个模型关掉 Effort 行（比如实测 `reasoning_effort` 无效的模型），
  把它的档位全部取消勾选，再保存——写入的是 `reasoningEfforts: false`（显式非推理模型），
  而不是删掉字段，所以下次不会又被预填回来；
- 只有当你偏离预填档位时，行下才会显示 `将写入: {...}`，方便核对真正落盘的值；
- 不信任缓存或想重测，点「重新探测」。

### 自动探测是怎么做的

DSH 侧没有任何能力信息源（详见下一节），所以插件直接**问网关**。
这需要凭证，而浏览器不能持有凭证，因此插件是**双半**的：

```
Client 半（浏览器）                    Host 半（进程内）
  点开面板
  └─ ctx.connection.rpc.call(          └─ 从 settings 服务读出该路由的
       '/dsh-model-thinking-levels',        baseURL 与 apiKeyEnv
       'probe',                             ↓
       { settingsNs, settingsPath,        ctx.credentials.resolve(apiKeyEnv)
         models })                        ↓
                                          对每个模型发一次 reasoning_effort=非法值
                                          ↓
                                          从 400 报错里解析 "valid levels: ..."
     ←── { results: [...] } ──────────────┘
```

**安全边界**：客户端只能给「设置命名空间 + 命名空间内的路径 + 模型 id」，
**URL 和凭证都由 Host 自己从 profile 文档里读**。所以这个通道没法被用来
让 Host 拿着你的 key 去请求任意地址；凭证也从不回传给浏览器。

探测失败（没配 key、网关不可达、不是表驱动网关）时，面板如实报错并退回显示全部七档，
不会假装知道。

### 新加的模型会自动出现

在官方编辑器里加完模型、点它的**保存**之后，这个面板会自己更新，**不需要刷新页面**：

```
官方编辑器保存 → 宿主改写该 namespace → 宿主 emit "settings/document-updated"
              → ui-settings 的共享 describe 镜像 reload → 面板重算草稿
```

`settings/document-updated` 是 `dsh-api-remotes` 里声明转发的远端事件
（`mode: 'emit'`），宿主在 namespace 原始配置变化时必定发出；
面板读的正是那个镜像，所以它跟着动。

⚠️ 但**顺序很重要**：先在官方编辑器里把模型列表**保存**，再用这个面板保存档位。
反过来的话，你那个还没保存的新模型不在文档里，面板的整数组替换自然带不上它，
随后官方编辑器再保存会因为 revision 过期而冲突。

### 顺序小结

1. 官方编辑器：填 API Key / Base URL / 模型列表 → **保存**；
2. 本面板：勾档位 → **保存**；
3. 回对话页，输入框上出现 Effort 行。

保存成功后 `~/.dsh/profiles/desktop/cordis.patch.yml` 的
`llm-pi-ai` → `providers` → `cpa` → `models` 下会多出：

```yaml
reasoningEfforts:
  off:            # 空值 = 支持关闭，且不发送思考字段
  low: low
  medium: medium
  high: high
```

回对话页，输入框上就会出现 **Effort** 行。

## 每个模型到底有哪几档？传错了会怎样？

**DSH 回答不了，但网关能。**

DSH 侧没有任何信息源：`reasoningEfforts` 是**你的**声明；已装 catalog 里没有自定义路由的条目；
`GET /v1/models` 只返回 `{id, object, owned_by}`；官方「拉取模型列表」用的 `llm.discoverModels`
也只返回 `{id, name?, contextWindow?, maxTokens?, inputModalities?}`。

但 **CPA 自己有一张 per-model 档位表**（`internal/registry/models/models.json` 里的 `thinking.levels`，
Antigravity home 模型还会向上游动态拉取），它用这张表校验 `reasoning_effort`，
**校验不过时就把整张表写进 400 报错里**：

```
level "bogus" not supported, valid levels: low, medium, high
```

所以**一个故意写错的请求就能问出权威答案**——不需要管理密钥，也不需要翻源码。
`tools/probe-levels.mjs` 就是这么做的：

```bash
node tools/probe-levels.mjs <baseURL> <model> [<model>...] [--reps 3]
# 例：node tools/probe-levels.mjs https://gateway.example.com/v1 model-a model-b
```

（另外两条路：`/v0/management/models`，需要 CPA 管理密钥——API key 不够；
或直接读源码里的注册表。前者返回的正是同一张表。）

### 一个实例上的实测结果

```
gemini-3.8-flash-high    网关接受: low, medium, high        ← 真能力（CPA 静态表）
  (不发送) 154     low  79     medium 239     high 329

deepseek-v4.1-flash      网关接受: low, medium, high        ← CPA 的默认值，假的
  (不发送)  64     low  45     medium  46     high  60
```

（`reasoning_tokens`，3 次取中位数。）

四个结论：

1. **`gemini-3.8-flash-high` 的真实档位就是 low / medium / high**，梯度清晰（79 → 239 → 329）。
   本面板预填的 `关闭 / 低 / 中 / 高` 对它**正好合适**。
2. **`deepseek-v4.1-flash` 的三档是 CPA 的默认值，不是它的能力**——它其实只有 `high` 一档
   （见下一节「自定义 provider 的表，可能是网关的默认值」）。实测三档 45 / 46 / 60 几乎一样，
   正是因为它们全被归并到 `high`。建议把它的档位全部取消勾选再保存 → 写成 `reasoningEfforts: false` →
   对话页不再给它显示一个没用的 Effort 行。
3. **表外的档位会被静默 clamp，不会报错。** 实测 `minimal` 被夹到 `low`（71 ≈ low 的 71），
   `xhigh` / `max` 被夹到 `high`（336 / 298 ≈ high 的 320）。
   所以在 DSH 里勾 `极高` / `最大` 是**有欺骗性的**：Effort 行会出现这些选项，
   但它们和 `高` 完全等价。网关这么做是因为跨协议族请求（openai → gemini）允许 clamp 而不允许失败。
4. **探测本身是逐模型的**，而且没有兜底默认——`no-such-model-xyz` 网关会回
   `unknown provider for model`。所以网关肯给出表，至少说明它认识这个模型；
   但「认识」不等于「表是真的」（见第 2 条）。

**传错了会怎样**——三种情况，只有第一种是显性的：

| 情况 | 结果 |
| --- | --- |
| 档位名根本不存在（如 `bogus`） | **HTTP 400，那一轮对话直接失败**，报错里附合法档位表 |
| 档位名存在、但该模型不支持（如 `minimal`） | 200，**被静默 clamp** 到最近的合法档位 |
| 合法、但模型本身不理会（如 deepseek 的 `high`） | 200，什么也没发生 |

### ⚠️ 自定义 provider 的表，可能是网关的默认值

**这是本插件最容易让人误判的一点，面板的措辞也为此改成了「网关接受」而不是「网关声明」。**

CPA 的静态注册表（`internal/registry/models/models.json`）里的表是**真的逐模型能力**，
和网关回答逐条对得上（实测 `gemini-3.8-flash-high` → `low,medium,high`、
`gemini-3.6-flash-high` → `minimal,low,medium,high`、`gemini-3.1-flash-image` → `minimal,high`）。

但**配置里手写的自定义 provider 模型不是**。CPA 的 `config_types.go` 写着：

```go
// Thinking configures the thinking/reasoning capability for this model.
// If nil, the model defaults to level-based reasoning with levels ["low", "medium", "high"].
Thinking *registry.ThinkingSupport `yaml:"thinking,omitempty"`
```

也就是说：**`thinking` 没写，CPA 就报 `low, medium, high`**——这是它的默认值，不是模型的事实。

一个真实遇到的例子（下面用通用名字代指各层）。完整链路：

```
上游服务                  supportedEfforts: ["high"]              ← 真正的能力
   ↓
自建反代                  /v1/models 原样暴露该字段               ← 数据是有的
   ↓
CPA 接入（config.yaml）    该模型没写 thinking  →  CPA 默认 ["low","medium","high"]   ← 信息在这里丢了
   ↓
本插件的探测             如实报告 CPA 说的 → 面板显示「网关接受：低 / 中 / 高」
```

所以对这个模型来说，`低/中/高` 是**假的**——它其实只有 `high` 一档，
而且 `only_reasoning: true`（永远思考，关不掉）。实测三档 `reasoning_tokens` 是 62/59/55，
几乎一样，正是因为它们**全被归并到 `high`**（反代日志里能看到
`reasoning_effort floored model=... medium -> high`）。

**要让它变准，在网关侧显式声明**（CPA 的 `config.yaml`）：

```yaml
    models:
      - name: upstream-model-id
        alias: exposed-model-id
        thinking:
          levels: [high]
```

`ThinkingSupport` 的完整字段是 `min` / `max` / `zero-allowed` / `dynamic-allowed` / `levels`。
改完重启 CPA，面板再探测就会显示「网关接受：高」。

**判断方法**：如果某个自定义模型的档位表恰好是 `low, medium, high`，先怀疑它是默认值——
去问上游的 `/v1/models`（有些反代会带 `reasoning_supported_efforts` 这类字段），
或者去网关配置里确认有没有显式 `thinking`。

### ⚠️ 「关闭」不等于不思考

pi-ai 的 `off` 档语义是「不发送思考字段」（`off: null` 在 `thinkingLevelMap` 里整个缺席），
由网关自己决定默认值。这台网关的缺省值是 **154 token，接近 medium**——因为 CPA 缺省时走 `auto` → mid-range。

想要真正最低的思考量，选 **`低`**（79），那是它的地板。

## 设计要点

- **档位表来自网关，不是猜的**：`reasoning_effort` 传一个非法值，CLIProxyAPI 类的网关会把
  自己的 `thinking.levels` 表写进 400 报错里（`valid levels: low, medium, high`）。
  一次请求就能拿到权威答案，不需要管理密钥。声明是**逐模型**的，所以按行显示。
- **表外的档位仍然可选，只是被标记出来**：网关会把表外档位**静默归并**到相邻档位，
  所以勾了它不会报错、也不会生效。抽走这些选项看似更干净，实际上会让面板的调色板
  随探测结果变化、也让已经声明过的档位无处取消，所以改成虚线 + 变淡 + 悬停说明——
  信息给足，选择权留给用户。
- **双半分工**：Client 半做 UI，Host 半做需要凭证的网络调用。
  Client 只能指定「命名空间 + 路径 + 模型 id」，**URL 与凭证由 Host 自己从 profile 读**，
  凭证永不回传浏览器。这样这个通道不能被当成 SSRF 用。
- **只对自定义路由生效**：面板仅在 pi-ai 适配器把路由标为 `declared`（没有任何已装 catalog 描述它）时渲染。
  对 catalog 路由写 `models` 会整体替换掉 catalog 里的模型列表，所以这里刻意不碰。
- **`off` 写 `null`**：pi-ai 的「支持不思考，且不发送任何思考字段」语义（不是发 `none`）。
- **清空写 `false`，不删字段**：`false` 是 pi-ai 认可的「非推理模型」声明，
  既让 Effort 行消失，又让面板知道这是你的决定、不再预填。空字典 `{}` 是非法的，绝不写。
- **上送值 = 档位名**：对 CPA 这类 OpenAI 兼容网关（`thinkingFormat: "openai"`，发顶层 `reasoning_effort`）成立。
  个别网关需要别的拼写（如 `xhigh` → `x-high`）时，仍需在 YAML 里手写该模型。
- **写入是整数组替换**：`{op:"set", path:["providers","cpa","models"]}`，与官方表单保存用的是同一套路径操作；
  值从共享表单快照读出后只改 `reasoningEfforts`，其余字段原样回写。
- **没有自己的异步读**：早期版本自己调 `remote.settings.describe()`，一旦这次调用被拒绝或迟迟不答，
  面板就永远停在「载入中…」。现在读的是 `configForms` 暴露的**共享 describe 镜像快照**——
  和官方模型页渲染的是同一份数据，不产生独立请求，也不做二次 schema 解码
  （否则解码失败同样会一直 loading）。写入仍走共享表单的队列，revision 与冲突由它处理；
  读取确实没答上时面板显示「载入中…」并给出「重新载入」，不会静默卡死。
- **样式是照抄宿主再改前缀**：官方规范允许（并要求）这么做——
  「copy markup, CSS, and behavior from the primitive into the plugin …
  Rename copied classes under your plugin's prefix, keep only `--dsw-alias-*` token references」。
  本插件的 `CSS` 常量逐条抄自 `@deepseek-ai/dsh-client-ui-settings-models/lib/client.js` 里的
  `.xuTnra_rowCard` / `.xuTnra_customized` / `.xuTnra_customizedSummary` / `.xuTnra_rowTag` /
  `.xuTnra_secondaryButton` / `.xuTnra_primaryButton` / `.xuTnra_linkButton` / `.xuTnra_advancedHint` /
  `.xuTnra_error`，类名改为 `dsh-tl-*` 前缀，字面色全部换回同一批主题变量。
  尺寸因此和宿主一致：折叠行 12px/500、档位胶囊 = `rowTag`（11px/16px，`radius-xs`，`.5px` 描边）、
  按钮 28px 高 / `radius-sm` / 12px、分隔线 = `.5px solid var(--dsw-alias-border-l2)`。
  样式以 React 元素（`<style>`）随组件渲染，卸载即移除，不向 `document.head` 写任何东西；
  hover / focus-visible / disabled 状态也因此能用真 CSS 表达，而不是内联样式做不到的伪类。
- 只使用 `--dsw-alias-*` / `--dsw-radius-*` / `--dsw-focus-ring-color` 主题变量，文案走 `ctx.locale`（中英双语），
  不 require 任何 Harness 客户端包，不写 DOM、不读别的插件 DOM。

## 迭代

pnpm 以 `link:` 依赖安装本地目录（已确认 `node_modules/@local/model-thinking-levels`
是指向本目录的软链），所以：

| 改了什么 | 怎么生效 |
| --- | --- |
| `client.js`（Client 半） | **刷新页面**。Client bundle 的缓存键由文件的 mtime/ctime/size 算出，内容一变 URL 就变，不用清缓存。 |
| `index.js`（Host 半） | **重启 DSH**。它是进程内的 JS 模块，Node 的 ESM 模块缓存不会因为文件变了就重新加载。 |
| `cordis.patch.yml` | 热重载（HMR 同时监听 profile 与 home 的 patch）。 |

⚠️ **Host 半没重启时的症状**：面板能正常显示，但会报
`探测失败：transport failure for /dsh-model-thinking-levels/probe: HTTP 405`。

这不是权限或方法问题——DSH 的 web server 对**未知路径的 POST 就返回 405**（GET 才是 404），
所以 405 的含义是「这个 RPC 路由根本没注册」。可以直接验证：

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://127.0.0.1:19387/dsh-model-thinking-levels/probe
# 405 = Host 半没加载（重启）；404 = 路由在但方法不匹配
```

## 卸载

Plugins 页 → 找到该 bundle → 卸载（Host 会还原 `package.json` 与 lockfile）。
已写入的 `reasoningEfforts` 保留，需要的话在模型卡片里取消勾选保存即可清掉。

## 不装插件时的等效写法

临时用的话，直接在手填路由的 `models[]` 里加上这个字段，效果完全一样：

```yaml
- id: llm-pi-ai
  config:
    providers:
      my-gateway:
        displayName: My Gateway
        api: openai-completions
        baseURL: https://gateway.example.com/v1
        apiKeyEnv: MY_GATEWAY_KEY
        models:
          - id: model-a
            name: Model A
            reasoningEfforts:      # 只声明网关真正接受的档位
              off:                 # 空值 = 支持关闭，且不发送思考字段
              low: low
              medium: medium
              high: high
          - id: model-b
            name: Model B
            reasoningEfforts: false   # 实测 reasoning_effort 无效的模型，
                                      # 显式声明为非推理模型
```

> profile 里 `llm-pi-ai` 若被 home patch 覆盖，配置编辑器会拒绝保存——那种情况下
> 插件写入也会被同样拒绝，面板会显示「保存被拒绝」。
