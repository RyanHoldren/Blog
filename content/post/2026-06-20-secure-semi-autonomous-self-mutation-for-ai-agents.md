+++
date = '2026-06-20T8:00:00-07:00'
title = 'Secure, Semi-autonomous Self-mutation for AI agents'
+++

Giving an AI agent real capability and maintaining real security feel like opposing goals. The more the AI agent can do, the more damage it can do if something goes wrong. Nowhere is that tension sharper than when the agent's capability is the ability to change _itself_.

I run a personal AI agent that holds sensitive personal data (e.g. health, financial, etc.) and gives me insights using it. It has two capabilities that stand out as both the most useful and the most dangerous: it can **write and deploy its own tools**, and it can **propose changes to its own infrastructure**. Either of these would qualify the agent as self-mutating; it can grow new abilities and reshape the very system it runs on, with little or no involvement from me at all.

This post is about how I architected each of these capabilities to make them nonetheless safe.

## The containment context

The standard framing for safety with respect to AI agents is Simon Willison's ["lethal trifecta"](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/): a safe AI agent can have any two of the following, but not all three:

1) Access to private data
2) Exposure to untrusted content
3) The ability to communicate externally

The standard advice is to pick _which_ leg of the trifecta you are willing to sacrifice. The primary purpose of my AI agent is to gather and process private data, so that leg is fully intact. Also, because that data comes from third parties, I do not _fully_ trust it. By process of elimination, I ought to prevent my agent from communicating externally (i.e. with anyone but me).

To prevent external communication, the AI agent runs on a dedicated virtual machine, firewalled so that it can only communicate with a short allowlist of third parties (i.e. the APIs that I am using to gather data). This is enforced by transparently routing all its traffic to a separate virtual machine, the "firewall". Crucially, the firewall discards the original destination IP and uses the `ClientHello`'s SNI purely as a lookup key: if the requested hostname is on the allowlist, the firewall resolves it and connects there itself; otherwise it drops the connection. Because the client doesn't actually choose the destination, the usual SNI-filter bypasses don't apply. For example, the AI agent cannot reach an unlisted host by connecting to an allowlisted IP and lying about the hostname, and an encrypted `ClientHello` simply fails closed, since with no readable SNI there is no allowlist key to match.

There is also a DNS firewall at the resolver level that returns `NXDOMAIN` for any query not on the allowlist, so that the DNS protocol itself cannot be used as an exfiltration channel. Note that none of these controls can be modified from inside the agent; they live outside its virtual machine.

A careful reader will object that this _narrows_ external communication rather than eliminating it, and that's fair. Most of the allowlisted APIs will accept outbound data in a query string or request body, so a determined agent could in principle dribble information out through a service it is allowed to reach. That residual is real, which is why I don't lean on the egress allowlist alone; I also shrink the trifecta's _second_ leg. The agent ingests data from only a small set of structured APIs (no open web, no email, no arbitrary documents), and that data is almost entirely numeric. The surface for an injected instruction to reach the model at all is therefore tiny, and the surface for one to smuggle data back out is even smaller.

## Threat model

There are two threats, and it helps to keep them distinct.

### The malicious agent

My AI agent may become malicious and try to cause harm. These LLMs are inherently a little unpredictable; no one can guarantee that this can't just happen _spontaneously_. If this were to happen, the harm it might seek falls into a few categories:

**Exfiltration.** My AI agent has access to a lot of sensitive information (e.g. API credentials, private data, etc.). A malicious AI agent might try to leak that information to the outside world.

**Destruction.** A malicious AI agent might try to destroy or corrupt its data (e.g. by deleting its database or filling it with nonsense) or break its infrastructure (e.g. by killing processes or deleting its code repository).

**Financial damage.** Running LLM inference, spinning up cloud infrastructure, and making API calls all cost money. A malicious AI agent may deliberately try to run up a bill by abusing these expensive resources. This is sometimes called a "Denial of Wallet" attack.

