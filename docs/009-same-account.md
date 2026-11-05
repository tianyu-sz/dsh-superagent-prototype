# 009 Same-account operation

This fork can run with **one GitHub account shared by the person and the agent**. The upstream setup (SETUP.md) recommends a dedicated agent account ([006 Deployment](006-deployment.md)); this document describes the single-account alternative.

## The problem

The GitHub Source drops every message whose author equals the agent CLI's login, so with one account a person's comments never become Triggers. With a dedicated account this is correct: it keeps the agent's own replies from starting new work. With one account it removes the person's requests too. No Policy, admission table or accept mode can fix that, because the messages are dropped before they enter the domain.

## Settings: Sources → GitHub

| Field | Meaning |
| --- | --- |
| 人类与智能体共用同一账号 / Human and agent share one account (`sharedAccount`) | Off by default: upstream behavior, every message of the agent account is dropped. On: unsigned comments from that account are ingested as human Triggers. |
| 智能体评论签名 / Agent comment signature (`agentSignature`) | A standalone line appended to every reply the host posts, default `by 大肥鱼`. Customizable; one line, 1–100 characters. |

## How the code tells a human from the agent

Deterministic, not a model judgment:

1. `sendGithub` appends `\n\n<signature>\n\n<!-- dsh-superagent:<message id> -->` to every outbound comment, including review-thread replies.
2. `isAgentAuthored(body, signature)` in `src/sources/github-trigger.ts` returns true when the body contains the hidden marker or has the signature as a standalone line.
3. `receiveComment` in `src/sources/github.ts` runs for every comment the Source reads (repository-wide streams and notification/subscribed-thread reads):
   - In shared mode, a signed comment is skipped **before** `ctx.receive`, from any author. It cannot start, resume or fan out to a Ticket, so the agent cannot trigger itself.
   - An unsigned comment from the agent account is ingested with `human: true`; the gate then judges it as the person's ([002 Authorization](002-authorization.md)).
   - With `sharedAccount` off, all comments from the agent account are dropped, as upstream does.

An unsigned comment written by a person quoting an agent reply is conservatively treated as the agent's. Keep the signature short and distinctive.

## Triggering work

- **Explicit**: the person writes `@<their own login> <request>` in a new comment. `mentions()` matches the agent login, which is the same account, so the gate admits an explicit request that works in every accept mode ([002 Authorization](002-authorization.md)).
- **Inferred**: an unsigned comment such as "可以做" needs the receive-all comments switch, an inferred admission cell, `auto` mode and a matching rule ([002 Authorization](002-authorization.md), [005 Panel](005-panel.md)).

New Issues and PRs opened by the shared account are still not Triggers: only comments are. This keeps the agent's own PRs out of the flow.

## Worker comments

The host signs every reply it sends. A worker that posts a comment directly with `gh` (for example a review-thread reply, as the GitHub skill describes) must append `$DSH_SUPERAGENT_GITHUB_SIGNATURE` as a standalone final line; the execution environment exposes it. Comments posted outside the reply channel without the signature are treated as human messages in shared mode.

## Limits

- All agent output (comments, PRs, commits) appears under the person's account; audit cannot distinguish them by author.
- Changing the signature does not re-sign comments already posted. Old unsigned comments from before the switch are read as human messages once the Source polls them.
- A human comment containing the signature line, or quoting a signed reply, is ignored. Choose a signature that people do not write by accident.
- Upgrades from upstream overwrite local edits; keep these changes in the fork ([008 Cookbook](008-cookbook.md) explains the Source layout).
