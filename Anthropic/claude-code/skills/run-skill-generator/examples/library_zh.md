# 示例：库/SDK

从流程的角度来看，库并没有“运行”这一步——没有需要启动的服务器，也没有需要调用的命令行工具。对于库而言，“运行”技能主要涉及：

1. **从源码构建**库
2. **运行测试套件**
3. **一个最小的可运行示例**，用于调用库并验证其已正确安装

请保持简明。模板中的“构建”和“测试”部分已经完成了大部分工作。

## 烟囱测试示例

针对库的主要补充是一个小型程序（或 REPL 片段），它导入库并执行一项实际操作。这是代理确认“是的，该库可以使用”的方式：

> ## 验证
>
> ```bash
> python -c '
> from mylib import Client
> c = Client()
> print(c.ping())
> '
> # -> pong
> ```

或者对于编译型语言：

> ```bash
> cat > /tmp/smoke.go <<GO
> package main
> import "example.com/mylib"
> func main() { println(mylib.Version()) }
> GO
> go run /tmp/smoke.go
> # -> v1.2.3
> ```

## 示例片段

> ---
> 名称: run-mylib
> 描述: 从源码构建、安装并测试 mylib。当需要验证 mylib 是否可用、运行其测试或构建发布包时使用。
> ---
>
> `mylib` 是一个 Python 库——“运行”它意味着从源码构建并执行测试套件。
>
> ## 准备
>
> ```bash
> pip install -e '.[dev]'
> ```
>
> ## 验证
>
> ```bash
> python -c 'import mylib; print(mylib.__version__)'
> # -> 2.1.0
> ```
>
> ## 测试
>
> ```bash
> pytest
> ```
>
> 测试子集：`pytest tests/unit/`。带覆盖率：`pytest --cov=mylib`。
>
> ## 构建（发布包）
>
> ```bash
> pip install build
> python -m build
> # -> dist/mylib-2.1.0-py3-none-any.whl
> ```

## 需要考虑记录的内容

- **开发模式与安装模式的区别。** `pip install -e .` 与 `pip install .`——如果行为不同，请说明在什么场景下使用哪种方式。
- **可选依赖。** `[dev]`、`[test]`、`[docs]` 等 extras 以及何时需要它们。
- **生成代码。** 如果存在代码生成步骤（如 Protocol Buffers、OpenAPI 客户端），请予以说明——这类信息几乎总是缺失于 README 中。