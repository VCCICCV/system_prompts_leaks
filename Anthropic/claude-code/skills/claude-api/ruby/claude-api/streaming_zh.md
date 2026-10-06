# 流式传输 - Ruby

## 流式传输

```ruby
stream = client.messages.stream(
  model: :claude-opus-5-5,
  max_tokens: 64000,
  messages: [{ role: "user", content: "写一首俳句" }]
)

stream.text.each { |text| print(text) }
```

---