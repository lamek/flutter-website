---
title: Input and events
description: How user input, two-way bindings, and action events work in A2UI and GenUI Flutter applications.
prev:
  title: Get started with A2UI on Flutter
  path: /ai/genui/get-started
---

This guide explains how user interactions and data updates flow from your
Flutter widgets back to the AI agent in [A2UI][] (`a2ui_flutter` and
`a2ui_core`) and the legacy `genui` package.

:::note
For new Flutter projects, use [`a2ui_flutter`][] and [`a2ui_core`][] (plus
[`a2ui_agent`][] when calling models directly from Dart). Together, these
packages replace the deprecated [`package:genui`][] package.
:::

[A2UI]: https://a2ui.org/
[`a2ui_flutter`]: {{site.pub-pkg}}/a2ui_flutter
[`a2ui_core`]: {{site.pub-pkg}}/a2ui_core
[`a2ui_agent`]: {{site.pub-pkg}}/a2ui_agent

## Handling actions and bindings in `a2ui_flutter`

In `a2ui_flutter` and `a2ui_core`, user input flows back to the agent through
two mechanisms:

1. **Two-way data bindings (`props.writable`)**: When a component property is
   bound to a JSON Pointer path in the surface's `DataModel` (such as
   `/user/name`), calling `props.writable(key)?.set(newValue)` updates the
   `DataModel` in place and rebuilds any widgets bound to that path.
2. **Action messages (`props.action` and `outboundListener`)**: When a user
   activates a control (such as tapping a `Button`), calling
   `props.action(key)?.call()` dispatches an A2UI `action` message. Pass
   `outboundListener` with `unifiedOutbound: true` to your `MessageProcessor`
   to forward outbound `action`, `error`, and function-call messages to your
   agent alongside the current `processor.getRendererDataModel()`:

```dart
void sendToAgent(RendererToAgentMessage message) => send(
  RendererToAgentMessagePayload.of(message).toJson(),
  metadata: {'a2uiRendererDataModel': ?processor.getRendererDataModel()},
);

final processor = MessageProcessor<ComponentImplementation>(
  catalogs: [basicCatalog()],
  outboundListener: sendToAgent,
  unifiedOutbound: true,
);
```

If your agent is written in Dart using `a2ui_agent`, use
`RendererToAgentMessageProcessor` to handle incoming `action`, `error`, and
`callAgentFunction` messages from the renderer.

## Legacy `package:genui` event flow (deprecated)

In the legacy `genui` architecture, the UI is driven by the AI,
but user interactions (like clicking a button or submitting a form)
must be communicated back to the AI agent.
This allows the agent to update the UI or perform actions
in response to user input.

The flow of an event is as follows:

1. Interaction: User interacts with a widget;
   for example, the user taps a button.
2. Capture: The widget implementation dispatches a `UiEvent`.
3. Processing: The framework adds context
   (such as a `surfaceId` or data model values)
   and forwards the event.
4. Transmission: The Flutter widget generates the event,
   adds the appropriate context, and routes
   it to the AI through the `ContentGenerator`,
   which forwards it to the AI agent.

## Defining events

### Protocol level

The A2UI protocol defines an `action` message used to report events.
An `action` contains:

* `name`: The name of the action
  (defined by the AI when generating the component).
* `surfaceId`: The ID of the UI surface where the event occurred.
* `sourceComponentId`: The ID of the component that triggered the event.
* `context`: A JSON object containing data relevant to the event.
* `timestamp`: When the event occurred.

### Dart implementation

In [`package:genui`][], user events are represented by the
`UiEvent` extension type and its concrete implementation
`UserActionEvent`.

[`package:genui`]: {{site.pub-pkg}}/genui

The following structures are defined in
`lib/src/model/ui_models.dart`:

```dart title="lib/src/model/ui_models.dart"
/// A data object that represents a user interaction event in the UI.
extension type UiEvent.fromMap(JsonMap _json) { ... }

/// A UI event that represents a user action.
extension type UserActionEvent.fromMap(JsonMap _json) implements UiEvent {
  UserActionEvent({
    String? surfaceId,
    required String name,
    required String sourceComponentId,
    JsonMap? context,
    // ...
  }) : ...
}
```

