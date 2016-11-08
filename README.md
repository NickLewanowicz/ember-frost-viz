# ciena-frost-viz

This README outlines the details of collaborating on this Ember addon, as well as a basic usage guide.

## Background

This addon provides a visualization generator framework, which currently works something like this:

1. You provide an array to a `chart` component
2. Inside the `chart`, you define a `transform` for the data
   - The `transform` accepts `dataBindings`, which tell the framework what attributes to expect in each element in the data
   - The `bindings` accept `dimensions`, which create a mapping between the attribute value and some value to be displayed
3. Inside the `transform`, you optionally specify one or more `scales`, which explain to the viewer what values area associated with positions within the `transform`
4. Inside the `transform`, you specify one or more `plots`, which draw a visual representation of each element in the input array
5. Inside the `plot`, you specify one or more `elements`, which accept a transformed array element and draw themselves.

A fairly simple example of this is in the dummy app, at `/scatter`.

## History (TL;DR)
The viz framework originated as a visualization *helper grammar* within Ember, implemented using a single component with many properties. The properties expected generic data structure helpers like `hash` and `array`. The first iteration was interesting in that it was easy to express relationships with data, but suffered from several issues:

* Modifying the appearance of a chart was difficult, because the appearance was expressed with attributes instead of components.
* Some helpers were very heavy constructs because they needed to collect state from n-children and push updates back to parent components. The line between component and helper was blurred.

The component was rewritten in a different way: a visualization is expressed as nested components. Data bindings and formats are still expressed using helpers, but the helpers can have both scoped and global effects. With so many linked components, succinct data binding became a challenge.

## Component Pattern
With a proliferation of different nested components, a simplified API based on positional parameters was implemented:

1. Components take a positional parameter, usually an instance of a `scope`.
2. Components yield as their first parameter, an instance of the object needed for the usual child component.

## Component API and Responsibility

### Component and scope visual overview

```{puml}
array - [chart component]
[chart component] --> Scope
Scope - [transform component]
[transform component] --> TransformScope
TransformScope - [scale component]
TransformScope - [plot component]
[plot component] --> PlotElement
PlotElement - [element component]

cloud {
  AnyScope - [scope component]
  [scope component] --> AnyScope
}

```

### chart
An ember-frost-viz chart.

This component creates the root scope for the visualization. It may be used to set the size of the visualization, and create shared `dimension` objects that are available for use in bindings through all scopes.

#### Positional Parameters
| Name | Type | Description |
|------|------|-------------|
| data | array | Data to chart

#### Properties
| Name | Type | Required? | Description |
|------|------|-----------|-------------|
| width | number | no (element width) | Forced width. If absent, width is set by looking at the element after first render.
| height | number | no (element height) | Forced height. If absent, height is set by looking at the element after first render.
| dimensions | object | no (`{}`) | A hash of dimensions to include in all child scopes.

#### Yields
| Name | Type | Description |
|------|------|-------------|
| scope | object | Chart scope object


### transform
A component that defines the area in which elements are to be drawn, and provides a coordinate transform from some system to the `x` and `y` of its area.

This component defines transformation of two coordinate systems -- the input system, which is specific to the transform component, and the output system, which is always (`x` right, `y` down).

If this component has peer `scale` components, it will automatically resize its child area to make room for the scale components along its edges.

#### Positional Parameters
| Name | Type | Description |
|------|------|-------------|
| scope | object | Any viz scope

#### Properties
| Name | Type | Required? | Description |
|------|------|-----------|-------------|
| dataBindings | object | yes | Sets the bindings for input dimensions to the transform. A Cartesian transform expects at least one value for x and y, e.g. `{x: binding, y: binding}`. Multiple bindings can be provided, e.g. `{x: [bindings], ...}`
| data | array | no (`scope.data`) | Overrides the data provided to the component. Child components and bindings will see this data, not the data in the parent scope.
| area | Rectangle | no (`scope.area`) | Overrides the transform's `area` rectangle provided in the scope. The `area` provided to child components is the component's own `innerArea`, calculated based on the its `area` and any padding.
| width | number | no (`area.width`) | Overrides only the width from the scope.
| height | number | no (`area.height`) | Overrides only the height from the scope.

