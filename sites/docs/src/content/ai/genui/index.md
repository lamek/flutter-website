---
title: GenUI and A2UI for Flutter
shortTitle: GenUI and A2UI
description: >-
  Learn how to build agent-generated user interfaces in Flutter using A2UI
  (a2ui_flutter, a2ui_core, and a2ui_agent).
next:
  title: Main components and concepts
  path: /ai/genui/components
---

:::note
In Flutter, generative UI (GenUI) is built on the open [A2UI][]
(Agent-to-User Interface) protocol. For new projects, install
[`a2ui_flutter`][] and [`a2ui_core`][] (plus [`a2ui_agent`][] when calling
models directly from Dart). Together, these packages replace the deprecated
[`genui`][] and `genui_a2a` packages. Do not use the discontinued
`flutter_genui` or `genui_a2ui` packages.
:::

## What are A2UI and GenUI?

[A2UI][] (Agent-to-User Interface) is an open protocol and SDK suite for
building agent-generated user interfaces. Instead of returning a wall of plain
text or executing untrusted code on the client, an AI agent streams declarative
JSON messages that describe UI surfaces, components from a registered widget
catalog, and a reactive data model. Your Flutter app renders those surfaces
using its own trusted Flutter widgets.

For example, instead of describing a list of options in text, an agent can
present a graphical UI with a row of labeled buttons, a slider, and a date
picker. As the user interacts with those widgets, state changes and action
events flow back to the agent, creating a high-bandwidth loop between the user,
your Flutter UI, and the AI agent.

| Package | Role |
| :--- | :--- |
| [`a2ui_flutter`][] | Renders A2UI surfaces as native Flutter widgets and provides the Material basic catalog. |
| [`a2ui_core`][] | Processes A2UI protocol messages (v1.0 and v0.9), manages surface state, and evaluates data bindings and functions. |
| [`a2ui_agent`][] | Negotiates catalogs with the renderer, generates system prompt instructions for the model, and parses streaming LLM responses into A2UI messages. |

## When should you use A2UI?

Use A2UI on Flutter whenever you want an AI agent or LLM to generate or update
native UI at runtime:

* **Rich interactive flows.** Replace static text replies with clickable
  product carousels, dynamic trip-planning forms, or multi-step wizards built
  from your own widget catalog.
* **Cross-platform agent backends.** Drive Flutter (mobile, web, desktop), web
  (React, Angular, Lit), and native clients (SwiftUI, Jetpack Compose) from a
  single A2UI agent stream.
* **Remote or multi-agent boundaries.** Render UI safely from remote or
  third-party agents using declarative JSON messages rather than executable
  client code.

To learn more about the A2UI protocol and cross-platform SDKs, check out
[a2ui.org][A2UI] and the [Getting started with GenUI video][]:

<YouTubeEmbed id="nWr6eZKM6no"
    title="Getting started with GenUI"></YouTubeEmbed>

Also, check out the Flutter + A2UI = GenUI video from
Google I/O 2026:

<YouTubeEmbed id="tXeyaV1gVJk"
    title="Flutter + A2UI = GenUI"></YouTubeEmbed>

[A2UI]: https://a2ui.org/
[`a2ui_flutter`]: {{site.pub-pkg}}/a2ui_flutter
[`a2ui_core`]: {{site.pub-pkg}}/a2ui_core
[`a2ui_agent`]: {{site.pub-pkg}}/a2ui_agent
[`genui`]: {{site.pub-pkg}}/genui
[Getting started with GenUI video]: https://www.youtube.com/watch?v=nWr6eZKM6no
