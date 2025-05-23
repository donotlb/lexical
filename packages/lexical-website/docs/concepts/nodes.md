# Nodes

## Base Nodes

Nodes are a core concept in Lexical. Not only do they form the visual editor view, as part of the `EditorState`, but they also represent the
underlying data model for what is stored in the editor at any given time. Lexical has a single core based node, called `LexicalNode` that
is extended internally to create Lexical's five base nodes:

- `RootNode`
- `LineBreakNode`
- `ElementNode`
- `TextNode`
- `DecoratorNode`

Of these base nodes, three of them can be extended to create new types of nodes:

- `ElementNode`
- `TextNode`
- `DecoratorNode`

### [`RootNode`](https://github.com/facebook/lexical/blob/main/packages/lexical/src/nodes/LexicalRootNode.ts)

There is only ever a single `RootNode` in an `EditorState` and it is always at the top and it represents the
`contenteditable` itself. This means that the `RootNode` does not have a parent or siblings. It can not be
subclassed or replaced.

- To get the text content of the entire editor, you should use `rootNode.getTextContent()`.
- To avoid selection issues, Lexical forbids insertion of text nodes directly into a `RootNode`.

#### Semantics and Use Cases

Unlike other `ElementNode` subclasses, the `RootNode` has specific characteristics and restrictions to maintain editor integrity:

1. **Non-extensibility**  
   The `RootNode` cannot be subclassed or replaced with a custom implementation. It is designed as a fixed part of the editor architecture.

2. **Exclusion from Mutation Listeners**  
   The `RootNode` does not participate in mutation listeners. Instead, use a root-level or update listener to observe changes at the document level.

3. **Compatibility with Node Transforms**  
   While the `RootNode` is not "part of the document" in the traditional sense, it can still appear to be in some cases, such as during serialization or when applying node transforms. A node transform on the `RootNode` will be called at the end of *every* node transform cycle. This is useful in cases where you need something like an update listener that occurs before the editor state is reconciled.

4. **Document-Level Metadata**  
   If you are attempting to use the `RootNode` for document-level metadata (e.g., undo/redo support), use the [NodeState](/docs/concepts/node-state) API.

By design, the `RootNode` serves as a container for the editor's content rather than an active part of the document's logical structure. This approach simplifies operations like serialization and keeps the focus on content nodes.

### [`LineBreakNode`](https://github.com/facebook/lexical/blob/main/packages/lexical/src/nodes/LexicalLineBreakNode.ts)

You should never have `'\n'` in your text nodes, instead you should use the `LineBreakNode` which represents
`'\n'`, and more importantly, can work consistently between browsers and operating systems.

### [`ElementNode`](https://github.com/facebook/lexical/blob/main/packages/lexical/src/nodes/LexicalElementNode.ts)

Used as parent for other nodes, can be block level (`ParagraphNode`, `HeadingNode`) or inline (`LinkNode`).
Has various methods which define its behaviour that can be overridden during extension (`isInline`, `canBeEmpty`, `canInsertTextBefore` and more)

### [`TextNode`](https://github.com/facebook/lexical/blob/main/packages/lexical/src/nodes/LexicalTextNode.ts)

Leaf type of node that contains text. It also includes few text-specific properties:

- `format` any combination of `bold`, `italic`, `underline`, `strikethrough`, `code`, `highlight`, `subscript` and `superscript`
- `mode`
  - `token` - acts as immutable node, can't change its content and is deleted all at once
  - `segmented` - its content deleted by segments (one word at a time), it is editable although node becomes non-segmented once its content is updated
- `style` can be used to apply inline css styles to text

### [`DecoratorNode`](https://github.com/facebook/lexical/blob/main/packages/lexical/src/nodes/LexicalDecoratorNode.ts)

Wrapper node to insert arbitrary view (component) inside the editor. Decorator node rendering is framework-agnostic and
can output components from React, vanilla js or other frameworks.

## Node Properties

:::tip