**Tricking me.** It almost goes without saying, but my AI agent can talk to me, so it _could_ try to social-engineer me into doing something against my best interest.

### The malicious actor

A malicious actor might also try to achieve any of these damage goals _directly_, without involving the AI agent at all, using a conventional vulnerability in the infrastructure. This is just classic IT security.

Alternatively, a malicious actor might try to poison my AI agent through prompt injection. This is more likely than, but not dissimilar in outcome from, the AI agent _spontaneously_ becoming malicious.

## Why command allowlisting is security theatre

The obvious way to let an agent act autonomously but safely is a command allowlist. Most AI agent harnesses (OpenClaw, Codex, Claude Code, etc.) have one: it restricts which shell commands the agent can execute without asking. If the agent wants to run `curl` and `curl` isn't on the list, the human gets prompted for approval.

This _feels_ like security; it ought to prevent exfiltration, right? It's certainly _super_ annoying, which probably contributes to it _feeling_ like security. If you have ever given an agent a prompt and gone for lunch expecting to review the finished product when you get back, only to discover an hour later that it stopped _almost immediately_ to ask permission for something trivial, you know exactly what I'm talking about.

Here is the problem: **the moment your agent can write code and execute that code, a command allowlist is security theatre**; it does not meaningfully restrain the agent's capabilities. You cannot enumerate your way to a secure allowlist, because the dangerous thing was never the command: it's the process and its credentials. An agent blocked from running `curl` can write and run a five-line program that opens a socket. An agent blocked from `aws s3 cp` can call the SDK directly or even write its own client for the REST API.

This is the heart of the self-mutation problem. Once you accept that you _want_ the agent to author and run its own code, you have to throw out command-level permissions entirely and contain the agent via its process, its network, and its credentials.

## Writing and deploying its own tools

### The goal

I want my AI agent to **autonomously** iterate on its own tooling. This is a lesson I've learned through experimentation: it is often _much cheaper_ and _much more accurate_ to have an agent write itself a tool to do something than to have the agent do it directly, token by token. But I am a busy guy; I don't have time to sit around approving pull requests from my own agent. It needs to build, debug, improve, and maintain these tools on its own.

### The risk

This is precisely the capability that makes command allowlisting pointless (see above). An agent that writes and runs code can do anything its process can do. So the risk isn't any particular command; it's that a self-written tool could open a new exfiltration path, abuse the agent's credentials, or pull in a malicious dependency as a supply-chain attack.

### The architecture

The only way to securely allow an AI agent to autonomously write and execute its own tools is to lock down its process and let the tool inherit that containment. A tool the agent writes runs as the same Unix user, with the same credentials, behind the same network egress filter as the agent itself. It can do nothing the agent couldn't already do. It's worth separating two mechanisms here: _runtime containment_ bounds what a tool can **do** once it runs—the process, network, and credential limits it inherits—while _build-time vetting_ (described below) bounds what it can **pull in**. Neither asks me to read the tool line by line; together they are what make human review of every tool unnecessary.

Here is the workflow I have arrived at:

1. The AI agent writes some code in Rust.
2. The AI agent calls the `tools.deploy` tool.
3. The `tools.deploy` tool starts an ephemeral HTTP server.
4. The `tools.deploy` tool commits the new source code to a Git repository, appending a URL to the ephemeral server in the commit message.
5. The `tools.deploy` tool pushes the commit.
6. The `tools.deploy` tool then blocks, waiting for a webhook notification to that server.
7. Meanwhile, the CI/CD pipeline sees the new Git commit and checks whether any of the dependencies (a.k.a. "crates" in Rust) have changed:
    - If none have changed, it reuses the existing pre-compiled dependency image and skips ahead to compilation (step 8).
    - If some have changed, it verifies them with `cargo vet` and `cargo deny`. An unvetted crate aborts the build, and the failure is reported back to the agent (see step 10) so it can drop the dependency or ask me to allowlist the publisher. Once every dependency is vetted, the pipeline produces a new Docker image containing the pre-compiled dependencies. This is the build-time vetting from earlier: it bounds what the agent can pull in, stopping a malicious dependency from becoming a supply-chain attack.
