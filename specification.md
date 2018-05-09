# Graphical Server Protocol

This document describes the first version of the graphical server protocol. An implementation of the complete protocol described here can be found in the web-based prototype of [Eclipse Sirius](https://www.eclipse.org/sirius/) realized by [Obeo](https://www.obeo.fr). This prototype has been created on top of [Sprotty](https://github.com/theia-ide/sprotty) which has defined and implemented the core parts of this protocol.

## Base Protocol

The following TypeScript definitions describe the base protocol:

### ActionMessage
A general message as defined by JSON. It holds the action that are transmited between the client and the server.

```typescript
class ActionMessage {
  /**
   * Used to identify a specific client.
   */
  public readonly clientId: string;

  /**
   * The action to execute.
   */
  public readonly action: Action;
}
```

Action messages processed by the client or the server do not automatically require a response. Additional information regarding the lifecycle of some action messages can be found in the [Sprotty documentation](https://github.com/theia-ide/sprotty/wiki/Client-Server-Protocol).

## Actions

Actions contained in the action messages are identified by their `kind` attribute. This attribute is required for all actions. Some actions may most of the time be supported only by a client or only by the server but they can be supported by both clients and servers in theory depending on the situation. All actions must extend the default action interface.

```typescript
interface Action {
  /**
   * Unique identifier specifying the kind of action to process.
   */
  readonly kind: string;
}
```

### CenterAction

Centers the viewport on the elements with the given identifiers. It changes the scroll setting of the viewport accordingly and resets the zoom to its default. This action can also be created on the client but it can also be sent by the server in order to perform such a viewport change remotely.

```typescript
class CenterAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'center';

  /**
   * The identifier of the elements on which the viewport should be centered.
   */
  public readonly elementIds: string[];

  /**
   * Indicate if the modification of the viewport should be realized with or without support of animations.
   */
  public readonly animate: boolean = true;
}
```

### CollapseExpandAction

Recalculates a diagram when some specific elements are collapsed or expanded. This action can be sent by a client to the server to let the server compute a new version of the diagram.

```typescript
class CollapseExpandAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'collapseExpand';
  
  /**
   * The identifier of the elements to expand.
   */
  public readonly expandIds: string[];

  /**
   * The identifier of the elements to collapse.
   */
  public readonly collapseIds: string[];

  /**
   * Indicates if the elements should be expanded (true) or collapsed (false)
   */
  public readonly expand: boolean = true;
}
```

### CollapseExpandAllAction

Collapses or expands all elements of the diagram.

```typescript
export class CollapseExpandAllAction {
  /**
   * The kind of the action.
   */
  public readonly kind = 'collapseExpandAll';
  
  /**
   * Indicates if the elements should be expanded (true) or collapsed (false)
   */
  public readonly expand: boolean = true;
}
```

### ComputedBoundsAction

Sent from the client to the server to transmit the result of bounds computation as a response to a `RequestBoundsAction`. If the server is responsible for parts of the layout (see `needsServerLayout` viewer option), it can do so after applying the computed bounds received with this action. Otherwise there is no need to send the computed bounds to the server, so they can be processed locally by the client.

```typescript
export class ComputedBoundsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'computedBounds';

  /**
   * The new bounds of the diagram elements.
   */
  public readonly bounds: ElementAndBounds[];

  /**
   * The new alignment of the diagram elements.
   */
  public readonly alignments?: ElementAndAlignment[];
}
```

The `ElementAndBounds` type is used to associate new bounds with a model element, which is referenced via its id.

```typescript
class ElementAndBounds {
  /**
   * The identifier of the element.
   */
  public readonly elementId: string;

  /**
   * The new bounds of the element.
   */
  public readonly newBounds: Bounds;
}
```

The bounds are the position (x, y) and dimension (width, height) of an object. As such the `Bounds` type extends both `Point` and `Dimension`.

```typescript
class Bounds extends Point, Dimension {
}
```

A `Point` is composed of the (x,y) coordinates of an object.

```typescript
class Point {
  /**
   * The abscissa of the point.
   */
  public readonly x: number;

  /**
   * The ordinate of the point.
   */
  public readonly y: number;
}
```

The `Dimension` of an object is composed of its width and height.

```typescript
class Dimension {
  /**
   * The width of an element.
   */
  public readonly width: number;

  /**
   * the height of an element.
   */
  public readonly height: number;
}
```

The `ElementAndAlignment` type is used to associate a new alignment with a model element, which is referenced via its id.

```typescript
class ElementAndAlignment {
  /**
   * The identifier of an element.
   */
  public readonly elementId: string;

  /**
   * The new alignment of the element.
   */
  public readonly newAlignment: Point;
}
```

### ExecuteNodeCreationToolAction

Triggered when the user requests the execution of a node creation tool.

```typescript
class ExecuteNodeCreationToolAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'executeNodeCreationTool';

  /**
   * The id of the node creation tool to execute.
   */
  public readonly toolId: string;

  /**
   * The location where the tool has been invoked by the user.
   */
  public readonly location?: Point;

  /**
   * The id of the container element in which the tool has been invoked by the user.
   */
  public readonly containerId?: string;
}
```

### ExecuteToolAction

Triggered when the user requests the execution of a generic tool.

```typescript
class ExecuteToolAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'executeTool';

  /**
   * The id of the generic tool to execute.
   */
  public readonly toolId: string;
  
  /**
   * The location where the tool has been invoked by the user.
   */
  public readonly location?: Point;

  /**
   * The id of the element on which the tool has been invoked by the user.
   */
  public readonly elementId?: string;
}
```

### RequestBoundsChangeHintsAction

Sent from the client to the server in order to request hints on whether which elements may be resized and repositioned. The `RequestBoundsChangeHintsAction` is optional, but should usually be among the first messages sent from the client to the server after receiving the model via `RequestModelAction`. The response is a `SetBoundsChangeHintsAction`. The rationale is to avoid a client-server round-trip for user feedback of each synchronous user interaction, such as drag and drop.

```typescript
class RequestBoundsChangeHintsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestBoundsChangeHints';
}
```

### SetBoundsChangeHintsAction

Sent from the server to the client in order to provide hints on which element may be resized or repositioned. These hints concern only the elements' graphical size and position (see also `ChangeBoundsAction`).

```typescript
class SetBoundsChangeHintsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setBoundsChangeHints';

  /**
   * The hints.
   */
  public readonly hints: BoundsChangeHint[];
}

class BoundsChangeHint {
  /**
   * The id of the element.
   */
  public readonly elementId: string;

  /**
   * Specifies whether the element can be resized.
   */
  public readonly isResizable: boolean;

  /**
   * Specifies whether the element can be relocated.
   */
  public readonly isRepositionable: boolean;
}
```

### ChangeBoundsAction

Triggered when the user changes the position or size of an element. This action concerns only the element's graphical size and position. Whether an element can be resized or repositioned may be specified by the server with a `SetChangeBoundsHintsAction` to allow for immediate user feedback before resizing or repositioning.

```typescript
class ChangeBoundsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'changeBounds';

  /**
   * The new bounds of an element.
   */
  public readonly newBounds: ElementAndBounds[];
}
```

### RequestMoveHintsAction

Sent from the client to the server in order to request hints on whether which elements may be moved into which containers. The `RequestMoveHintsAction` is optional, but should usually be among the first messages sent from the client to the server after receiving the model via `RequestModelAction`. The response is a `SetMoveHintsAction`.

```typescript
class RequestMoveHintsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestMoveHints';
}
```

### SetMoveHintsAction

Sent from the server to the client in order to provide hints on which element may be moved into which target element. These hints specify whether an element can change its container (see also `MoveAction`). The rationale is to avoid a client-server round-trip for user feedback of each synchronous user interaction, such as drag and drop.

```typescript
class SetMoveHintsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setMoveHints';

  /**
   * The move hints.
   */
  public readonly hints: DragAndDropHints[];
}

/*
 * TODO Up for discussion:
 * An enumerative id-based drag and drop hint would be the most
 * explicit and clear hint, but it may introduce  scalability issues
 * as we may deal with a filtered  cross-product of elements that needs
 * to be computed and sent to the client.
 * Probably best would be a class-based drag and drop hint
 * (e.g. nodes with class `a` may be moved into nodes  with class `b`).
 */

class DragAndDropHint {
  /**
   * The class of the element being a drag source.
   */
  public readonly dragElementClass: string;

  /**
   * The classes of elements being a drop target for elements with the class dragElementClass.
   */
  public readonly dropElementClasses: string[];
}
```

### MoveAction

Triggered when the user moves an element to a new container. Whether an element can be moved into another container may be specified by the server with a `SetMoveHintsAction` to allow for immediate user feedback before moving.

```typescript
class MoveAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'move';

  /**
   * The id of the element being the drag source.
   */
  public readonly movedElementId: string;

  /**
   * The id of the element being the drop target.
   */
  public readonly targetContainerId: string;
}
```

### Connection Tools

TODO up for discussion:

* `RequestConnectionTools`
* `SetConnectionTools`
* `ExecuteConnectionTool`
* SetConnectionHints: Similar to the `SetMoveHints`, we should introduce connection hints for the client, ideally reusing the `DragAndDropHint`. The rationale is to avoid a client-server round-trip for user feedback of each synchronous user interaction, such as drag and drop.
* `ExecuteReconnectAction`


### ExportSvgAction

Used to export the diagram as an SVG image.

```typescript
class ExportSvgAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'exportSvg';

  /**
   * The SVG image data.
   */
  public readonly svg: string;
}
```

### FitToScreenAction

Triggered when the user requests the viewer to fit its content to the available drawing area. The resulting FitToScreenCommand changes the zoom and scroll settings of the viewport so the model can be shown completely. This action can also be sent from the model source to the client in order to perform such a viewport change programmatically.

```typescript
class FitToScreenAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'fit';

  /**
   * The identifier of the elements to fit on screen.
   */
  public readonly elementIds: string[];

  /**
   * The padding that should be visible on the viewport.
   */
  public readonly padding?: number;

  /**
   * The max zoom level authorized.
   */
  public readonly maxZoom?: number;

  /**
   * Indicate if the action should be performed with animation support or not.
   */
  public readonly animate: boolean = true;
}
```

### OpenAction

Used to indicate that an element has been opened. The default behavior will be triggered when an end user double click on an element. It can allow a server to react to this event.

```typescript
class OpenAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'open';

  /**
   * The identifier of the element.
   */
  public readonly elementId: string;
}
```

### RequestBoundsAction

Sent from the server to the client to request bounds for the given model. The model is rendered invisibly so the bounds can derived from the DOM. The response is a `ComputedBoundsAction`. This hidden rendering round-trip is necessary if the client is responsible for parts of the layout.

```typescript
class RequestBoundsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestBounds';

  /**
   * The diagram elements to consider to compute the new bounds.
   */
  public readonly newRoot: SModelRootSchema;
}
```

### RequestExportSvgAction

Used to request the export of the diagram as an SVG image.

```typescript
class RequestExportSvgAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestExportSvg';
}
```

### RequestLayersAction

Sent from the client to the server in order to request diagram layers. With `RequestModelAction` and `RequestToolsAction`, they are the firsts messages that are sent to the server. The response is a `SetLayersAction`.

```typescript
class RequestLayersAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestLayers';
}
```

### RequestModelAction

Sent from the client to the server in order to request a model. Usually this is the first message that is sent from the client to the server, so it is also used to initiate the communication. The response is a `SetModelAction` or an `UpdateModelAction`.

```typescript
class RequestModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestModel';

  /**
   * Additional options used to compute the diagram.
   */
  public readonly options?: { [key: string]: string });
}
```

### RequestPopupModelAction

Triggered when the user hovers the mouse pointer over an element to get a popup with details on that element. This action is sent from the client to the server. The response is a `SetPopupModelAction`.

```typescript
class RequestPopupModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestPopupModel';

  /**
   * The identifiers of the elements for which a popup is requested.
   */
  public readonly elementIds: string[];

  /**
   * The bounds.
   */
  public readonly bounds: Bounds;

  // TODO indicate hover vs. right-click? We probably don't want
  // to bound these to the concrete user interaction (right-click vs hover)
  // and leave that to the client implementation, however, we may want to
  // distingiush between two types of popups.
}
```

### RequestToolsAction

Sent from the client to the server in order to request diagram tools. With `RequestModelAction` and `RequestLayersAction`, they are the firsts messages that is sent to the server. The response is a `SetToolsAction`.

```typescript
class RequestToolsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestTools';
}
```

### SelectAction

Triggered when the user changes the selection, e.g. by clicking on a selectable element. The action should trigger a change in the `selected` state accordingly, so the elements can be rendered differently. This action is also forwarded to the server, if present, so it may react on the selection change. Furthermore, the server can send such an action to the client in order to change the selection remotely.

```typescript
class SelectAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'elementSelected';

  /**
   * The identifier of the elements to mark as selected.
   */
  public readonly selectedElementsIDs: string[] = [];

  /**
   * The identifier of the elements to mark as not selected.
   */
  public readonly deselectedElementsIDs: string[] = [];
}
```

### SelectAllAction

Used for selecting or deselecting all elements.

```typescript
class SelectAllAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'allSelected';

  /**
   * If `select` is true, all elements are selected, othewise they are deselected.
   */
  public readonly select: boolean = true;
}
```

### ServerStatusAction

Sent by the server to signal a state change.

```typescript
class ServerStatusAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'serverStatus';

  /**
   * The severity of the status.
   */
  public readonly severity: string;

  /**
   * The message describing the status.
   */
  public readonly message: string;

  /**
   * Element ids this status concerns.
   */
  public readonly elementIds?: string[];
}
```

### SetBoundsAction

Sent from the server to the client to update the bounds of some (or all) model elements.

```typescript
class SetBoundsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setBounds';

  /**
   * The elements to update with their new bounds.
   */
  public readonly bounds: ElementAndBounds[];
}
```

### SetLayersAction

Sent from the server to the client in order to set the diagram layers. If layers are already presents, they are replaced.

```typescript
class SetLayersAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setLayers';

  /**
   * The layers of the diagram.
   */
  public readonly layers: Layer[];
}
```

```typescript
interface Layer {
  /**
   * The identifier.
   */
  readonly id: string;

  /**
   * The name.
   */
  readonly name: string;

  /**
   * Indicates if the layer is currently active or not.
   */
  readonly isActive: boolean;
}
```

### SetModelAction

Sent from the model source to the client in order to set the model. If a model is already present, it is replaced.

```typescript
class SetModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setModel';

  /**
   * The new diagram elements.
   */
  public readonly newRoot: SModelRootSchema; // imported from sprotty
}
```

### SetPopupModelAction

Sent from the model source to the client to display a popup in response to a `RequestPopupModelAction`. This action can also be used to remove any existing popup by choosing EMPTY_ROOT as root element.

```typescript
class SetPopupModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setPopupModel';

  /**
   * The model elements composing the popup to display.
   */
  public readonly newRoot: SModelRootSchema; // imported from sprotty
}
```

### SetToolsAction

Sent from the server to the client in order to set the available tools. A tool is available only if the layer that contains the tool is activated. If tools are already presents, they are replaced.

```typescript
class SetToolsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setTools';

  /**
   * The tools of the diagram.
   */
  public readonly tools: Tool[];
}

interface Tool {
  /**
   * The tool identifier.
   */
  readonly id: string;

  /**
   * The tool name.
   */
  readonly name: string;

  /**
   * The tool type.
   */
  readonly toolType: string;
  // TODO what is a toolType?
  // Is it e.g. "palette"? Should probably be an enum to be more clear.
}
```

### ToggleLayerAction

Sent from the client to the server in order to toggle a layer. The model, the layers and the tools may be updated after the changes.

```typescript
class ToggleLayerAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind: string = 'toggleLayer';

  /**
   * The name of the layer.
   */
  public readonly layerName: string;

  /**
   * The new state of the layer.
   */
  public readonly newState: boolean;
}
```

### UpdateModelAction

Sent from the server to the client in order to update the model. If no model is present yet, this behaves the same as a `SetModelAction`. The transition from the old model to the new one can be animated.

```typescript
class UpdateModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'updateModel';

  /**
   * The new root element of the diagram.
   */
  public readonly newRoot?: SModelRootSchema;

  /**
   * The matches used while comparing the diagram elements.
   */
  public readonly matches?: Match[]; // imported from sprotty

  /**
   * Indicates if the update should be performed with animation support or not.
   */
  public readonly animate: boolean = true;
}
```


