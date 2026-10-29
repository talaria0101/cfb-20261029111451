---
title: "Your agent needs a computer, not a container"
published: "2026-08-03T13:15:24.000Z"
description: "Agents need more than just a container to scale. We're introducing @cloudflare/computer, an agent runtime that dynamically orchestrates between fast, efficient isolates and full Linux containers to give every agent a computer of its own."
words: 1755
upstream_status: 200
content_kind: page
extraction: article
source: extractor
title_source: meta
final_url: "https://blog.cloudflare.com/cloudflare-computer/"
fetched_at: "2026-10-29T11:41:31.579Z"
---
August 3, 2026

# Your agent needs a computer, not a container — introducing @cloudflare/computer

[Matt Carey](https://blog.cloudflare.com/author/matt-carey/) and [Aron Carroll](https://blog.cloudflare.com/author/aron-carroll/)

7 minute read

This post is also available in [Deutsch](https://blog.cloudflare.com/de-de/cloudflare-computer/), [Español](https://blog.cloudflare.com/es-es/cloudflare-computer/), [Español (Latinoamérica)](https://blog.cloudflare.com/es-la/cloudflare-computer/), [Français](https://blog.cloudflare.com/fr-fr/cloudflare-computer/), [Italiano](https://blog.cloudflare.com/it-it/cloudflare-computer/), [日本語](https://blog.cloudflare.com/ja-jp/cloudflare-computer/), [한국어](https://blog.cloudflare.com/ko-kr/cloudflare-computer/), [繁體中文](https://blog.cloudflare.com/zh-tw/cloudflare-computer/), and [简体中文](https://blog.cloudflare.com/zh-cn/cloudflare-computer/).

The most capable agents have something simple in common: they are given their own computer to work with.

Coding agents work this way. You give them a filesystem, a shell, tools, packages, and the ability to run code. They inspect the environment, make changes, test their work, and keep going. The computer gives the model a familiar way to act on the world. At Cloudflare, we’re working hard to provide the right primitives on which to build the most capable agents.

**Today we’re introducing an early preview of [@cloudflare/computer](https://github.com/cloudflare/workspace).** The @cloudflare/computer package provides an agent runtime where the details and mechanics of what code runs in an isolate, a container sandbox, or a web browser are handled by the platform. Each agent gets a computer, the runtime optimizes for efficiency, and scalability.

We believe that in order to meet the growing demand for compute required by agentic systems we need to look to solutions beyond traditional containerization.

## Changing how agents are built

We’ve seen a subtle evolution of this story over the past six months. At the start of the year, spinning up a container and running an agent inside of it was the norm. In recent months, we’ve seen a rapid move for agent harnesses to provide sandboxed code execution via tools. This separates the hands (the sandbox where work is done) from the brain (the agent loop).

BLOG-3464_image6.png

No matter where the harness runs, giving every agent a container presents a challenge — across all the clouds, all the hyperscalers, there’s nowhere near enough compute in the world for every company to give each of their users’ agents their own containerized compute environment. This will not scale to hundreds of millions, then billions, of concurrent agents. This is why there is desperate, panicked industry demand for CPU compute, not just GPU compute.

We’ve been working on this problem for a long time at Cloudflare, creating a more efficient compute primitive: isolates. We made that out-of-consensus bet almost 10 years ago when we [introduced Cloudflare Workers](https://blog.cloudflare.com/introducing-cloudflare-workers/). We made it again when we [introduced Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/) almost six years ago. We made this bet because isolates are infinitely horizontally scalable. They spin up and tear down incredibly quickly. They can [hibernate](https://developers.cloudflare.com/durable-objects/examples/websocket-hibernation-server/) when the agent is idle, [store the agent’s own state](https://blog.cloudflare.com/sqlite-in-durable-objects/), and even [spin up their own isolates](https://blog.cloudflare.com/dynamic-workers/) to run untrusted code. Isolates are the best way to scale horizontally, and horizontal scale is what agents demand.

BLOG-3464_image3.png

Last year, we [gave isolates the ability to spin up their own container sandboxes](https://blog.cloudflare.com/containers-are-available-in-public-beta-for-simple-global-and-programmable/). From day one, Cloudflare’s architecture has been designed to run the agent harness in the isolate (in a Durable Object) and call an attached container on-demand as a tool. This allows you to utilize heavier compute primitives only when required, optimizing performance and cost. Durable Objects scale infinitely horizontally, and the attached container lets it scale vertically to perform any task. This is how we build agents ourselves, and we’re seeing customers build incredible things this way too.

BLOG-3464_image8.png

But when we look at this need to have multiple underlying compute primitives to build agents (isolates and containers) and the need for our customers and developers to combine them themselves in userspace, we think we can do better. We think that we can provide a simpler abstraction.

That’s why we’re starting this experiment by shipping @cloudflare/computer as an open-source library, to learn with our customers who are pushing the bounds of running agents at scale.

## A shared filesystem across isolates and containers

The @cloudflare/computer package starts with a simple premise: what if we give an agent a primed filesystem, declaratively defined, containing everything required for the task at hand and a selection of execution environments to operate on those files, each with their own pros and cons regarding speed, capability and cost?

It turns out that agents today are surprisingly capable of selecting the right environment for the task at hand. A job that only needs to manipulate files, process data, or manage a git repository can run inside an isolate. A command that needs Linux, `npm`, or a native binary can run inside a container. Both work against the same files that are kept in sync with the source filesystem.

BLOG-3464_image2.png

The @cloudflare/computer package provides a durable filesystem that you can use with git repositories, storage buckets or any files you choose. It provides tools that let you read, write and edit files using [Code Mode](https://blog.cloudflare.com/code-mode/) or bash commands. All operations are gated, audited and observed, giving you fine-grained control over changes the agent is allowed to perform as well as a clear paper trail showing what the agent did.

## How you use it

An instance of a @cloudflare/computer workspace can be instantiated on any Durable Object to provide a virtual filesystem and execution runtime.

It is installed via npm:

```
npm install @cloudflare/computer
```

The primary use case is to provide that filesystem and tooling to an agent. For example, here’s how to instantiate the workspace on an agent powered by @cloudflare/think intended to triage bug reports.

```
import { Think } from "@cloudflare/think";
import { Workspace, type DurableObjectStorageLike } from "@cloudflare/computer";
import { createWorkersAI } from "workers-ai-provider";

export class Agent extends Think {
  override workspaceBash = false;

  override workspace = new Workspace({
    storage: this.ctx.storage,
    useThink: true, // soon will not be needed
  });

  override getModel() {
    return createWorkersAI({ binding: this.env.AI })("@cf/zai-org/glm-5.2");
  }

  override getSystemPrompt() {
    return `
You are a bug triage agent.

Use the project in /workspace/repo to reproduce the bug, inspect the
code, make a focused fix when it is safe, and run verification. In your
final answer, include what you changed, which commands you ran, and
whether verification passed.`;
  }
}
```

Several execution backends are provided as part of the @cloudflare/computer package, or you can write your own. Here we wire up a Cloudflare Container.

```
import { Think } from "@cloudflare/think";
import { Workspace, WorkspaceProxy } from "@cloudflare/computer";
import {
  CloudflareContainerBackend,
  withWorkspaceContainer,
} from "@cloudflare/computer/backends/container";

export { WorkspaceProxy };

export class Agent extends withWorkspaceContainer(Think) {
  override workspaceBash = false;

  override workspace = new Workspace({
    storage: this.ctx.storage,
    useThink: true, // soon will not be needed
    backends: [
      new CloudflareContainerBackend({
        container: () => this,
        workspace: {
          binding: "Agent",
          id: this.ctx.id.toString(),
        },
      }),
    ],
  });

  /* Example code truncated for readability... */
}
```

Expose the file, git, and shell tools alongside product specific tools to reply to reported issues.

```
import { createAITools } from "@cloudflare/computer/tools";
import type { ToolSet } from "ai";
import { replyToIssue } from "./tools/github";

export class Agent extends withWorkspaceContainer(Think) {
  override workspaceBash = false;

  /* Example code truncated for readability... */

  override getTools(): ToolSet {
    return {
      ...createAITools({
        workspace: this.workspace,
        shell: {
          defaultBackend: "container",
          backends: {
            container: {
              description:
                "Cloudflare Container with a full Linux userland: " +
                "npm, node, package managers, test runners, and real " +
                "binaries on $PATH. Use it when a task needs more than " +
                "file manipulation.",
            },
          },
        },
      }),
      replyToIssue,
    };
  }
}
```

The model can use tools during the agent loop, but you can also use the workspace API directly, for example, to prepare the environment before prompting the agent.

```
export class Agent extends withWorkspaceContainer(Think) {
  override workspaceBash = false;

  /* Example code truncated for readability... */

  async startTriage(report: { title: string; body: string; repoUrl: string }) {
    await this.workspace.fs.mkdir("/workspace", { recursive: true });
    await this.workspace.fs.writeFile(
      "/workspace/BUG_REPORT.md",
      `# ${report.title}\n\n${report.body}\n`,
    );

    await this.workspace.git.clone({
      url: report.repoUrl,
      dir: "/workspace/repo",
    });

    return this.submitMessages([
      {
        id: crypto.randomUUID(),
        role: "user",
        parts: [
          {
            type: "text",
            text: [
              `Triage this bug: ${report.title}`,
              "The bug report is in /workspace/BUG_REPORT.md.",
              "The repository is checked out at /workspace/repo.",
            ].join("\n"),
          },
        ],
      },
    ]);
  }
}
```

Check out the [workspace repository](https://github.com/cloudflare/computer) for more examples of how to use the different backends and tools including a [step-by-step tutorial](https://github.com/cloudflare/computer/tree/main/examples/tutorial)walking through building an agent from scratch.

## How it works

The central piece of @cloudflare/computer is the workspace. A virtual filesystem backed by SQLite that can be populated from various sources including cloud storage and source control.

BLOG-3464_image7.png

The workspace supports optional execution runtimes that allow code to be run against the file system. All runtimes support the same interface `exec(string, options)` and currently two are provided out of the box (but you can write your own):

- An isolate-based runtime environment that uses [just-bash](https://justbash.dev/) to translate shell code into JavaScript runs in a [dynamic worker](https://developers.cloudflare.com/dynamic-workers/). Here, the filesystem is available directly via worker bindings.
- A container runtime that uses [Cloudflare Containers](https://developers.cloudflare.com/containers/) to provide a full Linux environment. Here, the filesystem is provided via a Filesystem in Userspace (FUSE) mount, which ensures files are available to the container and changes are synced back.

The `Workspace` class provides an API interface for manipulating the filesystem directly as well as a `node:fs` compatible wrapper so that it can be used easily with third-party JavaScript libraries.

image4.png

For use with agents, we provide an AI SDK compatible toolkit that provides the most common tools: read, write, edit, ls and exec. The exec tool is a little special as it works across the runtimes taking a `backend` argument. The tool description guides the agent into choosing the correct runtime for the task at hand: either a fast, cheap worker backend or the fully featured container. In our testing, the frontier models are very good at making the correct decision and falling back to using containers only when needed.

## What’s next

Here at Cloudflare we’re already seeing agents exclusively using isolates to build, test, and deploy JavaScript applications with modern tooling, generate tailored documentation for each of our customers, and use web browsers to perform complex tasks.

Our goal with @cloudflare/computer is to provide an agent with a runtime where a container is required for less than 10% of its work, and coding tasks, audio/video manipulation, and document creation can all be handled by isolates.

Try out the [early preview today](https://github.com/cloudflare/computer) - we can’t wait to hear your thoughts.

On this page

Discuss Online

## Related tags

[Agents](https://blog.cloudflare.com/tag/agents/)[Agents Week](https://blog.cloudflare.com/tag/agents-week/)[AI](https://blog.cloudflare.com/tag/ai/)[Cloudflare Workers](https://blog.cloudflare.com/tag/workers/)[Containers](https://blog.cloudflare.com/tag/containers/)

Follow on Social Media

- CloudflareCloudflare
- Matt Carey[Matt Carey](https://blog.cloudflare.com/author/matt-carey/)
- Aron Carroll[Aron Carroll](https://blog.cloudflare.com/author/aron-carroll/)

## Subscribe to receive notifications of new posts

Email address

We’ll never share your email address.
