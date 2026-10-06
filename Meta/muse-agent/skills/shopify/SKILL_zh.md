---
name: "shopify"
description: >
  使用 Shopify 官方 MCP 服务器设置并运营 Shopify 商店（
  已安装 `shopify` 命令，而非 npm 版的 Shopify CLI）。当有人希望……时使用。
  无论是在线销售、开设商店或企业，还是管理 Shopify 商店——甚至
  如果他们不说“Shopify”：“在线销售蜡烛”、“开设我的第一家商店”，
  “添加商品”、“查看我的订单”、“设置库存”、“设置折扣”
  “显示我的销售”。在连接之前，它可以显示相关的模拟店铺样品。
  目录、建议企业名称、查询域名、搜索 shopify.dev，以及
  验证 GraphQL。连接流程可以为新用户创建账户并存储数据。
  用户。关键词：Shopify MCP、电子商务、样品商品、入门目录、
  ShopifyQL，第一个商店，创建产品。
icon: "shopify"
metadata: { "不包含在提示中": 假 }
---
# Shopify

已安装的 `shopify` 命令封装了 Shopify 官方的 MCP 服务器。它不是来自 npm 的 Shopify CLI；该命令仅包含以下四个子命令。工具名称和架构均来自 `shopify list-tools`，而非凭记忆判断——其他 Shopify 连接器会暴露一些本工具不提供的工具。

```text
shopify status
shopify authorize-url
shopify list-tools
shopify call-tool --name <tool-name> --arguments-json '<JSON 对象>'
```

## 连接流程

1. 执行 `shopify status`。如果已有商店已连接，则可直接开始工作。
2. 否则，执行 `shopify authorize-url`，并将返回的 `connect_url` **仅** 提供给用户。响应中的其他内容均无需向用户展示。
3. 用户完成连接后，再次执行 `shopify status`，然后调用 `call-tool --name get-shop-info` 确认当前连接的商店。

在商店未连接之前，`search_docs_chunks`（搜索 https://shopify.dev）、`validate_graphql_codeblocks`、`find-mock-shop-catalogs`、`generate-domain-names` 和 `generate-business-names` 等功能均可使用。`get-storefront-generation` 仅能通过 Shopify 商店前台生成组件创建的 `generationUUID` 使用；`claim-storefront-preview` 是组件回调，不得直接调用。除上述工具外，包括目录导入在内的所有其他功能均需要商店连接。请勿仅为探索示例目录、名称、域名或文档而要求用户进行连接。

## 示例目录

当用户准备开设商店时，在要求其连接前，可先提供相关的 mock.shop 示例目录：

1. 使用用户描述的商品类别调用 `find-mock-shop-catalogs`。若用户尚未说明其经营品类，请先询问此问题。
2. 展示匹配度最高的目录及其简介、商品与系列数量、币种以及 `storefrontUrl`。明确标注每个链接为示例商店来源，并询问用户希望选择哪个目录。
3. 连接成功后，在调用 `import-mock-shop-catalog` 导入所选目录前，请先确认。
4. 通过 `search_products` 和 `search_collections` 核实导入结果，报告创建、复用、跳过、未发布或部分成功的记录，以及任何币种不匹配的情况。

导入操作会复制每个目录中的前两个系列及最多八件商品，总计不超过十六件商品，并将新导入的记录发布到在线商店。重复导入是安全的：它会补充缺失的记录和关联关系，但不会覆盖已导入的商品或商家编辑的内容。若 Shopify 返回 `partial: true`，请稍等片刻后重新执行相同导入。这些均为供上线前编辑或替换的示例数据，而非供应商库存。

## 尚无商店的用户

对于从未使用过 Shopify 的用户，相同的 `connect_url` 同样适用。Shopify 登录页面允许他们创建账户并开设新店，随后会将他们重定向回连接流程。请提前告知用户这一点，以免他们先单独创建商店。连接成功后，可按以下步骤引导其完成首次开店流程：