If you're using Lexical v0.26.0 or later, you should consider using the [NodeState](/docs/concepts/node-state) API instead of defining properties directly on your subclasses. NodeState features automatic support for `afterCloneFrom`, `exportJSON`, and `updateFromJSON` requiring much less boilerplate and some additional benefits. You may find that you do not need a subclass at all in some situations, since your NodeState can be applied ad-hoc to any node.

:::

Lexical nodes can have properties. It's important that these properties are JSON serializable too, so you should never
be assigning a property to a node that is a function, Symbol, Map, Set, or any other object that has a different prototype
than the built-ins. `null`, `undefined`, `number`, `string`, `boolean`, `{}` and `[]` are all types of property that can be
assigned to node.

By convention, we prefix properties with `__` (double underscore) so that it makes it clear that these properties are private
and their access should be avoided directly. We opted for `__` instead of `_` because of the fact that some build tooling
mangles and minifies single `_` prefixed properties to improve code size. However, this breaks down if you're exposing a node
to be extended outside of your build.

If you are adding a property that you expect to be modifiable or accessible, then you should always create a set of `get*()`
and `set*()` methods on your node for this property. Inside these methods, you'll need to invoke some very important methods
that ensure consistency with Lexical's internal immutable system. These methods are `getWritable()` and `getLatest()`.

We recommend that your constructor should always support a zero-argument instantiation in order to better support collab and
to reduce the amount of boilerplate required. You can always define your `$create*` functions with required arguments.

```js
import type {NodeKey} from 'lexical';

class MyCustomNode extends SomeOtherNode {
  __foo: string;

  constructor(foo: string = '', key?: NodeKey) {
    super(key);
    this.__foo = foo;
  }

  setFoo(foo: string): this {
    // getWritable() creates a clone of the node
    // if needed, to ensure we don't try and mutate
    // a stale version of this node.
    const self = this.getWritable();
    self.__foo = foo;
    return self;
  }

  getFoo(): string {
    // getLatest() ensures we are getting the most
    // up-to-date value from the EditorState.
    const self = this.getLatest();
    return self.__foo;
  }
}
```

Lastly, all nodes should have `static getType()`, `static clone()`, and `static importJSON()` methods.
Lexical uses the type to be able to reconstruct a node back with its associated class prototype
during deserialization (important for copy + paste!). Lexical uses cloning to ensure consistency
between creation of new `EditorState` snapshots.

Expanding on the example above with these methods:

```js
interface SerializedCustomNode extends SerializedLexicalNode {
  foo?: string;
}

class MyCustomNode extends SomeOtherNode {
  __foo: string;

  static getType(): string {
    return 'custom-node';
  }

  static clone(node: MyCustomNode): MyCustomNode {
    // If any state needs to be set after construction, it should be
    // done by overriding the `afterCloneFrom` instance method.
    return new MyCustomNode(node.__foo, node.__key);
  }

  static importJSON(
    serializedNode: LexicalUpdateJSON<SerializedMyCustomNode>
  ): MyCustomNode {
    return new MyCustomNode().updateFromJSON(serializedNode);
  }

  constructor(foo: string = '', key?: NodeKey) {
    super(key);
    this.__foo = foo;
  }

  updateFromJSON(
    serializedNode: LexicalUpdateJSON<SerializedMyCustomNode>
  ): this {
    const self = super.updateFromJSON(serializedNode);
    return typeof serializedNode.foo === 'string'
      ? self.setFoo(serializedNode.foo)
      : self;
  }

  exportJSON(): SerializedMyCustomNode {
    const serializedNode: SerializedMyCustomNode = super.exportJSON();
    const foo = this.getFoo();
    if (foo !== '') {
      serializedNode.foo = foo;
    }
    return serializedNode;
  }

  setFoo(foo: string): this {
    // getWritable() creates a clone of the node
    // if needed, to ensure we don't try and mutate
    // a stale version of this node.
    const self = this.getWritable();
    self.__foo = foo;
    return self;
  }

  getFoo(): string {
    // getLatest() ensures we are getting the most
    // up-to-date value from the EditorState.
    const self = this.getLatest();
    return self.__foo;
  }
}
```

