---
title: A2UI and GenUI main components and concepts
breadcrumb: Main components & concepts
description: >-
  Learn the main components and concepts of A2UI for Flutter (a2ui_flutter,
  a2ui_core, and a2ui_agent) and the legacy GenUI SDK.
prev:
  title: GenUI and A2UI overview
  path: /ai/genui
next:
  title: Get started with A2UI on Flutter
  path: /ai/genui/get-started
---

:::note
For new Flutter projects, use the modular [A2UI][] packages:
[`a2ui_flutter`][], [`a2ui_core`][], and [`a2ui_agent`][]. These packages
replace the deprecated [`genui`][] package. Do not use the discontinued
`flutter_genui` or `genui_a2ui` packages.
:::

## A2UI components (`a2ui_flutter`, `a2ui_core`, and `a2ui_agent`)

The A2UI Flutter stack separates protocol processing, widget rendering, and
model prompting into three focused packages:

`MessageProcessor` (`a2ui_core`)
: Processes incoming A2UI protocol messages (`createSurface`,
  `updateComponents`, `updateDataModel`, `deleteSurface`), maintains the active
  `SurfaceGroupModel`, and emits outbound `action`, `error`, and function-call
  messages back to the agent through `outboundListener`.

`SurfaceModel` and `DataModel` (`a2ui_core`)
: Hold the reactive component graph and observable data state for a single UI
  surface. Components bind properties to JSON Pointer paths in the `DataModel`
  so only affected widgets rebuild when data changes.

`A2uiSurface` (`a2ui_flutter`)
: A Flutter widget that renders a `SurfaceModel<ComponentImplementation>` using
  your registered widget catalogs.

`CatalogImplementation` and `basicCatalog()` (`a2ui_flutter`)
: Define the vocabulary of widgets and functions the agent is allowed to use.
  `basicCatalog()` provides the standard Material widget catalog (`Row`,
  `Column`, `List`, `Card`, `Tabs`, `Modal`, `Divider`, `Text`, `Image`,
  `Icon`, `Video`, `AudioPlayer`, `Button`, `TextField`, `CheckBox`,
  `ChoicePicker`, `Slider`, and `DateTimeInput`), which you can extend with
  custom `ComponentImplementation` definitions.

`A2uiGenerator` and `ResponseStream` (`a2ui_agent`)
: Used when your Dart app or server calls an LLM directly. `A2uiGenerator`
  negotiates catalog capabilities and builds the system prompt snippet, while
  `ResponseStream` (or `StreamedAnswer`) parses streaming LLM output into A2UI
  messages.

## How the interaction cycle works

Whether your agent runs on a remote server or inside your Flutter app, the A2UI
interaction loop follows these steps:

 1. **Catalog registration**

    Your Flutter app initializes a `MessageProcessor` with `basicCatalog()`
    and any custom `CatalogImplementation` objects.

 2. **User input**

    The user enters a prompt or interacts with a rendered widget, and your app
    sends the request (along with `a2uiRendererCapabilities` and
    `a2uiRendererDataModel` metadata) to the agent.

 3. **Agent generation**

    The agent (or an LLM guided by `A2uiGenerator` and `A2uiRequestProcessor`)
    streams back A2UI protocol messages such as `createSurface`,
    `updateComponents`, and `updateDataModel`.

 4. **Message processing**

    Your app feeds the incoming JSON messages into
    `processor.processMessages()`, which updates the reactive `SurfaceModel`
    and `DataModel`.

 5. **Native widget rendering**

    `processor.model.onSurfaceCreated` notifies your app of new surfaces, and
    `A2uiSurface` builds the corresponding Flutter widgets.

 6. **User actions and two-way data binding**

    When the user edits an input widget (such as `TextField` or `Slider`), the
    bound path in `DataModel` updates automatically. When the user triggers an
    action (such as tapping a `Button`), `MessageProcessor` emits a
    `RendererToAgentMessage` through `outboundListener` so the agent can respond
    with the next UI update.

{:.steps}

## Legacy `package:genui` components (deprecated)

If you are maintaining an existing app built with the deprecated [`genui`][]
package (prior to migrating to `a2ui_flutter` and `a2ui_core`), its facade
classes map to the concepts above as follows:

`Conversation`
: The primary facade and entry point in `package:genui`. It wraps
  `SurfaceController`, manages conversation history, and orchestrates the
  generative UI loop.

`Catalog`
: A collection of `CatalogItem` objects that defines the set of widgets the AI
  is allowed to use in `package:genui`.

`DataModel`
: A centralized, observable store for dynamic UI state in `package:genui`,
  accessed through a `DataContext` passed to each widget builder.

`A2uiTransportAdapter`
: A bridge in `package:genui` that parses raw text streams from an LLM into
  `A2uiMessage` commands for `SurfaceController`.

`SurfaceController`
: Handles `A2uiMessage` processing, manages the `DataModel`, and maintains the
  state of UI surfaces in `package:genui`.

To migrate an existing `genui` app to `a2ui_flutter` and `a2ui_core`, use the
[A2UI migration skill][]. The next section walks you through adding A2UI to
your Flutter app.

[A2UI]: https://a2ui.org/
[A2UI migration skill]: https://github.com/a2ui-project/a2ui/blob/main/.agents/skills/a2ui-migration/SKILL.md
[`a2ui_flutter`]: {{site.pub-pkg}}/a2ui_flutter
[`a2ui_core`]: {{site.pub-pkg}}/a2ui_core
[`a2ui_agent`]: {{site.pub-pkg}}/a2ui_agent
[`genui`]: {{site.pub-pkg}}/genui
