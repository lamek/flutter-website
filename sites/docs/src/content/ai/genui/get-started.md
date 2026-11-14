---
title: Get started with A2UI and GenUI for Flutter
shortTitle: Get started with A2UI
breadcrumb: Get started
description: >-
  Learn how to add A2UI (a2ui_flutter, a2ui_core, and a2ui_agent) or the
  legacy GenUI SDK to your Flutter app.
next:
  title: Input and events
  path: /ai/genui/input-events
prev:
  title: Main components and concepts
  path: /ai/genui/components
---

This guide explains how to get started with [A2UI][] (Agent-to-User Interface)
for Flutter using [`a2ui_flutter`][], [`a2ui_core`][], and [`a2ui_agent`][].
The core architecture and classes are described on the [main components][] page.

:::note
For new Flutter apps, use [`a2ui_flutter`][] and [`a2ui_core`][] (plus
[`a2ui_agent`][] when calling an LLM directly from Dart). Together, these
packages replace the deprecated [`genui`][] and `genui_a2a` packages. Do not
use the discontinued `flutter_genui` or `genui_a2ui` packages.
:::

Use the following instructions to add A2UI to a new project created with
[`flutter create`][] or to an existing Flutter app.

[A2UI]: https://a2ui.org/
[`a2ui_flutter`]: {{site.pub-pkg}}/a2ui_flutter
[`a2ui_core`]: {{site.pub-pkg}}/a2ui_core
[`a2ui_agent`]: {{site.pub-pkg}}/a2ui_agent
[`genui`]: {{site.pub-pkg}}/genui
[main components]: /ai/genui/components
[`flutter create`]: /reference/create-new-app

## Configure your agent provider

Choose the integration setup that matches your architecture:

**A2UI (`a2ui_flutter` and `a2ui_core`)**
: Recommended for all new Flutter projects. Render A2UI v1.0 and v0.9 surfaces
  from any remote agent server, or add `a2ui_agent` to prompt an LLM directly
  from your Dart app.

**Firebase AI Logic (legacy `genui`)**
: Legacy [`genui`][] integration where interactions with the Gemini API run in
  your Flutter client through Firebase AI Logic.

**GenUI A2UI (legacy `genui_a2a`)**
: Legacy [`genui`][] and `genui_a2a` integration for Agent-to-Agent (A2A)
  servers.

**Build your own**
: Connect a custom LLM client or transport adapter.

<Tabs key="agent-provider" wrapped="true">

<Tab name="A2UI (Recommended)">

Use [`a2ui_flutter`][] and [`a2ui_core`][] to render A2UI surfaces in your
Flutter app:

 1. Add `a2ui_flutter` and `a2ui_core` to your `pubspec.yaml` file:

    ```console
    $ flutter pub add a2ui_flutter a2ui_core
    ```

 2. Create a `MessageProcessor` with `basicCatalog()` and render an
    `A2uiSurface` inside a `MaterialApp` and `Scaffold`:

    ```dart title="lib/main.dart"
    import 'package:a2ui_core/a2ui_core.dart';
    import 'package:a2ui_flutter/a2ui_flutter.dart';
    import 'package:a2ui_flutter/basic_catalog.dart';
    import 'package:flutter/material.dart';

    void main() => runApp(const MaterialApp(home: HelloA2ui()));

    class HelloA2ui extends StatefulWidget {
      const HelloA2ui({super.key});

      @override
      State<HelloA2ui> createState() => _HelloA2uiState();
    }

    class _HelloA2uiState extends State<HelloA2ui> {
      final processor = MessageProcessor<ComponentImplementation>(
        catalogs: [basicCatalog()],
      );
      SurfaceModel<ComponentImplementation>? surface;

      @override
      void initState() {
        super.initState();
        processor.model.onSurfaceCreated.subscribe(
          (created) => setState(() => surface = created),
        );
        processor.model.onAction.subscribe(
          (action) => debugPrint('Action: ${action.name}'),
        );

        // In a production app, these messages stream from your agent.
        processor.processMessages([
          {
            'version': '1.0',
            'createSurface': {
              'surfaceId': 'hello',
              'catalogId': BasicCatalog.v1_0Id,
            },
          },
          {
            'version': '1.0',
            'updateComponents': {
              'surfaceId': 'hello',
              'components': [
                {
                  'id': 'root',
                  'component': 'Column',
                  'children': ['greeting', 'button'],
                },
                {'id': 'greeting', 'component': 'Text', 'text': 'Hello, A2UI!'},
                {
                  'id': 'button',
                  'component': 'Button',
                  'child': 'label',
                  'action': {
                    'event': {'name': 'greet'},
                  },
                },
                {'id': 'label', 'component': 'Text', 'text': 'Say hi'},
              ],
            },
          },
        ]);
      }

      @override
      void dispose() {
        processor.dispose();
        super.dispose();
      }

      @override
      Widget build(BuildContext context) {
        final currentSurface = surface;
        return Scaffold(
          body: currentSurface == null
              ? null
              : A2uiSurface(surface: currentSurface),
        );
      }
    }
    ```

 3. If your app calls a model directly without a separate agent server, add
    [`a2ui_agent`][] to generate the A2UI system prompt snippet and parse the
    streamed response:

    ```console
    $ flutter pub add a2ui_agent
    ```

    ```dart
    final generator = A2uiGenerator(catalogs: [CatalogConfig(basicCatalog())]);
    final processor = generator.createProcessor(rendererCapabilities);
    final stream = processor.openStream(state: RendererState.of(renderer));

    await for (final chunk in llmStream) {
      stream.feed(chunk);
    }
    stream.close();
    ```