8. The CI/CD pipeline pulls the dependency image, overlays the new application code, and compiles it. This step has no internet access, so even if a `build.rs` tried to phone home during compilation, it would fail.
9. The CI/CD pipeline copies the resulting binary directly into the AI agent's virtual machine.
10. The CI/CD pipeline sends an HTTP POST containing the outcome of the build to the AI agent's ephemeral HTTP server.
11. Upon receiving the webhook, the `tools.deploy` tool finishes and reports success or failure to the AI agent.

The new binary appears, ready to run, in the AI agent's `$PATH` immediately; no restart required.

### Benefits

This architecture has a number of benefits:

1) Compiling Rust is expensive. By delegating compilation to the CI/CD pipeline, the virtual machine for my AI agent can be **dramatically** smaller, avoiding idle capacity.
2) This works surprisingly well because AI is very good at writing Rust. From a frontier model, you will almost never see a compiler error. Even cheap, open-source models are good enough to get a small tool to compile after a couple iterations.
3) Moreover, the agent needs very little context on _how_ the pipeline actually works, since it is all encapsulated by the `tools.deploy` tool.
4) Finally, because the `tools.deploy` tool blocks, it takes only a single LLM inference call to build a new binary. If an AI agent tried to do this directly, it would take _at least_ four (i.e. `git add`, `git commit`, `git push`, and some sort of Bash loop to watch the CI/CD pipeline) and usually a lot more than that on a cheap, open-source model.

### Known issues

If this sounds a little too good to be true, it is. I have run into a number of issues with it:

1. There are a number of third parties that make public their human reviews of Rust crates so that you can use them with `cargo vet` (Google, Mozilla, etc.). Unfortunately, even in aggregate, the breadth is narrow: none of the AWS SDK crates, for example, are vetted. I have had to explicitly allowlist a bunch of publishers. Note that the allowlist cannot be changed by my AI agent without my approval, so this adds friction.
2. Even though the `tools.deploy` tool is always in context, my AI agent will frequently try to run `cargo` or `rustc` directly. The first few times this happened, it would go off the rails: first trying desperately to find them (which failed because they were never installed), then trying to install them (which failed because of the firewall), then just flailing around helplessly. In the end, I added some guardrails by creating _fake_ `cargo` and `rustc` binaries that return an error and tell the agent that "Local compilation is not available; use the `tools.deploy` tool instead."

## Changing its own infrastructure

### The goal

To allow my AI agent to **participate in its own infrastructure evolution** (e.g. by proposing new capabilities, upgrading its vendors, fixing issues, etc) without me being the sole author of every change.

### The risk

This is the one place where the "let it inherit the containment" logic breaks down—because the infrastructure _is_ the containment. A malicious infrastructure change could dismantle the agent's own restraints by, for example, opening a new egress path or granting itself broader permissions. This is the most dangerous capability in the system, because a successful infrastructure change can undo every other control. So unlike self-written tools, infrastructure changes _cannot_ be fully autonomous; they require a human in the loop.

### The architecture

The trust boundary is drawn in the platform's access-control layer, not in a system prompt. There are two separate Git repositories with deliberately different access:

| Repo    | Access                                                                                            |
|---------|---------------------------------------------------------------------------------------------------|
| `tools` | Agent _can_ push directly to `main` but not force-push                                            |
| `infra` | Agent _cannot_ push directly to `main`. It can only push to a branch and then open a pull request |

Note that these access rules are not a convention or a system-prompt instruction; they are enforced by the Git host's access-control policy, which forbids the agent from pushing to `main` in the infra repo.

With this limitation in mind, here is how the workflow works:

1. The AI agent makes changes to the infrastructure-as-code and then calls the `infra.deploy` tool.
2. The `infra.deploy` tool starts an ephemeral HTTP server.
3. The `infra.deploy` tool commits the changes, recording the session ID of the current conversation and the server's URL in the commit message.
4. The `infra.deploy` tool pushes the commit to a branch.
5. The `infra.deploy` tool opens a pull request.
6. The `infra.deploy` tool then blocks, waiting for a webhook notification to that server.
7. The pull request triggers an email notification to me with the full diff and buttons to "Approve" or "Reject" the changes. These buttons have HMAC-signed URLs. Note that the HMAC secret lives outside any store the AI agent can read, so forged approval URLs aren't possible.
8. I review the diff and click "Approve".
9. A serverless function merges the pull request into `main`.
10. The CI/CD pipeline picks up the new commit and rebuilds the machine images.
11. The CI/CD pipeline updates the infrastructure-as-code stack, which, if necessary, replaces the virtual machine that is running the AI agent.
12. When the deployment completes, the pipeline tries to `POST` the outcome to the ephemeral HTTP server that the `infra.deploy` tool is blocking on. If that works, the tool returns success and the agent moves on.
13. If that POST fails—it _definitely_ will if the virtual machine running the AI agent was replaced during the rollout, taking the ephemeral server down with it—the pipeline falls back to the agent harness's stable webhook endpoint, which lives at a fixed address that survives reboots and instance replacement. It addresses the message to the session ID recorded in the Git commit, and because the harness persists sessions across reboots, this resurrects that exact conversation with the results of the pipeline, so it can resume where it left off.
14. Alternatively, if the pull request was rejected or the rollout failed, the `infra.deploy` tool receives a `POST` request containing a failure report, and prints it for the AI agent.

This is the "semi-autonomous" in the title. The agent does the work—it reads the infrastructure-as-code, writes the change, opens the pull request, waits for the deploy, and picks up where it left off—but a human is the one irreducible gate, positioned exactly at the one boundary where autonomy would let the agent free itself.

It's worth being honest about what that gate rests on. My threat model includes an agent that tries to social-engineer me, and nothing stops it from pairing a subtly-hostile diff with a convincing justification. So the gate is only as strong as the review behind it. I review with a default-reject posture: if I don't understand both the _purpose_ of a change and its full _implications_, it gets rejected. The burden is on the proposal to be self-evidently safe, not on me to prove that it isn't—a persuasive story wrapped around a diff I don't understand is a rejection, not an approval.

## The design principle

Every control in this system is designed to hold against the same question: _what if the AI agent is running hostile code, right now?_

Not "what if it makes a mistake." Not "what if the LLM hallucinates." But: what if an attacker has complete control of the AI agent's reasoning and execution?

Command-level permission systems don't survive that question—and for a self-mutating agent, they were never going to, because the agent can always write a new command. Process-level controls—network egress, access-control policy, external authentication—do survive it. That premise dictates the whole design. Self-written tools are safe because they inherit a containment the agent cannot loosen. Infrastructure changes are safe because the agent physically cannot push them: the two-repo access split exists not because the agent is _instructed_ not to touch the infra repo, but because it _cannot_, and the only path forward runs through a human holding an HMAC key the agent can't forge.

Containment answers "can the agent get out of the box?" It does not, on its own, answer "how much damage can it do _inside_ the box?"—and the threat model named two such harms: destruction and Denial of Wallet. Those are bounded by separate controls, none of which the agent can reach. The data-bearing resources—e.g. the Git repositories—cannot be deleted by the infra pipeline, and the database is backed up daily to storage that is likewise undeletable, so the durable data survives even a malicious change. Spending is watched by alarms that fire almost immediately on any increase; hard spend caps on both infrastructure and inference bound the worst case; and most of the third-party APIs are free—an agent that hammers them gets rate-limited, not billed.

The result is an agent that can genuinely evolve by writing its own tools, proposing upgrades and reshaping its own infrastructure, all without ever being able to evolve its way out of the box. Capability and containment are both real. The art is making sure they're independent of each other.