## Creating custom nodes

As mentioned above, Lexical exposes three base nodes that can be extended.

> Did you know? Nodes such as `ElementNode` are already extended in the core by Lexical, such as `ParagraphNode` and `RootNode`!

### Extending `ElementNode`

Below is an example of how you might extend `ElementNode`:

```js
import {ElementNode, LexicalNode} from 'lexical';

export class CustomParagraph extends ElementNode {
  static getType(): string {
    return 'custom-paragraph';
  }

  static clone(node: CustomParagraph): CustomParagraph {
    return new CustomParagraph(node.__key);
  }

  createDOM(): HTMLElement {
    // Define the DOM element here
    const dom = document.createElement('p');
    return dom;
  }

  updateDOM(prevNode: this, dom: HTMLElement, config: EditorConfig): boolean {
    // Returning false tells Lexical that this node does not need its
    // DOM element replacing with a new copy from createDOM.
    return false;
  }
}
```

It's also good etiquette to provide some `$` prefixed utility functions for
your custom `ElementNode` so that others can easily consume and validate nodes
are that of your custom node. Here's how you might do this for the above example:

```js
export function $createCustomParagraphNode(): CustomParagraph {
  return $applyNodeReplacement(new CustomParagraph());
}

export function $isCustomParagraphNode(
  node: LexicalNode | null | undefined
): node is CustomParagraph  {
  return node instanceof CustomParagraph;
}
```

### Extending `TextNode`

```js
export class ColoredNode extends TextNode {
  __color: string;

  constructor(text: string, color: string, key?: NodeKey): void {
    super(text, key);
    this.__color = color;
  }

  static getType(): string {
    return 'colored';
  }

  static clone(node: ColoredNode): ColoredNode {
    return new ColoredNode(node.__text, node.__color, node.__key);
  }

  createDOM(config: EditorConfig): HTMLElement {
    const element = super.createDOM(config);
    element.style.color = this.__color;
    return element;
  }

  updateDOM(
    prevNode: this,
    dom: HTMLElement,
    config: EditorConfig,
  ): boolean {
    const isUpdated = super.updateDOM(prevNode, dom, config);
    if (prevNode.__color !== this.__color) {
      dom.style.color = this.__color;
    }
    return isUpdated;
  }
}

export function $createColoredNode(text: string, color: string): ColoredNode {
  return $applyNodeReplacement(new ColoredNode(text, color));
}

export function $isColoredNode(
  node: LexicalNode | null | undefined
): node is ColoredNode {
  return node instanceof ColoredNode;
}
```

### Extending `DecoratorNode`

```ts
export class VideoNode extends DecoratorNode<ReactNode> {
  __id: string;

  static getType(): string {
    return 'video';
  }

  static clone(node: VideoNode): VideoNode {
    return new VideoNode(node.__id, node.__key);
  }

  constructor(id: string, key?: NodeKey) {
    super(key);
    this.__id = id;
  }

  createDOM(): HTMLElement {
    return document.createElement('div');
  }

  updateDOM(): false {
    return false;
  }

  decorate(): ReactNode {
    return <VideoPlayer videoID={this.__id} />;
  }
}

export function $createVideoNode(id: string): VideoNode {
  return $applyNodeReplacement(new VideoNode(id));
}

export function $isVideoNode(
  node: LexicalNode | null | undefined,
): node is VideoNode {
  return node instanceof VideoNode;
}
```

Using `useDecorators`, `PlainTextPlugin` and `RichTextPlugin` executes `React.createPortal(reactDecorator, element)` for each `DecoratorNode`,
where the `reactDecorator` is what is returned by `DecoratorNode.prototype.decorate`,
and the `element` is an `HTMLElement` returned by `DecoratorNode.prototype.createDOM`.