To explore complete v1.0 and v0.9 example apps, check out the
[a2ui_flutter package README][] and the [A2UI documentation][A2UI].

[a2ui_flutter package README]: https://github.com/a2ui-project/a2ui/tree/main/dart/a2ui_flutter

</Tab>

<Tab name="Firebase AI Logic">

To connect to Gemini using the Firebase AI Logic SDK,
follow these instructions:

 1. [Create a new Firebase project][] using the Firebase Console.

 2. [Enable the Gemini API][] for that project.

 3. Follow the first three steps in [Firebase's Flutter setup guide][]
    to add Firebase to your app.

 4. Use `dart pub add` to add `genui` and `firebase_ai` as
    dependencies in your `pubspec.yaml` file:

    ```console
    $ dart pub add genui firebase_ai
    ```

 5. In your app's `main` method, ensure that the widget
    bindings are initialized and then initialize Firebase:

    ```dart
    import 'package:material_ui/material_ui.dart';
    import 'package:firebase_core/firebase_core.dart';
    import 'firebase_options.dart';

    void main() async {
      WidgetsFlutterBinding.ensureInitialized();
      await Firebase.initializeApp(
        options: DefaultFirebaseOptions.currentPlatform,
      );
      runApp(const MyApp());
    }
    ```

 6. Create an instance of the Firebase AI Logic generative model and wrap it
    with your `SurfaceController` and `A2uiTransportAdapter`:

    ```dart
    import 'package:genui/genui.dart';
    import 'package:firebase_ai/firebase_ai.dart';

    final catalog = Catalog([
      // ...
    ]);
    final catalogs = [catalog];

    final surfaceController = SurfaceController(catalogs: catalogs);
    
    final promptBuilder = PromptBuilder.chat(
      catalog: catalog,
      systemPromptFragments: ['You are a helpful assistant.'],
    );

    final model = FirebaseAI.agentPlatform().generativeModel(
      model: 'gemini-3.5-flash',
      systemInstruction: Content.system(promptBuilder.systemPromptJoined()),
    );

    // The Conversation wires transport -> controller internally.
    late final A2uiTransportAdapter transportAdapter;
    transportAdapter = A2uiTransportAdapter(onSend: (message) async {
      // final stream = model.generateContentStream(...);
      // await for (final chunk in stream) {
      //   transportAdapter.addChunk(chunk.text ?? '');
      // }
    });

    final conversation = Conversation(
      controller: surfaceController,
      transport: transportAdapter,
    );
    ```

[Create a new Firebase project]: https://support.google.com/appsheet/answer/10104995
[Enable the Gemini API]: https://firebase.google.com/docs/gemini-in-firebase/set-up-gemini
[Firebase's Flutter setup guide]: https://firebase.google.com/docs/flutter/setup
[`firebase_ai_logic`]: {{site.pub-pkg}}/firebase_ai_logic

</Tab>

<Tab name="GenUI A2UI">

An integration package for [`genui`][] and the 
[A2UI Streaming UI Protocol][]. This package allows
Flutter applications to connect to an Agent-to-Agent (A2UI)
server and render dynamic user interfaces generated by an
AI agent using the `genui` framework.

The main components in this package include:

* `A2uiAgentConnector`:
  Handles the low-level web socket communication with the
  A2A server, including sending messages and parsing stream events.
* `AgentCard`:
  A data class that holds metadata about the connected AI agent.

Follow these instructions:

 1. Set up dependencies:
    Use `dart pub add` to add `genui` and `genui_a2a` as
    dependencies in your `pubspec.yaml` file.

    ```console
    $ dart pub add genui genui_a2a
    ```

 2. Initialize `SurfaceController`:
    Set up `SurfaceController` with your widget `Catalog`s.

 3. Create `A2uiTransportAdapter`:
    Instantiate `A2uiTransportAdapter` to parse the messages.

 4. Create `A2uiAgentConnector`:
    Instantiate `A2uiAgentConnector`, providing the A2A server URI.

 5. Create `Conversation`:
    Pass the adapter and controller to the `Conversation`.

 6. Render with `Surface`:
    Use `Surface` widgets in your UI to display
    the agent-generated content.

 7. Send Messages:
    Use `connector.connectAndSend` or `Conversation.sendMessage` to send user input
    to the agent-generated content.

    ```dart
    import 'package:material_ui/material_ui.dart';
    import 'package:genui/genui.dart';
    import 'package:genui_a2a/genui_a2a.dart';
    import 'package:logging/logging.dart';

    void main() {
      // Setup logging.
      Logger.root.level = Level.ALL;
      Logger.root.onRecord.listen((record) {
        print('${record.level.name}: ${record.time}: ${record.message}');
        if (record.error != null) {
          print(record.error);
        }
        if (record.stackTrace != null) {
          print(record.stackTrace);
        }
      });

      runApp(const GenUIExampleApp());
    }

    class GenUIExampleApp extends StatelessWidget {
      const GenUIExampleApp({super.key});

      @override
      Widget build(BuildContext context) {
        return MaterialApp(
          title: 'A2UI Example',
          theme: ThemeData(
            primarySwatch: Colors.blue,
          ),
          home: const ChatScreen(),
        );
      }
    }

    class ChatScreen extends StatefulWidget {
      const ChatScreen({super.key});

      @override
      State<ChatScreen> createState() => _ChatScreenState();
    }

    class _ChatScreenState extends State<ChatScreen> {
      final TextEditingController _textController = TextEditingController();
      final SurfaceController _surfaceController =
          SurfaceController(catalogs: [BasicCatalogItems.asCatalog()]);
      late final A2uiTransportAdapter _transportAdapter;
      late final Conversation _uiAgent;
      late final A2uiAgentConnector _connector;
      final List<ChatMessage> _messages = [];

      @override
      void initState() {
        super.initState();
        
        // The Conversation wires transport -> controller internally.
        _transportAdapter = A2uiTransportAdapter(onSend: (message) async {
          // Implement sending to LLM if needed, or handled by connector
        });
        
        _connector = A2uiAgentConnector(
          // TODO: Replace with your A2A server URL.
          url: Uri.parse('http://localhost:8080'),
        );
        _uiAgent = Conversation(
          controller: _surfaceController,
          transport: _transportAdapter,
        );

        // Listen for messages from the remote agent.
        _connector.stream.listen(_surfaceController.handleMessage);

      }

      @override
      void dispose() {
        _textController.dispose();
        _uiAgent.dispose();
        _transportAdapter.dispose();
        _surfaceController.dispose();
        _connector.dispose();
        super.dispose();
      }

      void _handleSubmitted(String text) async {
        if (text.isEmpty) return;
        _textController.clear();
        final message = ChatMessage.user(text);
        setState(() {
          _messages.insert(0, message);
        });
        
        final responseText = await _connector.connectAndSend(message);
        
        // Handling response depends on your app's logic
      }

      @override
      Widget build(BuildContext context) {
        return Scaffold(
          appBar: AppBar(
            title: const Text('A2UI Example'),
          ),
          body: Column(
            children: <Widget>[
              Expanded(
                child: ListView.builder(
                  padding: const EdgeInsets.all(8.0),
                  reverse: true,
                  itemBuilder: (_, int index) =>
                      _buildMessage(_messages[index]),
                  itemCount: _messages.length,
                ),
              ),
              const Divider(height: 1.0),
              Container(
                decoration: BoxDecoration(color: Theme.of(context).cardColor),
                child: _buildTextComposer(),
              ),
              // Surface for the main AI-generated UI:
              SizedBox(
                height: 300,
                child: Surface(
                  surfaceContext: _surfaceController.contextFor('main_surface'),
                ),
              ),
            ],
          ),
        );
      }

      Widget _buildMessage(ChatMessage message) {
        return Container(
          margin: const EdgeInsets.symmetric(vertical: 10.0),
          child: Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: <Widget>[
              Container(
                margin: const EdgeInsets.only(right: 16.0),
                child: CircleAvatar(child: Text(message.role == ChatMessageRole.user ? 'U' : 'A')),
              ),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: <Widget>[
                    Text(message.role == ChatMessageRole.user ? 'User' : 'Agent',
                        style: const TextStyle(fontWeight: FontWeight.bold)),
                    Container(
                      margin: const EdgeInsets.only(top: 5.0),
                      child: Text(message.parts.whereType<TextPart>().map((e) => e.text).join('\n')),
                    ),
                  ],
                ),
              ),
            ],
          ),
        );
      }

      Widget _buildTextComposer() {
        return IconTheme(
          data: IconThemeData(color: Theme.of(context).colorScheme.secondary),
          child: Container(
            margin: const EdgeInsets.symmetric(horizontal: 8.0),
            child: Row(
              children: <Widget>[
                Flexible(
                  child: TextField(
                    controller: _textController,
                    onSubmitted: _handleSubmitted,
                    decoration:
                        const InputDecoration.collapsed(hintText: 'Send a message'),
                  ),
                ),
                Container(
                  margin: const EdgeInsets.symmetric(horizontal: 4.0),
                  child: IconButton(
                    icon: const Icon(Icons.send),
                    onPressed: () => _handleSubmitted(_textController.text),
                  ),
                ),
              ],
            ),
          ),
        );
      }
    }
    ```

The [example][] directory on pub.dev contains a
complete application demonstrating how to use this package.

[example]: {{site.pub-pkg}}/genui_a2a/example
[A2UI Streaming UI Protocol]: https://a2ui.org/

</Tab>

<Tab name="Build your own">

To use `genui` with another agent provider,
follow that provider's SDK documentation to implement a connection,
and stream its results into an `A2uiTransportAdapter`.

:::warning
`PromptBuilder.chat()` generates a system prompt that might be
**3,000–5,000+ tokens** long, which can
exceed the context window of on-device or small models.
If you are targeting an on-device LLM, consider:

- Writing a compact custom system prompt that covers only the
  A2UI `createSurface` to `updateComponents` flow.
- Using `systemPromptFragments` to pass only the portions of
  the schema your use case requires.
:::

</Tab>

</Tabs>

## Create the connection to an agent

If you build your Flutter project for iOS or macOS,
add this key to your `{ios,macos}/Runner/*.entitlements` file(s)
to enable outbound network requests:

```xml
<dict>
...
<key>com.apple.security.network.client</key>
<true/>
</dict>
```

Next, use the following instructions to connect your app
to your chosen agent provider.

 1. Create a `SurfaceController`, and provide it with the catalogs
    of widgets that you want to make available to the agent.
    Create an `A2uiTransportAdapter` to parse messages and connect it.

 2. Create a `PromptBuilder`, and provide it with a
    system instruction and the tools (functions
    you want the agent to be able to invoke).
    You should always include the tools provided by `SurfaceController`,
    but feel free to include others. Add this to your LLM system prompt.

 3. Create a `Conversation` using the instances of
    `SurfaceController` and `A2uiTransportAdapter`. Your app will
    primarily interact with this object to get things done.

    For example:

    ```dart
    class _MyHomePageState extends State<MyHomePage> {
      late final SurfaceController _surfaceController;
      late final A2uiTransportAdapter _transportAdapter;
      late final Conversation _conversation;

      @override
      void initState() {
        super.initState();

        // Create a SurfaceController with a widget catalog.
        // The BasicCatalogItems contain basic widgets for text, markdown, and images.
        _surfaceController = SurfaceController(catalogs: [BasicCatalogItems.asCatalog()]);

        // The Conversation wires transport -> controller internally.
        _transportAdapter = A2uiTransportAdapter(onSend: (message) async {
          // Implement sending to LLM and pipe chunks back.
        });

        final catalog = BasicCatalogItems.asCatalog();
        final promptBuilder = PromptBuilder.chat(
          catalog: catalog,
          systemPromptFragments: [
            '''
            You are an expert in creating funny riddles. Every time I give you a word,
            you should generate UI that displays one new riddle related to that word.
            Each riddle should have both a question and an answer.
            '''
          ],
        );

        // ... initialize your LLM Client of choice using promptBuilder.systemPromptJoined()

        // Create the Conversation to orchestrate everything.
        _conversation = Conversation(
          controller: _surfaceController,
          transport: _transportAdapter,
        );
        
        // Listen for surface lifecycle events:
        _conversation.events.listen((event) {
          if (event is ConversationSurfaceAdded) {
            _onSurfaceAdded(event);
          } else if (event is ConversationSurfaceRemoved) {
            _onSurfaceDeleted(event);
          }
        });
      }

      @override
      void dispose() {
        _textController.dispose();
        _conversation.dispose();
        _transportAdapter.dispose();
    
        super.dispose();
      }
    }
    ```
   
## Send messages and display the agent's responses

Send a request to the agent using the `sendRequest` method
in the `Conversation` class,
or by directly streaming into your LLM Client and pumping
the result stream to the adapter by using `_transportAdapter.addChunk`.

To receive and display generated UI:

  1. Listen to the `events` stream in `Conversation` to track the addition
     and removal of UI surfaces as they are generated.
     These events include a _surface ID_ for each surface.

  2. Build a `Surface` widget for each active surface using
     the surface IDs received in the previous step.

     For example:

     ```dart
     class _MyHomePageState extends State<MyHomePage> {
       // ...

       final _textController = TextEditingController();
       final _surfaceIds = <String>[];

       // Send a request containing the user's [text] to the agent.
       void _sendMessage(String text) async {
         if (text.trim().isEmpty) return;
         // await _conversation.sendRequest(ChatMessage.user(TextPart(text)));
       }

       // Invoked by the events stream listener when a new
       // UI surface is generated. Here, the ID is stored so the
       // build method can create a Surface to display it.
       void _onSurfaceAdded(ConversationSurfaceAdded update) {
         setState(() {
           _surfaceIds.add(update.surfaceId);
         });
       }

       // Invoked by the events stream listener when a UI surface is removed.
       void _onSurfaceDeleted(ConversationSurfaceRemoved update) {
         setState(() {
           _surfaceIds.remove(update.surfaceId);
         });
       }

       @override
       Widget build(BuildContext context) {
         return Scaffold(
           appBar: AppBar(
             backgroundColor: Theme.of(context).colorScheme.inversePrimary,
             title: Text(widget.title),
           ),
           body: Column(
             children: [
               Expanded(
                 child: ListView.builder(
                   itemCount: _surfaceIds.length,
                   itemBuilder: (context, index) {
                     // For each surface, create a Surface to display it.
                     final id = _surfaceIds[index];
                     return Surface(surfaceContext: _surfaceController.contextFor(id));
                   },
                 ),
               ),
               SafeArea(
                 child: Padding(
                   padding: const EdgeInsets.symmetric(horizontal: 16.0),
                   child: Row(
                     children: [
                       Expanded(
                         child: TextField(
                           controller: _textController,
                           decoration: const InputDecoration(
                             hintText: 'Enter a message',
                           ),
                         ),
                       ),
                       const SizedBox(width: 16),
                       ElevatedButton(
                         onPressed: () {
                           // Send the user's text to the agent.
                           _sendMessage(_textController.text);
                           _textController.clear();
                         },
                         child: const Text('Send'),
                       ),
                     ],
                   ),
                 ),
               ),
             ],
           ),
         );
       }
     }
     ```

## Add your own widgets to the catalog {:#custom-widgets}

Both `a2ui_flutter` and the legacy `genui` package include a built-in basic
catalog of Material widgets, and both allow you to register custom components.

### Defining a custom component in `a2ui_flutter`

In `a2ui_flutter`, define a `ComponentImplementation` with a JSON Schema, place
it in a `CatalogImplementation` with its own catalog ID, and register it
alongside `basicCatalog()` in your `MessageProcessor`:

```dart
final rating = ComponentImplementation(
  name: 'Rating',
  schema: Schema.object(
    properties: {
      'label': CommonSchemasV1.dynamicString,
      'value': CommonSchemasV1.dynamicNumber,
      'onChanged': Schema.fromMap({r'$ref': r'common_types.json#/$defs/Action'}),
    },
    required: ['value'],
  ),
  builder: (context, node, props, buildChild) {
    final WritableBinding<Object?>? value = props.writable('value');
    return StarRating(
      label: props.string('label') ?? '',
      value: props.number('value') ?? 0,
      onChanged: value == null
          ? null
          : (stars) {
              value.set(stars);
              props.action('onChanged')?.call();
            },
    );
  },
);

final customCatalog = CatalogImplementation(
  id: 'https://example.com/demo/catalog.json',
  protocolVersion: A2uiProtocolVersion.v1_0,
  components: [rating],
);

final processor = MessageProcessor<ComponentImplementation>(
  catalogs: [basicCatalog(), customCatalog],
);
```

### Defining a custom widget in legacy `package:genui`

If you are maintaining an existing `package:genui` app, use the following
instructions:

 1. Depend on the `json_schema_builder` package

    Use `dart pub add` to add `json_schema_builder` as
    a dependency in your `pubspec.yaml` file:
   
    ```console
    $ dart pub add json_schema_builder
    ```

 2. Create the new widget's schema

    Each catalog item needs a schema that defines the data required
    to populate it. Using the `json_schema_builder` package,
    define one for the new widget.

    ```dart
    import 'package:json_schema_builder/json_schema_builder.dart';
    import 'package:material_ui/material_ui.dart';
    import 'package:genui/genui.dart';

    final _schema = S.object(
      properties: {
        'question': S.string(description: 'The question part of a riddle.'),
        'answer': S.string(description: 'The answer part of a riddle.'),
      },
      required: ['question', 'answer'],
    );
    ```

 3. Create a `CatalogItem`

    Each `CatalogItem` represents a type of widget that the agent
    is allowed to generate. To do that, it combines a name,
    a schema, and a builder function that produces the widgets
    that compose the generated UI.

    ```dart
    final riddleCard = CatalogItem(
      name: 'RiddleCard',
      dataSchema: _schema,
      widgetBuilder:
          (itemContext) {
            final json = itemContext.data as Map<String, Object?>;
            final question = json['question'] as String;
            final answer = json['answer'] as String;

            final context = itemContext.buildContext;
    
            return Container(
              constraints: const BoxConstraints(maxWidth: 400),
              decoration: BoxDecoration(border: Border.all()),
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(question, style: Theme.of(context).textTheme.headlineMedium),
                  const SizedBox(height: 8.0),
                  Text(answer, style: Theme.of(context).textTheme.headlineSmall),
                ],
              ),
            );
          },
    );
    ```

 4. Add the `CatalogItem` to the catalog

     Include your catalog items when instantiating `SurfaceController`.

    ```dart
    _surfaceController = SurfaceController(
      catalogs: [BasicCatalogItems.asCatalog().copyWith(newItems: [riddleCard])],
    );
    ```

 5. Update the system instruction to use the new widget

    To make sure that the agent knows to use your new widget,
    tell the system instruction how and when to do so.
    Provide the name from the `CatalogItem` when you do.

    ```dart
    final promptBuilder = PromptBuilder.chat(
      catalog: catalog,
      systemPromptFragments: [
        '''
        You are an expert in creating funny riddles. Every time I give you a word,
        generate a RiddleCard that displays one new riddle related to that word.
        Each riddle should have both a question and an answer.
        '''
      ],
    );
    
    // Pass promptBuilder.systemPromptJoined() to your LLM Config
    ```

{:.steps}

## Data model and data binding

A core concept in `genui` is the `DataModel`, a centralized,
observable store for all dynamic UI state. Instead of each widget
managing its own state, its state is stored in the `DataModel`.

Widgets are _bound_ to data in this model.
When data in the model changes, only the widgets that depend
on that specific piece of data are rebuilt.
This is achieved through a `DataContext` object passed to each
widget's builder function.

### Binding to the data model

To bind a widget's property to the data model,
specify a special JSON object in the data sent from the AI.
This object can contain standard JSON primitives
(for static values) or an object with a `path` property
(to bind to a value in the data model).

For example, to display a user's name in a `Text` widget,
the AI would generate:

```json
{
  "id": "welcome-text",
  "component": "Text",
  "text": "Welcome to GenUI",
  "variant": "h1"
}
```

### Image

```json
{
  "id": "image",
  "component": "Image",
  "url": "https://example.com/image.png",
  "variant": "mediumFeature"
}
```

### Updating the data model

Input widgets, like `TextField`, update the DataModel directly.
When the user types in a text field that is bound to `/user/name`,
the `DataModel` updates, and any other widgets bound to that same
path will automatically rebuild to show the new value.

This reactive data flow simplifies state management and creates a powerful,
high-bandwidth interaction loop between the user, the UI, and the AI.

## Next steps

Check out the [examples][] included in the `genui` repo.
The [travel app][] shows how to define your own widget
catalog that the agent can use to generate domain-specific UI.

If something is unclear or missing, please [create an issue][].

[examples]: {{site.repo.organization}}/genui/blob/main/examples
[travel app]: {{site.repo.organization}}/genui/blob/main/examples/travel_app
[create an issue]: {{site.repo.organization}}/genui/issues/new/choose

## System instructions

The `genui` package gives the LLM a set of tools it can use to generate UI.
To get the LLM to use these tools,
the system instructions provided through `PromptBuilder` must
explicitly tell it to do so.

This is why the [earlier example][instruction-example] includes
a system instruction for the agent with the line
"Every time I give you a word, you should generate UI that...":

```dart highlightLines=5-6
final promptBuilder = PromptBuilder.chat(
  catalog: catalog,
  systemPromptFragments: [
    '''
    You are an expert in creating funny riddles.
    Every time I give you a word, you should generate UI that
    displays one new riddle related to that word.
    Each riddle should have both a question and an answer.
    '''
  ],
);
```

[instruction-example]: /ai/genui/get-started#create-the-connection-to-an-agent

## Troubleshooting and FAQ {:#troubleshoot}

### How can I configure logging?

To observe communication between your app and the agent,
enable logging in your `main` method.

```dart
import 'package:logging/logging.dart';
import 'package:genui/genui.dart';

final logger = configureLogging(level: Level.ALL);

void main() async {
  logger.onRecord.listen((record) {
    debugPrint('${record.loggerName}: ${record.message}');
  });

  // Additional initialization of bindings and Firebase.
}
```

### I'm getting errors about my minimum macOS/iOS version.

Choose a [deployment target supported by Flutter][supported-platforms]
that meets the minimum in the build error.
In Xcode,
[set the Runner target's minimum deployment version][deployment-target]
to that version.

[Create a new Firebase project]: https://support.google.com/appsheet/answer/10104995
[create an issue]: {{site.repo.organization}}/genui/issues/new/choose
[Enable the Gemini API]: https://firebase.google.com/docs/gemini-in-firebase/set-up-gemini
[examples]: {{site.repo.organization}}/genui/blob/main/examples
[Firebase's Flutter setup guide]: https://firebase.google.com/docs/flutter/setup
[`genui`]: {{site.pub-pkg}}/genui
[Key components]: /ai/genui/components
[deployment-target]: /packages-and-plugins/swift-package-manager/for-app-developers#how-to-use-a-swift-package-manager-flutter-plugin-that-requires-a-higher-os-version
[supported-platforms]: /reference/supported-platforms
