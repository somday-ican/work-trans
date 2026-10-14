# project-handoff

**Hand off a session to any agent, on any model — with a map of the project.**
把当前会话交接给任意 agent 或模型，生成的不是聊天记录摘要，而是一张项目地图。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
Works with Claude Code · Codex · Kimi · DeepSeek · Cursor · WorkBuddy · anything that reads Markdown

English | [中文](#中文说明)

---

## The problem

Long sessions rot. After a few hours the context is full of dead ends: files opened and rejected, approaches tried and rolled back. The agent carries all of it, and its answers get worse because the useful state is buried under stale exploration.

Your options today are all bad:

- `/clear` — throws away the state along with the noise.
- `/compact` — lossy, invisible, and you don't control what survives.
- Fork the old conversation — carries every redundant byte into the new session. (One public report: single requests ballooning to ~20.9 MB, first-token waits of 97 seconds.)
- Start fresh — re-explain everything by hand.

And when the next session is a **different agent on a different model** (Claude → Codex → Kimi → DeepSeek), none of the built-in mechanisms help at all. Existing handoff skills assume the same client on both ends and store documents inside one client's private namespace (`.claude/`, `.cursor/`), where the receiving agent can't see them.

## What this skill does

`project-handoff` compacts the current conversation into a structured, plain-Markdown handoff document **stored as a machine-local file any agent can read**, referenced by absolute path. It is built for long-horizon work — research projects, paper writing, multi-week engineering — where the receiving agent needs to know:

1. **What every file in the project is and does** (a file manifest with current status)
2. **How far the work has progressed** — completed-and-verifiable vs. open loops, each tagged with its blocker (waiting on a person / an external reply / a resource / a decision)
3. **Where the project is heading** — near-term and mid-term direction
4. **Who is responsible for what** — division of labor across agents and humans
5. **The single next concrete action** — not a to-do list, one executable step

Other handoff skills snapshot a *conversation*. This one maps the *project*.

## Install

```bash
# Option A — skills CLI (installs into your agent's skills directory)
npx skills add somday-ican/work-trans -g -y

# Option B — clone into your agent's skills directory
git clone https://github.com/somday-ican/work-trans.git ~/.claude/skills/project-handoff

# Option C — single file, zero tooling (any client that reads SKILL.md)
mkdir -p ~/.workbuddy/skills/project-handoff && \
curl -o ~/.workbuddy/skills/project-handoff/SKILL.md \
  https://raw.githubusercontent.com/somday-ican/work-trans/main/SKILL.md
```

> Repo name is `work-trans`; the skill inside is named `project-handoff`. Skill directories use the skill name.

Requires nothing but an agent that can read a Markdown file. Python 3.10+ only for the optional scripts (roadmap).

## Use it

End a session (or when context fills up) and say any of:

> "交接一下" · "handoff" · "传给下一个会话" · "总结工作，让另一个 agent 接手" · "context 快满了，存个交接"

Pass what the next session will focus on as an argument — the document is tailored to it:

> "交接，下一轮跑 R4 实验"

The agent writes the document to `~/.handoff/` (configurable via `HANDOFF_DIR` or a project-level `.handoffrc`) and tells you the absolute path. Point the next agent at that file — on any model, any client, same machine.

## What a handoff document looks like

Abridged from a real session (see [`example/`](example/)):

```markdown
## 2. 文件职责表
| 路径 | 职责 | 现状 | 备注 |
|---|---|---|---|
| gen_stimuli.py | 刺激生成器：出题+调渲染+落 metadata | 修改中 | Step 5 核心 |
| run_batch.py | R1–R9 流水线编排 | 待验证 | R1–R3 已验证，R4–R9 在写 |
| output/协作状态板.md | Kimi 与 deepseek 的主信息通道 | 稳定 | 每次任务后更新 |

## 3.2 未闭环事项
| 事项 | 阻塞源 | 说明 |
|---|---|---|
| REST/REST+ 数据申请 | 等外部回复 | 已邮件问询 UvA 作者，等回信 |
| GPU 2 号卡占用 | 等资源 | 跑批前先 nvidia-smi |

## 7. 下一步唯一具体动作
写完 check_stimuli.py 的 metadata 校验函数并在 ToySample 上跑通，
验收标准：python check_stimuli.py --sample toysample 退出码 0。
```

## How it differs

|  | Built-in `/compact` | Existing handoff skills | **project-handoff** |
|---|---|---|---|
| Trigger | Automatic, lossy | Manual | Manual (交接时机只能由人决定) |
| Output | Hidden compression | Chat summary | **Project map**: manifest + progress + direction + division of labor |
| Audience | Same agent, degraded | Same client family | **Any agent, any model, same machine** |
| Storage | Client internals | Client namespace (`.claude/`) | **Host-neutral** (`~/.handoff/`, configurable) |
| Duplication | — | Sometimes copies artifacts | References by path; never duplicates |
| Open loops | Lost | Rarely captured | **Every item tagged with its blocker** |

## Design decisions (the interesting parts)

- **Store at `~/.handoff/`, not the workspace.** Handoff docs are transient state; they shouldn't pollute a repo's git history. Resolution order: `HANDOFF_DIR` env → project `.handoffrc` → existing known layouts → default.
- **Reference, never duplicate.** Copying a spec into the handoff creates two sources of truth; when one updates, the other silently rots. The handoff is an *index plus delta*.
- **`disable-model-invocation: true`.** Handoff is lossy compression. The model must never decide on its own when it's acceptable to lose information.
- **Fixed section order, no deletions.** The receiving agent navigates by structure; "无" is always better than a missing section.
- **Fail loud on project root.** If no project marker is found, ask — never silently write into the cwd. Silent fallback is how handoff files end up everywhere.
- **Manual trigger, but argument-aware.** "下一轮跑 R4 实验" reshapes the document: environment, repro steps, and known pitfalls move to the front.

## Roadmap

- [ ] `validate_handoff.py` — required sections, no `[TODO]` leftovers, secret-pattern scan, quality score
- [ ] `list_handoffs.py` / `check_staleness.py` — find and freshness-check handoffs for a project
- [ ] Handoff chains — explicit `--continues-from` linking
- [ ] `--brief` mode (goal + state + next action only)

## 中文说明

一个 agent skill：会话结束时，把当前工作压缩成一份**结构化交接文档**，让另一个 agent 或模型（不限客户端：Claude Code、Codex、Kimi、DeepSeek、Cursor 等；唯一要求是同机）零追问接手长期项目。

文档不是总结，而是项目地图，固定九节：项目一页纸、**文件职责表**、进度（已完成可验证 / 未闭环且标注阻塞源）、决策与被否决方案、未来方向、分工、**下一步唯一具体动作**、给接手方的规则、建议先读清单。

默认存放 `~/.handoff/`，可用环境变量 `HANDOFF_DIR` 或项目级 `.handoffrc` 配置；项目文件一律绝对路径引用，不复制、不脱敏遗漏（密钥类脱敏并注明去向）。

触发方式：对话里说"交接 / handoff / 传给下一个会话/模型 / 总结工作"，并可带参数指定下一轮焦点，如"交接，下一轮跑 R4 实验"。

## License

MIT