1. 调用 `get-shop-info` 获取商店名称、币种及套餐信息。
2. 导入用户选定的示例目录，或使用 `create-product` 添加用户自有的商品（标题、描述、价格及图片 URL）。请询问用户经营什么商品，切勿自行编造商品目录。
3. 调用 `create-collection` 创建系列，并使用 `add-to-collection` 将商品归类。
4. 若用户需管理库存，请先调用 `get-inventory-levels` 获取库存项、仓库位置及当前数量，再调用 `set-inventory` 设置库存。
5. 如有需要，可调用 `create-discount` 创建开业促销活动。

商业名称和域名的结果中可能包含用于开设商店的 `signupUrl` 链接。这些仅供探索之用；当用户准备将 Muse 连接到其商店时，请改用连接器的 `connect_url`。

对于本服务器无法支持的功能，例如主题、结账、支付、配送以及域名绑定或购买，请引导用户前往 Shopify 管理后台（即 `get-shop-info` 返回的域名）。

## 常见任务

| 任务 | 工具 |
| --- | --- |
| 商店详情 | `get-shop-info` |
| 商业名称与可用域名 | `generate-business-names`、`generate-domain-names` |
| 示例商品目录 | 先使用 `find-mock-shop-catalogs`，再用选定的 `subdomain` 调用 `import-mock-shop-catalog`；此操作导入的是目录数据，而非店铺设计 |
| 店铺预览 | 仅对 Shopify 小组件已生成的预览调用 `get-storefront-generation` |
| 商品 | `search_products`、`get-product`、`create-product`、`update-product`、`bulk-update-product-status` |
| 商品系列 | `search_collections`、`get-collection`、`create-collection`、`update-collection`、`add-to-collection` |
| 库存 | `get-inventory-levels`、`set-inventory` |
| 订单与客户 | `list-orders`、`get-order`、`list-customers` |
| 折扣 | `create-discount` |
| 报表与趋势 | `run-analytics-query`（ShopifyQL；说明中附有示例） |
| 管理后台中的其他功能 | 按照以下顺序依次调用：`graphql_schema` → `validate_graphql_codeblocks` → `graphql_query` 或 `graphql_mutation`，每次均需按此顺序执行 |
| Shopify 的工作原理 | `search_docs_chunks` |
| 切换至另一家商店 | 先调用 `switch-shop`，再调用 `get-shop-info` |

## 规则

- 已认证的命令会自动刷新过期的访问令牌。对于正常过期的情况，无需提示用户断开连接后再重新连接。如果 Shopify 报告刷新授权已被撤销或需要重新授权，请运行 `shopify authorize-url`；除非返回结果明确指出必须先断开连接，否则无需事先要求断开。
- 除了检查命令状态外，还应仔细查看返回结果：即使 MCP 响应显示成功，Shopify 也可能报告工具层面的失败。
- 在执行任何写入类操作前——包括创建、更新、设置、批量操作、折扣、目录导入、预览申领以及 `graphql_mutation`——务必先征得用户同意。目录导入会将示例商品和商品系列发布到在线商店；如遇部分导入或币种不匹配（价格直接复制且未进行换算），请予以说明。
- 目录导入不会更改商店名称、主题、导航、首页区块或现有商品。`find-mock-shop-catalogs` 返回的 `storefrontUrl` 预览的是源模拟目录，而非导入后的目标商店。切勿声称目标店铺外观会与该预览一致，亦不得以该预览首页作为依据。导入完成后，请通过 `search_products` 和 `search_collections` 核实新增记录，并告知商家：若希望首页展示导入的目录内容，还需在 Shopify 管理后台自行配置主题。
- 执行 `switch-shop` 后，必须紧接着调用另一个工具（即所需执行的操作，或再次调用 `get-shop-info`）以完成切换。
- `get-storefront-generation` 仅用于轮询由 Shopify 店铺预览小组件已创建的 `generationUUID`；它无法启动新的预览生成，也无法修改现有店铺。`claim-storefront-preview` 仅由该小组件调用，模型不得直接调用。这两个工具均不会将模拟目录的主题应用到已连接的商店。
- Muse 展示的是结构化数据，而非 Shopify 的各类小组件。回复时请对结果进行总结，切勿提及卡片、图表或用户无法看到的预览。