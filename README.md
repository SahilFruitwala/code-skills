# code-skills

A small set of agent skills for developing and reviewing code.

| Skill | What it does |
| --- | --- |
| [`auto-develop`](skills/auto-develop/SKILL.md) | Takes a ticket or description from clarification through implementation, evidence, and review. |
| [`code-reviewer`](skills/code-reviewer/SKILL.md) | Context-aware review of local changes, branches, or PRs, with a second-model revalidation pass. |
| [`council`](skills/council/SKILL.md) | Asks three models from different providers the same question and reconciles their answers. |

## Inspiration

Inspired by [t3dotgg/tstack](https://github.com/t3dotgg/tstack).

## Install

```sh
# all skills
npx skills add SahilFruitwala/code-skills

# a single skill
npx skills add SahilFruitwala/code-skills --skill code-reviewer
```
