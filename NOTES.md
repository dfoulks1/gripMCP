# gripMCP

An MCP-gated approval mechanism for AI agents.

**Purpose:** gate changes before a commit or other big event, so nothing slips
past my attention. Review and approve without losing momentum. Approving a
change is attesting that I understand it and it does what it claims.

## Problem

AI is faster than I can read, and much faster than I can comprehend. Changes
slip past unseen. On 2026-10-27 an agent configured GitHub Actions in my knowledge base
and I didn't notice. Actions cost money. That can't happen again.

## What it does

The change pauses, not the agent.

1. **Classify** every change (file edits and actions: push, mail, API calls)
   against configurable rules: *allow* (log it, keep going), *queue* (hold for
   review), or *block*.
2. **Log** one line per change: what, where, and which rule caught it. A day
   should read in a minute.
3. **Queue** gated changes as proposals. The agent keeps working on anything
   that doesn't depend on them.
4. **Cherry-pick** (the idea, borrowed from git, not the command). I review
   at my leisure and approve this but not that, down to the components of a
   single change (this file, not that one). Rejections go back to the agent
   with my one-line reason.

Decisions:

- Default posture: allow and log, queue on rule match. Queueing everything
  would bury me, which is the problem.
- Dependent work: the agent may build on its own pending changes, marked as
  dependent. If I reject the parent, its children fall with it (like stacked
  PRs).
- Review surface: terminal first. A web page later, maybe.

Rule categories to start with: **cost** (CI workflows, cloud resources, paid
APIs), **outward** (push, send, publish), and **irreversible** (delete,
overwrite).

## How it does it

**Enforcement lives outside the model.** An MCP server only offers tools, and
an agent can skip calling them. Built-in file tools never touch MCP. So there
are three layers:

1. **Gate.** The agent edits freely. The gate fires at a checkpoint: an event
   type I configure (commit, push, deploy, or an immediate action like mail,
   an API call, or a delete). The agent's pre-tool hook (Claude Code
   `PreToolUse`, Kiro hooks later) sends the checkpoint and its batch of
   changes to gripMCP, which matches each against the rules and answers
   *allow*, *block*, or *queued as #N*. On a queue answer the hook denies the
   call with that message and the agent moves on. The hooks are agent-specific
   (lock-in), so they stay thin and the logic lives in gripMCP.
2. **Queue.** A queued change is held as a set of components (files,
   commands) that I can approve whole or in part. Only the approved
   components land. Not line by line.
3. **MCP tools.** The agent calls `status` to learn outcomes and reasons. I
   call `pending`, `approve`, and `reject` from the terminal or in-session.

**Rules** live in one YAML file. `on` names the checkpoint event:

```yaml
- on: commit
  match: { path: ".github/workflows/**" }
  action: queue
  tag: cost
- on: push
  match: { branch: "main" }
  action: queue
  tag: outward
- on: mail
  action: queue
  tag: outward
- on: commit
  match: { lang: python, delete: class }
  action: queue
  tag: irreversible
```

**Language aware where possible.** Rules can match code structure, not just
paths: deleting a class definition in a Python file queues for review. Where
a language isn't understood, matching falls back to the file.

**Log** is append-only JSONL, rendered as
`14:02 queue #12 cost .github/workflows/ci.yml  adds CI on every push`.

**Speed.** The allow path must cost close to nothing: rules load once and
match locally, with no network or model calls. Only queued changes wait, and
they wait on me.

## Attestation

An attestation says: *this change does what I say it does.* I can't say that
about a change I don't understand. So understanding is the job, and review
has to make it cheap.

Each queued change arrives as a **claim plus evidence**:

- **Claim:** one plain line from the agent saying what the change does
  ("adds a CI workflow that runs on every push").
- **Evidence:** the diff or command, plus the rule tag that queued it (cost,
  outward, irreversible).

Approving means I read the evidence and the claim holds. If it doesn't, I
reject it with a reason. A claim that is vague, or that hides a side effect
(like a workflow that bills minutes), is grounds for rejection on its own.

Each decision records:

- **Claim:** the line I attested to.
- **What:** a hash of the exact components I approved. If the content
  changes after approval, the hash stops matching and the approval doesn't
  carry over.
- **Who and when:** the approver and a timestamp.
- **How:** the model that produced it and the rule that routed it.
- **Reviewed or not:** *queue → approved* means a human read it. *allow →
  logged* means a rule passed it.

When approved components land, gripMCP writes trailers from the record:
`Co-Authored-By:` for the model, and `Reviewed-by:` only on changes a human
actually approved. That supports the disclosure the ASF Generative Tooling
policy asks for.

## Later

- Use cases, so we know the design doesn't slow things down.
- Signing approvals (GPG/SSH commits or sigstore).
- Kiro hooks and other agents.
- How queued components are stored (staging branch, patches, or both).
- Trailer for auto-allowed changes.
- Partial approval must be cognizant of a change's components: which ones
  depend on each other, so approving one without its dependency is caught,
  and which dependent changes survive.

---

*AI-generated (Claude Opus 5.5, Claude Code) from a design conversation with
Drew Foulks. Reviewed by Drew Foulks.*
