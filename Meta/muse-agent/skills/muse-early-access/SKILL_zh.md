---
name: "muse_early_access"
title: "Muse 提前访问权"
description: "用于咨询关于Muse的通用抢先体验计划、申请加入该计划、查询或撤回加入申请，以及准入状态更新等相关问题。"
metadata: { "包含在提示中": 真 }
---
# Muse 提前访问计划

Muse 团队会从普通提前访问用户群中挑选成员参与新功能的早期测试。加入该计划并不保证可使用相关功能或享有测试优先权。

## 加入流程

仅在用户明确表示希望加入普通计划时才提交申请，即使特定功能是其加入的动机也不应例外。有关计划的疑问、测试特定功能的请求以及主动提供帮助的意愿均不属于加入申请。其他反馈请通过 `/opt/hatch/skills/muse-feedback/SKILL.md` 提交。

1. 阅读 `/opt/hatch/bin/feature-request show --help`，并检查 `show --kind missing-capability --subject early-access-group` 的内容。根据帮助信息说明现有状态。若已确认交付或已记录退出，则无需再次提交。对于尚未确认的请求，可在后续对话中重新提交加入申请；CLI 限制每日仅允许一次提交尝试。
2. 对于新的或未确认的请求，阅读 `/opt/hatch/bin/feature-request file --help`，并直接以 `missing-capability` 为类型、`early-access-group` 为主题提交申请。无需草拟或重复询问。在 `--summary` 中描述用户的兴趣，包括其加入的理由（在不涉及隐私细节的前提下）。将姓名和个人情况保留在本地的 `--context` 中。同时填写这两个字段，并根据当前消息的元数据设置 `--client-surface`；如无法获取则使用 `unknown`。无需再索要更多信息。
3. 以自己的语气简要回复，表达对其兴趣的认可。若 `sent_to_developers: true`，则确认请求已送达 Muse 团队；若仅 `delivery_confirmed` 为真，则请求此前已送达团队；若两者皆为假，则交付状态尚未确认。切勿暗示已被接纳，亦不要提及隐私声明、工单编号或回复免责声明，除非用户主动提出。

若因每小时提交次数限制而被拒绝，则未记录任何内容且未发送请求。对于其他错误，请按结果处理，但不要假设请求已被送达或已被接纳。请勿在此对话中再次提交申请，也不要承诺会在后台重试。

## 状态与退出

如需查询状态或了解是否已被接纳，请阅读 `/opt/hatch/bin/feature-request show --help`，并以相同的类型和主题进行查询。状态为 `addressed` 表示用户已加入该群体；其他状态均不确认其成员身份。请勿承诺具体时间安排或其他通知渠道。

在撤回申请之前，请阅读 `/opt/hatch/bin/feature-request delete --help`，并按照其中的审批步骤操作。请勿声称撤回申请会改变其成员身份。若日后再次申请加入，仍需用户提供明确的新指示。