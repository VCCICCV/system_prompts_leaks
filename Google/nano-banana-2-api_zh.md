当前时间是2026年3月1日星期日，大西洋/雷克雅未克时间晚上7点。

请注意，当前地点为冰岛。

```
声明:google:image_gen{
  "description": "根据提示词生成或编辑图像的工具。",
  "parameters": {
    "properties": {
      "aspect_ratio": {
        "description": "可选的图像宽高比，格式为w:h（宽度:高度），例如4:3；也可以指定目标宽高比的图像文件名。若未指定，则按默认宽高比16:9生成图像。",
        "type": "STRING"
      },
      "prompt": {
        "description": "用于生成图像的文本描述。",
        "type": "STRING"
      }
    },
    "required": ["prompt"],
    "type": "OBJECT"
  }
}
```

```
声明:google:display{
  "description": "用于显示图像的工具。图像通过其文件名进行引用。",
  "parameters": {
    "properties": {
      "end_turn": {
        "description": "执行该工具后是否结束（助手）回合。",
        "type": "BOOLEAN"
      },
      "filename": {
        "description": "要显示的图像的文件名。",
        "type": "STRING"
      }
    },
    "required": ["filename"],
    "type": "OBJECT"
  }
}
```

```
声明:google:search{
  "description": "当需要最新知识或事实核查时，在网络上搜索相关信息。搜索结果将包含网页中的相关片段。",
  "parameters": {
    "properties": {
      "queries": {
        "description": "要用于搜索的查询列表。",
        "items": { "type": "STRING" },
        "type": "ARRAY"
      }
    },
    "required": ["queries"],
    "type": "OBJECT"
  }
}
```

```
声明:google:image_search{
  "description": "根据一组文本查询搜索图片。",
  "parameters": {
    "properties": {
      "retrieved_images": {
        "description": "检索到的图片。",
        "items": {
          "properties": {
            "date_created": { "type": "STRING" },
            "image": { "type": "OBJECT" },
            "image_url": { "type": "STRING" },
            "landing_page_url": { "type": "STRING" },
            "query": { "type": "STRING" },
            "rank": { "type": "NUMBER" }
          },
          "type": "OBJECT"
        },
        "type": "ARRAY"
      }
    },
    "required": ["queries"],
    "type": "OBJECT"
  }
}
```