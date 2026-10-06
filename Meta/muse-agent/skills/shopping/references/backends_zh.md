# 购物后端

仅在需要特定于后端的过滤器时加载此文件。

## Facebook Marketplace

适用于本地、二手、自提、严格预算和寻宝类需求。

基础搜索流程：

```sh
MARKETPLACE_RESULTS_JSON=$(mktemp "${TMPDIR:-/tmp}/facebook-marketplace-search.XXXXXX")

facebook-cli marketplace search --query "<item>" --limit <N> --out "$MARKETPLACE_RESULTS_JSON"
```

常用参数：
- `--max-price` 和 `--min-price`（单位：美元）
- `--latitude` 与 `--longitude="<lng>"` 配合使用进行位置搜索；负坐标需用引号括起并加等号
- `--radius-in-miles` 指定本地搜索半径（英里）
- `--sort-by best_match|price_ascend|price_descend|creation_time_descend|distance_ascend` 设置排序方式
- `--allowed-item-conditions new,refurbished,used` 进行商品状况筛选
- `--delivery-method local_pickup_only|shipping_only|pickup_and_shipping` 设置配送方式
- `--max-listing-age-in-days` 限制 listing 发布时间
- `--limit <N>` 设置每页结果数（默认及最大值均为 20，超过上限会被截断）
- `--after <cursor>` 继续搜索——传入上一次响应中 `paging.cursors.after` 的值；若无 `paging` 字段，则表示已无更多结果

调用 `shopping.resolve_results`：

```json
{
  "result_paths": ["<MARKETPLACE_RESULTS_JSON>"],
  "selected_ids": ["listing-id-1", "listing-id-2"]
}
```

解析器会将 Marketplace 的 `listing_id` 映射为购物结果卡片。  
使用 `widget.create` 并指定 `kind: "shopping_results"` 和 `data.path` 来呈现返回的 `path`。

若需同时展示目录和 Marketplace 推荐：

```json
{
  "result_paths": ["<CATALOG_RESULTS_JSON>", "<MARKETPLACE_RESULTS_JSON>"],
  "selected_ids": ["catalog-id-1", "listing-id-1"]
}
```

使用 `widget.create` 并指定 `kind: "shopping_results"` 和 `data.path` 来呈现返回的 `path`。