## Capturing events in widgets

Widgets in GenUI are defined in a `Catalog`,
which includes information about what events
the widget can send to the AI.
The AI can then send
information about how to communicate those events back.
When you implement a custom widget (or use the standard widgets),
you use the `dispatchEvent` method in `CatalogItemContext`
to dispatch events.

### Example: Button implementation {: #button-example}

The following example shows how a `Button` widget typically captures
a tap and dispatches an event. It retrieves the action definition
(provided by the AI) from its properties,
resolves any data bindings in the context, and sends the event.

```dart
// Inside a CatalogItem widgetBuilder:
widgetBuilder: (itemContext) {
  // 1. Extract action data from the component properties.
  final buttonData = _ButtonData.fromMap(itemContext.data as JsonMap);
  final JsonMap actionData = buttonData.action;
  final actionName = actionData['name'] as String;

  // 2. Extract context definition (which data to send back).
  final List<Object?> contextDefinition =
      (actionData['context'] as List<Object?>?) ?? <Object?>[];

  return ElevatedButton(
    onPressed: () {
      // 3. Resolve the context values from the data model.
      final JsonMap resolvedContext = resolveContext(
        itemContext.dataContext,
        contextDefinition,
      );

      // 4. Dispatch the event.
      itemContext.dispatchEvent(
        UserActionEvent(
          name: actionName,
          sourceComponentId: itemContext.id,
          context: resolvedContext,
        ),
      );
    },
    child: /* ... */
  );
},
```

## Event processing pipeline

Once `dispatchEvent` is called,
the event travels through the GenUI core layers.

### Surface

The `Surface` widget (in `lib/src/core/surface.dart`)
wraps the rendered widgets.
It provides the dispatchEvent callback implementation.

When `_dispatchEvent` is called:

1. It automatically injects the `surfaceId` into the event,
   ensuring the AI knows which surface the interaction came from.
2. It delegates handling to the `SurfaceHost`
   (implemented by `SurfaceController`).

```dart
// Surface implementation details
void _dispatchEvent(UiEvent event) {
  // ...
  final Map<String, Object?> eventMap = {
    ...event.toMap(),
    surfaceIdKey: widget.surfaceId, // Inject surfaceId
  };
  final UiEvent newEvent = UserActionEvent.fromMap(eventMap);
  widget.host.handleUiEvent(newEvent);
}
```

### SurfaceController

The `SurfaceController` (in `lib/src/core/surface_controller.dart`)
is the central hub for managing UI state.

When `handleUiEvent` is called, it does the following:

1.  Verifies the event type.
2.  Wraps the event in the `action` JSON envelope
   required by the protocol.
3.  Emits a `UserUiInteractionMessage` on its `onSubmit` stream.

```dart
// SurfaceController implementation details
@override
void handleUiEvent(UiEvent event) {
  if (event is! UserActionEvent) return;

  // Wrap in protocol 'action' envelope
  final String eventJsonString = jsonEncode({'action': event.toMap()});

  // Emit for listeners (like Conversation)
  _onSubmit.add(UserUiInteractionMessage.text(eventJsonString));
}
```

## Transmission to AI

The final step sends the event to the AI Agent.
This is typically handled by `Conversation`
(in `lib/src/facade/conversation.dart`).
The `Conversation` listens to the `onSubmit` stream
from the message processor.

```dart
// Conversation constructor
_userEventSubscription = surfaceController.onSubmit.listen(sendRequest);
```

When an event is received, the `sendRequest` method:

1. Wraps the `UserUiInteractionMessage` back to the developer's client code.
2. The custom integration or predefined transport adapter forwards
   the message to the LLM agent network transport.

The AI Agent receives this JSON message, processes the user action,
and might stream back new `surfaceUpdate` or `dataModelUpdate` messages
to modify the UI, or some other action, completing the full interaction loop.

