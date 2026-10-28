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
- Review surface: right here, in the session where the work happens. No
  separate tool to open.
- Rules are trust, written down. *Allow* means I trust that kind of change;
  *queue* means not yet. The log shows whether the trust held.

Rule categories to start with: **cost** (CI workflows, cloud resources, paid
APIs), **outward** (push, send, publish), and **irreversible** (delete,
overwrite).

## How it does it

**Enforcement lives outside the model.** An MCP server only offers tools, and
an agent can skip calling them. Built-in file tools never touch MCP. So there
are three layers:

1. **Gate.** The agent edits freely. The gate is the **verbs**: the few
   actions where work lands (commit, push, deploy, mail, an API call, a
   delete). gripMCP owns them. The agent calls gripMCP's verb, gripMCP
   matches the batch against the rules and answers *allow*, *block*, or
   *queued as #N*, and the agent moves on. gripMCP holds the credentials
   (push token, mail login), so the agent can't land work without it.
   Backstops: agent permissions deny the raw paths (`git push`), and git
   hooks (`pre-commit`, `pre-push`) run the same rules for any agent.
2. **Queue.** A queued change is held as a set of components (files,
   commands) that I can approve whole or in part. Only the approved
   components land. Not line by line.
3. **MCP tools.** The agent calls `status` to learn outcomes and reasons. I
   call `pending`, `approve`, and `reject` here in the session.

**Hold and freeze.** I can interrupt at any moment and make the work wait on
gripMCP:

- **Hold:** the agent keeps working, but every verb queues, whatever the
  rules say.
- **Freeze:** the agent itself waits. A thin agent hook (Claude Code
  `PreToolUse`) denies every tool call, edits included, with "held by
  gripMCP," until I release it. The work stays in place; nothing is lost.
  The hook is agent-specific (lock-in), so it stays thin.

Freeze is one word.

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
paths: deleting a class definition in a Python file queues for review.
gripMCP detects the language from the file itself (extension, shebang,
content), so `lang` is an optional filter, not a requirement. When a
definition is deleted or changed, gripMCP also finds the other files that
import or use it and shows them as part of the evidence, so I see what breaks.
Where a language isn't detected or understood, matching falls back to the
file.

**Extensible by plugin.** The core stays small: rules, queue, log. Two kinds
of plugin hang off it:

- **Verbs:** each gated action (commit, push, mail, ...) is a plugin. A new
  verb needs no core change.
- **Analyzers:** what understands a change. Each language is a plugin
  (detection, structural diff, cross-file impact).

Rules name plugins (`on: push`, `lang: python`), and plugins can extend the
rule language itself: a plugin might add a `never` action, or a new matcher.
A rule that uses a keyword no loaded plugin defines fails closed (queue, and
say why), never open. A plugin runs only when a rule asks for it, so the allow
path stays cheap.

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
- Rule promotion: gripMCP suggests loosening a rule from its record ("20
  approvals, 0 rejections, allow?"). Only I decide.
- Partial approval must be cognizant of a change's components: which ones
  depend on each other, so approving one without its dependency is caught,
  and which dependent changes survive.

---

*AI-generated (Claude Opus 5.5, Claude Code) from a design conversation with
Drew Foulks. Reviewed by Drew Foulks.*
