# 工具使用 - Ruby

有关概念概述（工具定义、工具选择、提示），请参阅 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用

Ruby SDK 通过原生 JSON Schema 定义支持工具使用，并提供了一个用于自动执行工具的 Beta 版工具运行器。

### 工具运行器（Beta 版）

```ruby
class GetWeatherInput < Anthropic::BaseModel
  required :location, String, doc: "城市和州，例如旧金山，加利福尼亚州"
end

class GetWeather < Anthropic::BaseTool
  doc "获取某个地点的当前天气"

  input_schema GetWeatherInput

  def call(input)
    "#{input.location} 的天气是晴朗，气温为 72°F。"
  end
end

client.beta.messages.tool_runner(
  model: :"claude-opus-5-5",
  max_tokens: 16000,
  tools: [GetWeather.new],
  messages: [{ role: "user", content: "旧金山的天气如何？" }]
).each_message do |message|
  puts message.content
end
```

### 手动循环

有关工具定义格式和代理式循环模式，请参阅 [共享工具使用概念](../../shared/tool-use-concepts.md)。

---