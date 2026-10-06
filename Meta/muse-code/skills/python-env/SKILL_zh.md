---
name: python-env
description: 在 Python 环境中进行设置或安装时，无论是否阅读正文，都有一条通用原则：环境应与项目绑定，因此应在项目目录内创建（例如 .venv），或交由 uv 来管理；切勿将其置于诸如 /tmp 之类的临时目录中，也绝不要强制安装到系统解释器中。在为 Python 项目创建虚拟环境、选择安装工具或编写运行指令之前，请先阅读并理解正文内容。
user-invocable: false
---
# Python 环境

环境是项目的一部分，而不是临时空间。明天打开该项目的用户，或者在另一台机器上克隆该项目的用户，都应该能在其编辑器、工具链和习惯所期望的位置找到该环境。

## 环境应放置在哪里

请将其创建在**项目目录内**——位于项目根目录下的 `.venv` 是几乎所有编辑器和工具都能自动识别的约定位置：

```
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
```

对于全新项目，建议使用 `uv`，它会为您管理一个项目本地的 `.venv`，且速度更快：

```
uv venv
uv pip install -e .      # 或者在有锁文件时使用 `uv sync`
uv run python -m yourpkg # 在项目环境中运行，无需激活
```

切勿将环境放在 `/tmp`、`~/envs` 或项目之外的任何其他临时位置。这些位置对用户的工具链不可见，用户也不会去那里查找，而且放在 `/tmp` 上还会被系统随时删除。

## PEP 668：“外部管理的环境”

Homebrew、Debian 和 Ubuntu 会将系统解释器标记为“外部管理”，因此在虚拟环境之外执行 `pip install` 时会报错：

```
error: externally-managed-environment
```

这正是提示您应该创建项目环境的信号，而不是需要绕过的障碍。**不要**使用 `--break-system-packages` 参数、设置 `PIP_BREAK_SYSTEM_PACKAGES` 环境变量，或删除 `EXTERNALLY-MANAGED` 标记：这些操作会修改由操作系统管理的解释器，导致项目没有自己的环境，并可能破坏机器上的其他软件。

## 提供用户可直接运行的命令

编写针对项目环境的运行指令，使其在项目目录下的新 shell 中即可正常工作：

```
source .venv/bin/activate
python -m yourpkg ...
```

或者，使用 `uv` 时，可以运行 `uv run python -m yourpkg ...`。

不要返回指向项目外环境的绝对路径（如 `/tmp/whatever/bin/python -m yourpkg`）。即使当前能正常工作，也会让用户误以为他们的项目没有自己的环境。

## 尊重已有的配置

如果项目中已经存在环境或已声明的工具——例如 `.venv`、`uv.lock`、Poetry、Pipenv 或 conda——请优先使用它们，而不是再引入第二个环境。创建之前务必先检查。