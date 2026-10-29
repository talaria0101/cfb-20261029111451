---
description: Cloudflare Containers now start 6x faster, let your agent choose each sandbox&#39;s image and instance type at runtime, and support filesystem snapshots in public beta, all controlled from a Durable Object.
title: Cloudflare Containers, rebuilt to scale agent sandboxes
image: https://blog.cloudflare.com/_emdash/api/media/file/01M3S8R4W0018F4ABEE2X1ZH4G.01M3S8R5VDFQC24Z3Q4E6RBE3E.png
---

[Skip to content](#main-content)

[Cloudflare home](https://blog.cloudflare.com/)

[Cloudflare home](https://blog.cloudflare.com/)

![BLOG-3310 hero image](https://blog.cloudflare.com/_image?href=https%3A%2F%2Fblog.cloudflare.com%2F_emdash%2Fapi%2Fmedia%2Ffile%2F01KW46H9N3ZPBVM99CMTJZRPF3.png&w=1999&h=1125&f=webp&fit=cover&position=center)![]()

Today, we’re making Cloudflare Containers more programmable and optimized for agent workloads. Agents don't deploy sandboxes ahead of time. They create sandboxes on demand, for each task, expect them to be ready immediately, and be able to pause and resume. So we rearchitected Containers to meet these requirements: your code can now choose each sandbox's image and instance type at runtime, Containers start 6x faster, and filesystem snapshots are available in public beta.  
  
To make this possible, we’ve rethought the Containers infrastructure from the bottom up. A new scheduling policy moves control over each sandbox into application code, while a redesigned runtime provides a faster path to a running Container. In ComputeSDK’s independent benchmark, median startup fell from just over four seconds to 648 milliseconds, and, in our own preliminary tests, burst testing successfully created hundreds of thousands of containers in seconds.

All of this builds on what has always set Containers on Cloudflare apart: every Container gets its own Durable Object, a persistent, programmable controller running right next to it that manages its lifecycle, outbound traffic, and more. We are bringing more capabilities directly to the native `ctx.container` API, so the Durable Object can control its Container without a wrapper class in between, and we’re carrying this model into Sandbox SDK 1.0.  
  
As we wrote earlier this year, [<u>your agent needs a computer.</u>](https://blog.cloudflare.com/cloudflare-computer/) These changes make Containers a better complement to Workers, Dynamic Workers, and Durable Objects when agents need a full Linux workspace.

## Rethinking Containers’ runtime for agents

Until now, Cloudflare Containers has been organized around application deployments. You choose an image and compute resources at deploy time, roll that configuration out across the application, and manage it centrally. The application is the unit of configuration and rollout.  
  
An agent workspace is different: it’s created on demand, while the agent is working. The task determines its image, resources, tools, and starting filesystem. It might exist for a few minutes, sleep between requests, or be restored days later. Those decisions need to live with the application code handling the task, and the agent's sandbox needs to start up fast, because every second of startup is time your users spend waiting.

We’ve seen this pattern with Base44 on app-building workspaces and Kilo Code on cloud-agent sessions. It also appears in our integrations with [<u>Cursor Cloud Agents</u>](https://developers.cloudflare.com/changelog/post/2026-09-02-cursor-cloud-agents/), [<u>Devin Outposts</u>](https://developers.cloudflare.com/changelog/post/2026-07-21-devin-outposts/), the [<u>OpenAI Agents API</u>](https://developers.cloudflare.com/changelog/post/2026-09-10-using-openai-agents-api-with-cloudflare-containers/), and [<u>Claude Managed Agents</u>](https://blog.cloudflare.com/claude-managed-agents/).

Each of these workloads needs something different from the workspace. Coding agents need repositories, package managers, compilers, test runners, and development servers. Evals need sandboxes that begin from a known state. Reinforcement learning systems need to create, grade, and reset large numbers of environments. Longer-running tasks need to preserve the files an agent produces, so work can continue later.

These requirements led us to fundamentally rethink how Cloudflare Containers are configured, scheduled, and saved. The result is a new way to provision and schedule Containers: the `durable_object` scheduling policy. It lets your code choose each sandbox’s image and compute resources at runtime, starts Containers more than 6x faster, and supports filesystem snapshots, so workspaces can be saved and restored.

> *“At Base44, we help anyone turn an idea into a working app. Cloudflare Containers gives each app an isolated development environment where our AI can execute commands, install dependencies, and bring changes to life in a live preview.”*
>
> *Dolev Epshtein, Software Engineer, App Infrastructure at Base44*

---

> *“At Kilo Code, every cloud-agent session needs its own workspace and environment, with the right repository, tools, and user configuration. Cloudflare Containers lets us create those isolated environments on demand, so our agents can start running commands quickly and get to work for our customers.”*
>
> *Emilie Schario, Co-founder of Kilo Code and VP Engineering, AI Workspaces at Anaconda*

## Choose the sandbox your agent needs, from code

[<u>From the start</u>](https://blog.cloudflare.com/containers-are-available-in-public-beta-for-simple-global-and-programmable/), every Cloudflare Container instance has been attached to a Durable Object. The Durable Object gives the environment a stable identity and lets application code control when it starts, sleeps, and stops. This model has proven particularly well-suited to agent sandboxes.

Until now, though, the two decisions that matter most for an agent sandbox were locked in at deploy time: which image it runs and how much compute it gets. Each combination of image and instance type was its own Containers application, with its own Durable Object namespace, set up ahead of time with `wrangler deploy`. 

Say one agent needs a small [<u>Node.js</u>](https://nodejs.org/) sandbox and another needs a large Python sandbox for builds. With our previous approach, that required two applications, two namespaces, and routing logic in your Worker to send each task to the right one. Every new environment meant another deployment. 

The new `durable_object` scheduling policy removes that. The image and instance type are now arguments your code passes when the sandbox starts. To opt in, set the scheduling policy and declare the images your Durable Object can choose from:

```
// wrangler.jsonc
{
  "containers": [
    {
      "class_name": "AgentSandbox",
      "scheduling_policy": "durable_object",

      "images": {
        "node": { "dockerfile": "./images/node/Dockerfile" },
        "python": { "dockerfile": "./images/python/Dockerfile" }
      }

    }
  ],
  "durable_objects": {
    "bindings": [{ "name": "SANDBOX", "class_name": "AgentSandbox" }]
  }
}
```

Each image you declare is available as `this.ctx.container.images.<name>` within the Durable Object. When a task arrives, your code picks the image and instance type for that task:

```
import { DurableObject } from "cloudflare:workers";

export class AgentSandbox extends DurableObject {
  async startWorkspace(workspace) {
    if (this.ctx.container.running) {
      return;
    }

    const image =
      workspace.toolchain === "python"
        ? this.ctx.container.images.python	
        : this.ctx.container.images.node;

    const instance =
      workspace.workload === "build"
        ? "standard-2"
        : "standard-1";

    this.ctx.container.start({
      image,
      instance,
      enableInternet: true,
    });
  }
}
```

This code makes two decisions after the task is known. It chooses the toolchain the workspace needs and how much compute to give it. What used to take a separate application and a separate `wrangler deploy` is now an `if` statement. One Durable Object class can start [<u>Node.js</u>](http://node.js) and Python sandboxes of different sizes side by side, and adding a new environment is a code change, not a new deployment. That’s the idea behind this whole update: infrastructure becomes code that runs at request time, right down to the environment itself.

### Rollouts are now just code

Choosing the image at start time also fixes one of the most painful parts of running Containers: rollouts.

Before, updating an image meant updating the whole application. You set grace periods, so running instances could drain, defined percentage splits to move traffic gradually, and called our API to push the new configuration. Throughout that process, the platform decided which instances got replaced and when, whether an agent was in the middle of a task.

With the `durable_object` scheduling policy, there's no rollout configuration at all. A Container can keep running the image it started with until your code stops it. The next time that Durable Object starts a Container, it uses whatever image your code chooses. That means any rollout strategy you want is a few lines of code:

```
const image =
  (await this.ctx.storage.get("pinned-image")) ??
  (isCanary(this.ctx.id)
    ? this.ctx.container.images.nodeV2
    : this.ctx.container.images.node);
```

For example, you can:

- **Canary** a new toolchain on 5% of new sandboxes by hashing the Durable Object ID.
- **Pin** active projects to their current image, so an agent never has its environment swapped out mid-task.
- **Migrate** a workspace at a natural checkpoint, like the next session or after a snapshot.
- **Roll back** by changing which image future starts choose. No config push, no waiting for a drain.

The rollout policy lives in your Durable Object code, next to the rest of your logic, and it can be as simple or as sophisticated as you need.

Each of these improvements comes from **leaning further into** the Durable Object, which already owns the workspace's identity, state, and lifecycle. Letting it choose and control its Container **gives you more control** over every instance and its rollout. It also gives the scheduler a better place to start the Container: wherever the Durable Object is already running. That's what gets the agent to its first command faster.

## Faster first commands

Previously, starting a Container required our global control plane to resolve the application configuration, find capacity, and coordinate placement. That model works well for application-wide fleets, but it put deployment machinery in the path of an agent’s first command.

With the `durable_object` scheduling policy, demand begins at the Durable Object. The Containers infrastructure serving it looks for capacity on the same machine first, then widens the search within the same location if it needs to. It also favors hosts that already have the Container’s image or snapshot in local storage, so the Container can start without downloading it first. 

We also cut work after the Container reaches a host. Instead of booting a new virtual machine from scratch, the new runtime restores a prepared virtual machine that isn’t yet assigned. It reuses networking and filesystem setup, batches repeated operations, and no longer waits on services the first command doesn’t need. 

Together, these changes substantially reduce the time it takes to go from creating a sandbox to running a command in it. On ComputeSDK’s independent [<u>Burst TTI Benchmark</u>](https://www.computesdk.com/benchmarks/sandboxes/burst-tti/cloudflare), which launches 100 sandboxes concurrently and measures time-to-interactive from the client:

| Startup measurement | Previous scheduling path | New scheduling policy | Improvement |
| --- | --- | --- | --- |
| Median | 4.049 seconds | 648 milliseconds | 6.2x faster |
| 95th percentile | 5.839 seconds | 910 milliseconds | 6.4x faster |
| 99th percentile | 6.717 seconds | 1129 milliseconds | 5.9x faster |

The new path also holds up under burst load. In our preliminary burst test, a single account started 100,000 Containers in 5.387 seconds across six locations.

### Start with a prepared system image

As the scheduling path gets faster, preparing the image becomes a larger part of the remaining wait. Before a Container can start, its image has to be on the host and unpacked into a filesystem. When that work happens after the request arrives, the agent is left waiting.

That’s why we are introducing `cloudflare/debian-trixie`: a ready-to-use system image for agents that can configure their environment at runtime, containing Debian Trixie Slim and Node.js 24.20.0 LTS:

```
this.ctx.container.start({
  image: "cloudflare/debian-trixie",
  instance: "standard-2",
  enableInternet: true,
  entrypoint: ["/bin/sleep", "infinity"]
});
```

This means your agent can start a Linux sandbox without first creating a Dockerfile, building an image, or pushing it to Cloudflare. Once the sandbox is running, your agent can use `exec()` to clone a repository, install packages, and configure the environment for its task.

And because Cloudflare controls this image, we can distribute and prepare it across eligible Containers hosts before requests arrive. Startups don’t have to download or unpack the base image while the user waits.

## Save the workspace and return to it later

A fast start still leaves one more wait: setting up the workspace. Cloning a repository, installing dependencies, and configuring a toolchain can take much longer than starting the Container itself. As the agent works, it also produces files you want to keep. Repeating setup on every start wastes time, and losing the agent’s changes makes it difficult to continue a task.  
  
That is why we’re adding native filesystem snapshots to Containers, in public beta. Snapshots let an agent begin a task in a prepared environment, save its workspace when the task pauses, and restore those files when the session resumes.

```
async saveWorkspace() {
  const snapshot = await this.ctx.container.snapshotContainer({
    name: "project-ready",
  });

  await this.ctx.storage.put("workspace-snapshot", snapshot);
}

async restoreWorkspace() {
  const snapshot = await this.ctx.storage.get("workspace-snapshot");

  if (!snapshot) {
    throw new Error("No workspace snapshot found");
  }

  this.ctx.container.start({
    containerSnapshot: snapshot,
    instance: "standard-2",
    enableInternet: true,
  });
}
```

Snapshots enable two useful patterns.

First, one workspace can continue across many sessions. For a coding agent, that might mean saving the workspace when the user finishes working and restoring it when they return the next day. The repository, installed dependencies, build caches, configuration, and edits are available without rebuilding the environment.

Second, a snapshot can provide a shared checkpoint for many sandboxes. Since snapshots are immutable and reusable, multiple Containers can start independently of the same prepared environment and make their own changes from there.

Agent evaluations are a good example. An eval might run the same task across different system prompts, skills, models, or agent versions. To compare the results, everything else has to stay fixed: the repository, dependencies, tools, and input files. One snapshot can start many isolated environments from the same baseline, reducing setup time and preventing environment drift from affecting the results.

Snapshots also complement the new system image we introduced above. An agent can start from `cloudflare/debian-trixie`, set up its environment with `exec()`, and save the result as a snapshot. Future sandboxes then start from that snapshot with the repository, dependencies, and toolchain already in place.

With snapshots available through the new `durable_object` scheduling policy, Containers can act as persistent agent workspaces. Compute can stop when work pauses, and a new Container can start from the latest snapshot to pick up where it left off.

## The Durable Object advantage for agent sandboxes

The faster scheduling path, runtime image selection, and filesystem snapshots all come from the same design choice: leaning further **into** the Durable Object as the controller for its attached Container.

Agent systems need a programmable, stateful environment outside the Container to keep state, hold credentials, and control the sandbox’s lifecycle. You can run the agent there, following the “ [<u>decoupling the brain from the hands</u>](https://www.anthropic.com/engineering/managed-agents)” pattern described by Anthropic: when the agent is separate from the sandbox where it works, the agent stays available while its sandboxes and tools can start, stop, fail, or be replaced independently. Or, if you run the agent inside the sandbox, the outside environment lets you supervise it and report progress back to the user.

This is where the Durable Object and Container architecture becomes uniquely well-suited. Every Container is attached to a stateful Durable Object with its own compute and storage running right next to it. You can run the agent in the Durable Object and use the Container as its workspace, or run the agent in the Container and use the Durable Object to supervise it. Add [<u>Dynamic Workers</u>](https://developers.cloudflare.com/dynamic-workers/) for lightweight isolated execution, and an application can choose the execution environment each task requires.

What’s new with the `durable_object` scheduling policy is that the Durable Object can now control its Container directly, with no wrapper class in between. `exec()` runs directly in the Workers runtime, and outbound request interception, runtime image and instance selection, and filesystem snapshots are all available on `ctx.container.` You can combine them with Durable Object storage, alarms, WebSockets, RPC and the rest of your code.

This makes the Container a compute extension of the Durable Object. The Container supplies the Linux environment, while your Durable Object retains the sandbox’s identity, state, policy, and lifecycle. That split is especially well-suited for several patterns:

![]()![image2.png](https://blog.cloudflare.com/_image?href=https%3A%2F%2Fblog.cloudflare.com%2F_emdash%2Fapi%2Fmedia%2Ffile%2F01M3RZHX4W63MBJM5R8X7VVE2G.01M3RZHXT3X1AAZRW96ZVHJNB2.png&w=715&h=427&f=webp&fit=cover&position=center)

**An agent can remain available while its Linux workspace sleeps.** The agent loop can run in the Durable Object, where it maintains session state, communicates with the user over WebSockets, and calls models. It can wake the Container when it needs a shell, compiler, or development server, then stop that compute while it waits for the user or model, paying nothing for idle Linux compute.

**The Durable Object can program the security boundary around its Container.** It can remember which services, repositories, and operations a user has authorized, then update the Container’s Outbound Request Handler to inject newly granted credentials, enforce new policies, or record additional activity. This resembles the [<u>Gatekeeper pattern used by Cloudflare OS</u>](https://blog.cloudflare.com/cloudflare-os/), applied to each agent computer.

**Evals and reinforcement learning systems can supervise each attempt from outside the environment being tested.** A coordinator snapshots a base workspace and forks it into N attempts, each with its own Durable Object and a Container. Each Durable Object runs its attempt, monitors the run, and preserves the result, even if the Container crashes. The coordinator grades the attempts, snapshots the best one, and forks again from there. 

These patterns don’t fit one generic lifecycle. Native APIs let you combine the Container with the Durable Object primitives your application needs, while still using higher-level utilities where they help. You keep control over the Container’s lifecycle, policy, and state.

### What this means for the Container class and Sandbox SDK

When we launched Containers, we deliberately hid the Durable Object behind the Container class. We wanted sandboxes to feel familiar and match what developers expected from other platforms. The Sandbox SDK was built on that class, and it filled real gaps: back then, the runtime had no native command execution, outbound request interception, or snapshots, so we built them in userspace.

Since then, agent workspaces have become one of the main workloads shaping Containers, and the cost of that abstraction has become clear. By hiding the Durable Object, we made it harder for you to see and combine the identity, state, and coordination it provides with the Container it controls. Nearly every team we worked with needed something slightly different from the generic lifecycle: their own sleep policy, their own credential handling, their own way of tracking eval runs.

These capabilities are now native, so we're making the Durable Object explicit in the developer experience:

- New capabilities are native-only. The `durable_object` scheduling policy, faster startup, runtime image and instance selection, and filesystem snapshots are available only through `ctx.container`.
- We'll maintain the `Container` class and legacy `Sandbox` class through December 31, 2026. Existing deployments keep running after that date, but the classes won't get updates. We recommend migrating to `ctx.container`.
- Sandbox SDK 1.0 is a set of utilities, not a base class. Its helpers work inside your own Durable Object class, alongside ctx.container.
- For a higher-level environment, `@cloudflare/computer` combines Dynamic Workers and Containers with a synchronized filesystem.

Migrating usually means changing extends `Container` to extends `DurableObject` and calling `this.ctx.container` directly. See the [<u>migration guide</u>](https://developers.cloudflare.com/containers/guides/migrate-to-durable-object-container-api/) for details. Or, get started with your agents:

Copy promptCopy promptPrompt copied!

## Get started

Today, most agents are measured by what they can accomplish in a single session. As agents take responsibility for projects that unfold across hours, days, and weeks, the environments where they work need to keep up.

We want every agent to be able to create the sandbox for the task at hand, release the compute when work pauses, and return to the same workspace when the project continues. Today’s changes bring us closer to sandboxes that are as programmable, persistent, and ready to work as the agents using them.

Try the new durable\_object scheduling policy, available to all today in public beta, and see what faster startup, filesystem snapshots, and runtime configuration unlock for your agents:

- [<u>Use the new Durable Object scheduling policy for Containers</u>](https://developers.cloudflare.com/containers/api/durable-object-container/)
- [<u>Save and restore files with filesystem snapshots</u>](https://developers.cloudflare.com/containers/guides/snapshots/)
- [<u>Deploy a full sandbox example using the Durable Object scheduling policy from GitHub</u>](https://github.com/cloudflare/containers-demos/tree/main/sandbox)
- Start from an integration for [<u>Cursor Cloud Agents</u>](https://developers.cloudflare.com/sandbox/tutorials/cursor-cloud-agents/), [<u>Devin Outposts</u>](https://developers.cloudflare.com/sandbox/tutorials/devin-outposts/), or the [<u>OpenAI Agents API</u>](https://developers.cloudflare.com/sandbox/tutorials/openai-agents-api/)

*Acknowledgements: This project was also made possible by the contributions of Greg Anders, Andrew Martinez, Nafeez Nazer, Kian Newman-Hazel, Sebastien Pahl, Naresh Ramesh, Cody Roseborough, Nikita Sharma, and Sarah Snell.*

![]()

Discuss Online

[Discuss on Hacker News (opens in a new tab)](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fblog.cloudflare.com%2Ffaster-agent-sandboxes%2F&t=Cloudflare%20Containers%2C%20rebuilt%20to%20scale%20agent%20sandboxes) [Discuss on X (opens in a new tab)](https://x.com/intent/post?text=Cloudflare+Containers%2C+rebuilt+to+scale+agent+sandboxes&url=https%3A%2F%2Fblog.cloudflare.com%2Ffaster-agent-sandboxes%2F) [Share on LinkedIn (opens in a new tab)](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fblog.cloudflare.com%2Ffaster-agent-sandboxes%2F) [Share on Bluesky (opens in a new tab)](https://bsky.app/intent/compose?text=Cloudflare+Containers%2C+rebuilt+to+scale+agent+sandboxes+https%3A%2F%2Fblog.cloudflare.com%2Ffaster-agent-sandboxes%2F) [Share on Mastodon (opens in a new tab)](https://mastodonshare.com/?text=Cloudflare+Containers%2C+rebuilt+to+scale+agent+sandboxes&url=https%3A%2F%2Fblog.cloudflare.com%2Ffaster-agent-sandboxes%2F) [Share on Threads (opens in a new tab)](https://www.threads.net/intent/post?text=Cloudflare+Containers%2C+rebuilt+to+scale+agent+sandboxes+https%3A%2F%2Fblog.cloudflare.com%2Ffaster-agent-sandboxes%2F)

## Related tags

[AI](https://blog.cloudflare.com/tag/ai/) [Birthday Week](https://blog.cloudflare.com/tag/birthday-week/) [Containers](https://blog.cloudflare.com/tag/containers/) [Developer Platform](https://blog.cloudflare.com/tag/developer-platform/) [Durable Objects](https://blog.cloudflare.com/tag/durable-objects/) [Internship Experience](https://blog.cloudflare.com/tag/internship-experience/) [Workers](https://blog.cloudflare.com/tag/workers-1/)

Follow on Social Media

- ![Cloudflare](https://blog.cloudflare.com/images/placeholder__cloudflare.png)Cloudflare [Cloudflare RSS feed (opens in a new tab)](https://blog.cloudflare.com/rss/) [Cloudflare on Twitter (opens in a new tab)](https://x.com/Cloudflare) [Cloudflare on LinkedIn (opens in a new tab)](https://www.linkedin.com/company/cloudflare-inc-) [Cloudflare on YouTube (opens in a new tab)](https://www.youtube.com/cloudflare) [Cloudflare on Instagram (opens in a new tab)](https://instagram.com/cloudflare) [Cloudflare on GitHub (opens in a new tab)](https://github.com/cloudflare) [Cloudflare on Bluesky (opens in a new tab)](https://bsky.app/profile/cloudflare.social) [Cloudflare on Threads (opens in a new tab)](https://www.threads.com/@cloudflare) [Cloudflare on TikTok (opens in a new tab)](https://www.tiktok.com/@cloudflare)
- ![Thomas Gauvin](https://blog.cloudflare.com/_image?href=https%3A%2F%2Fblog.cloudflare.com%2F_emdash%2Fapi%2Fmedia%2Ffile%2F01KW45J4TRGWP46SWRWBD5YV1M.jpg&w=64&h=64&f=webp&fit=cover&position=center) [Thomas Gauvin](https://blog.cloudflare.com/author/thomas-gauvin/) [Thomas Gauvin website (opens in a new tab)](https://thomasgauvin.com/) [Thomas Gauvin on Twitter (opens in a new tab)](https://x.com/thomasgauvin)

## Subscribe to receive notifications of new posts

Email address

We’ll never share your email address.

Subscribe

Thanks for subscribing! Check your inbox to confirm.

Search is temporarily unavailable.

[Login opens in a new tab](https://dash.cloudflare.com/login) [Dashboard opens in a new tab](https://dash.cloudflare.com) [Contact Sales opens in a new tab](https://www.cloudflare.com/resource/contact-enterprise-sales/) [Start Building opens in a new tab](https://dash.cloudflare.com/sign-up)

[opens in a new tab](https://x.com/cloudflare) [ opens in a new tab](https://www.linkedin.com/company/cloudflare-inc-) [ opens in a new tab](https://blog.cloudflare.com/rss/)

<details>

<summary>

All Categories

</summary>

- <a href="https://blog.cloudflare.com/tag/ai/">AI</a>
- <a href="https://blog.cloudflare.com/tag/developers/">Developers</a>
- <a href="https://blog.cloudflare.com/tag/cloudflare-radar/">Radar</a>
- <a href="https://blog.cloudflare.com/tag/product-news/">Product News</a>
- <a href="https://blog.cloudflare.com/tag/security/">Security</a>
- <a href="https://blog.cloudflare.com/tag/policy/">Policy &amp; Legal</a>
- <a href="https://blog.cloudflare.com/tag/zero-trust/">Zero Trust</a>
- <a href="https://blog.cloudflare.com/tag/speed-and-reliability/">Speed &amp; Reliability</a>
- <a href="https://blog.cloudflare.com/tag/life-at-cloudflare/">Life at Cloudflare</a>
- <a href="https://blog.cloudflare.com/tag/partners/">Partners</a>

</details>

<details>

<summary>

English

</summary>

- Switch Site Language
- <a href="https://blog.cloudflare.com/">English</a>
- <a href="https://blog.cloudflare.com/de-de/">Deutsch</a>
- <a href="https://blog.cloudflare.com/es-es/">Español</a>
- <a href="https://blog.cloudflare.com/es-la/">Español (Latinoamérica)</a>
- <a href="https://blog.cloudflare.com/fr-fr/">Français</a>
- <a href="https://blog.cloudflare.com/it-it/">Italiano</a>
- <a href="https://blog.cloudflare.com/ja-jp/">日本語</a>
- <a href="https://blog.cloudflare.com/ko-kr/">한국어</a>
- <a href="https://blog.cloudflare.com/zh-tw/">繁體中文</a>
- <a href="https://blog.cloudflare.com/zh-cn/">简体中文</a>
- <a href="https://blog.cloudflare.com/pt-br/">Português</a>
- <a href="https://blog.cloudflare.com/ru-ru/">Русский</a>
- <a href="https://blog.cloudflare.com/id-id/">Bahasa Indonesia</a>
- <a href="https://blog.cloudflare.com/th-th/">ภาษาไทย</a>
- <a href="https://blog.cloudflare.com/vi-vn/">Tiếng Việt</a>
- <a href="https://blog.cloudflare.com/pl-pl/">Polski</a>
- <a href="https://blog.cloudflare.com/ar-ar/">العربية</a>
- <a href="https://blog.cloudflare.com/he-il/">עברית</a>
- <a href="https://blog.cloudflare.com/sv-se/">Svenska</a>
- <a href="https://blog.cloudflare.com/nl-nl/">Nederlands</a>
- <a href="https://blog.cloudflare.com/tr-tr/">Türkçe</a>

</details>

LightDark

```json
{"@context":"https://schema.org","@type":"BlogPosting","headline":"Cloudflare Containers, rebuilt to scale agent sandboxes","description":"Cloudflare Containers now start 6x faster, let your agent choose each sandbox's image and instance type at runtime, and support filesystem snapshots in public beta, all controlled from a Durable Object.","image":"https://blog.cloudflare.com/_emdash/api/media/file/01M3S8R4W0018F4ABEE2X1ZH4G.01M3S8R5VDFQC24Z3Q4E6RBE3E.png","url":"https://blog.cloudflare.com/faster-agent-sandboxes/","datePublished":"2026-09-30T12:58:00.000Z","dateModified":"2026-09-30T13:41:49.981Z","publisher":{"@type":"Organization","name":"Cloudflare Blog"},"mainEntityOfPage":{"@type":"WebPage","@id":"https://blog.cloudflare.com/faster-agent-sandboxes/"}}
```