#### Yields
| Name | Type | Description |
|------|------|-------------|
| scope | object | Scope object including `coordinateTransforms` mapping function constructors.

### plot
A component that takes an array of input data and creates items to be displayed.

A plot takes a client area and transformation functions, and applies dimensions and bindings to each element in the scope `data` provided to generate output elements. A plot can output one element per item in the array, as in the case of a scatter plot, or it can combine elements for continuous plots, such as smoothed lines. Where a binding does not resolve a value, e.g. it is bound to an attribute missing from a data element, often the element is skipped. The effect of skipping an element depends on the plot.

#### Positional Parameters
| Name | Type | Description |
|------|------|-------------|
| scope | object | Any viz scope

#### Properties
| Name | Type | Required? (default) | Description |
|------|------|-----------|-------------|
| selectedBindings | object | no (if unambiguous: `{x: scope.dataBindings.x, y: scope.dataBindings.y}`) | If the scope has multiple bindings for one of its transform dimensions, this property selects a binding, e.g. `{x: scope.dataBindings.x[0]}`
| area | Rectangle | no (`scope.area`) | Sets the transform's `area` rectangle, overriding the `area` provided in the scope. The `area` provided to child components is the component's own `innerArea`, calculated based on the its `area` and any padding.
| width | number | no (`area.width`) | Overrides only the width from the scope.
| height | number | no (`area.height`) | Overrides only the height from the scope.
| x | number | no (`area.x`) | Overrides the left edge for the plot.
| y | number | no (`area.y`) | Overrides the top edge for the plot.
| dimensions | object | no | If specified, provides additional dimensions for elements. These aren't transformed, but can be used for additional element attributes (e.g. radius, color).
| coordinateTransforms | constructor | no (`scope.coordinateTransforms`) | Overrides the parent coordinate transforms provided. This must be a `function(area)` that returns `{ x: function(coord), y: function(coord) }`.
| dataBindings | object | no (`scope.dataBindings`) | Overrides the parent coordinate transforms provided.
| allowSparseElements | boolean | no (`false`) | Overrides whether sparse elements are rendered

Takes a `scope` with a transform. Yields `items` to be rendered.

#### Yields
| Name | Type | Description |
|------|------|-------------|
| element | object | An element item to render. The element's structure is like `{x: 0, y: 0, element: { /* original array element */ }, data: [ /* original array */] }`

### element
Takes an `item` and draws it.

The API for an element varies by element, but the attribute binding is conventional Ember. A `circle` element expects `cx`, `cy` and `r`, which can come from `element.x`, `element.y` and a constant or additional dimension.

### `scope` component

Accepts and yields a `scope`.

A scope is a powerful concept, and the existing components provide many opportunities to override properties from the parent scope. Not all properties are exposed, and peer components (e.g. multiple plotters) might want to share the same overrides. This is where a `scope` component can help.

The scope component exposes a single property, `inject`. All properties in `inject` will override any properties in the incoming scope, and will be yielded to child scopes.

#### Positional Parameters
| Name | Type | Description |
|------|------|-------------|
| scope | object | Any viz scope

#### Properties
| Name | Type | Required? (default) | Description |
|------|------|-----------|-------------|
| inject | object | no (`{}`) | Overrides the incoming scope properties with all  provided properties.

#### Yields
| Name | Type | Description |
|------|------|-------------|
| scope | object | Scope object including all `inject`ed overrides.


---


## Installation

* `git clone <repository-url>` this repository
* `cd ciena-frost-viz`
* `npm install && bower install`

## Running

* `ember serve`
* Visit your app at [http://localhost:4200](http://localhost:4200).

## Running Tests

* `npm test` (Runs `ember try:each` to test your addon against multiple Ember versions)
* `ember test`
* `ember test --server`

## Building

* `ember build`

For more information on using ember-cli, visit [http://ember-cli.com/](http://ember-cli.com/).
