---
title: "Client APIs Available to Pluggable Widgets"
parent: "pluggable-parent-9"
menu_order: 10
description: A guide for understanding the client APIs available to pluggable widgets.
tags: ["Widget", "Pluggable",  "JavaScript"]
---

## 1 Introduction

The main API the Mendix platform provides to a pluggable widget client component is the props the component receives. These props resemble the structure of properties specified in the widget definition XML file (a structure described in [Pluggable Widgets API](pluggable-widgets)). A property's attribute type affects how the property will be represented to the client component. Simply, an attribute's type defines what will it be. You can find the more details on property types and the interfaces that property value can adhere to in [Pluggable Widget Property Types](property-types-pluggable-widgets). To see examples of pluggable widgets in action, see [How To Build Pluggable Widgets](/howto/extensibility/pluggable-widgets)

The Mendix platform also exposes a few JavaScript modules, specifically extra Mendix APIs as well as existing libraries, like React, that client components must share with the platform to function properly. For more information on exposed libraries, see the [Exposed Libraries](#exposed-libraries) section below.

## 2 Bundling

Mendix does not provide you code as an *npm package*, which is the approach commonly used by JavaScript libraries. Instead, Mendix provides you modules available during execution. Hence, if you are using a module bundler like [webpack](https://webpack.js.org/), you should configure it to mark these modules as [externals](https://webpack.js.org/configuration/externals/).

This process can be cumbersome, so it is recommended you use this [tools package](https://www.npmjs.com/package/@mendix/pluggable-widgets-tools) which contains the correctly-configured bundlers to work with pluggable widgets. If you follow best practices and use the [Mendix Pluggable Widget Generator](https://www.npmjs.com/package/@mendix/generator-widget) to scaffold your widget, then this package is added automatically.

## 3 Standard Properties

Alongside the props that correspond to the properties specified in widget definition XML file, the props listed below are always passed to a client component.

### 3.1 Name 

In Mendix Studio and Mendix Studio Pro, every widget must have a name configured. The primary usage of a widget name is to make its component identifiable in the client so that it can be targeted using [Selenium](/howto/integration/selenium-support) or Appium test automation. In web apps, the Mendix platform automatically adds the class `mx-name-{widgetName}` to a widget so that no extra action from a component developer is required. Unfortunately, this solution is not possible for [native mobile apps](/howto/mobile/native-mobile). For native mobile apps a component developer must manually pass a given `string` `name` prop to an underlying React Native [testID](https://facebook.github.io/react-native/docs/view#testid).

### 3.2 Class

A user can specify multiple classes for every widget. They can do this either directly by configuring a [class](/refguide/common-widget-properties#class) property in the Studios, or by using design properties. In web apps, the Mendix platform creates a CSS class string from the configuration and passes it as a `string` `class` prop to every client component. Unfortunately, React Native does not have similar support for classes. Therefore in native mobile apps a component will not receive `class` prop, but a `style` prop instead.

### 3.3 Style

A user can specify a custom CSS for every widget on a web page by using the [style](/refguide/common-widget-properties#style) property. This styling is passed to a client component through an optional `style` prop of the type `CSSProperties`.

On native pages, the meaning of a `style` prop is very different. First of all, a user cannot specify the aforementioned inline styles for widgets on a native page. So a `style` prop is used to pass styles computed based on configured classes. A client component will receive an array with a single [style object](/refguide/native-styling-refguide#style-objects) with all applicable styles combined.

### 3.4 TabIndex

If a widget uses a TabIndex prop [system property](property-types-pluggable-widgets#tabindex), then it will receive a configured `Tab index` through a `number` `tabIndex` property, except in the case when a configured tab index is on its default value of 0. Currently, `tabIndex` is not passed to widgets used on native pages. 

## 4 Property Values

### 4.1 ActionValue {#actionvalue}

ActionValue is used to represent actions, like the [On click](/refguide/on-click-event#on-click) property of an action button. For any action except **Do nothing**, your component will receive a value adhering to the following interface. For **Do nothing** it will receive `undefined`. The `ActionValue` prop appears like this:

```ts
export interface ActionValue {
    readonly canExecute: boolean;
    readonly isExecuting: boolean;
    execute(): void;
}
```

The flag `canExecute` indicates if an action can be executed under the current conditions. This helps you prevent executing actions that are not allowed by the app's security settings. User roles can be set in the microflows and nanoflows, allowing users to call them. For more information on user roles and security, see the [Module Security Reference Guide](/refguide/module-security). You can also employ this flag when using a **Call microflow** action triggering a microflow with a parameter. Such an action cannot be executed until a parameter object is available, for example when a parent Data view has finished loading. An attempt to `execute` an action that cannot be executed will have no effect except generating a debug-level warning message. 

The flag `isExecuting` indicates whether an action is currently running. A long-running action can take seconds to complete. Your component might use this information to render an inline loading indicator which lets users track loading progress. Often it is not desirable to allow a user to trigger multiple actions in parallel. Therefore, a component (maybe based on a configuration) can decide to skip triggering an action while a previous execution is still in progress.

Note that `isExecuting` indicates only whether the current action is running. It does not indicate whether a target nanoflow, microflow, or object operation is running due to another action.

The method `execute` triggers the action. It returns nothing and does not guarantee that the action will be started synchronously. But when the action does start, the component will receive a new prop with the `isExecuting` flag set.

### 4.2 DynamicValue {#dynamic-value}

DynamicValue is used to represent values that can change over time and is used by many property types. It is defined as follows:

```ts
export type DynamicValue<X> =
    | { readonly status: ValueStatus.Available; readonly value: X }
    | { readonly status: ValueStatus.Unavailable; readonly value: undefined }
    | { readonly status: ValueStatus.Loading; readonly value: X | undefined };
    
export const enum ValueStatus {
    Loading = "loading",
    Unavailable = "unavailable",
    Available = "available"
}
```

A component will receive a `DynamicValue<X>`  where type `X` depends on a property configuration. For example, for the [TextTemplate property](property-types-pluggable-widgets#texttemplate) it will be `DynamicValue<string>`, but for the [expression property](property-types-pluggable-widgets#expression) `X` will depend on a configured `returnType`.

Though the type definition above looks complex, it is fairly simply to use because a component can always read `DynamicValue.value`. This field either contains an actual value, such as an interpolated `string` in the case of a Text template, or the last known correct value if the value is being recomputed, such as when a parent Data view reloads its Data source. In other cases the value is set as `undefined`.

`DynamicValue.status` provides a component with additional information about the state of a dynamic value, as well as if the component should handle them differently. This is done using a [discriminated union](https://www.typescriptlang.org/docs/handbook/advanced-types.html#discriminated-unions) that covers the following situations:

* When `status` is `ValueStatus.Available`, then the dynamic value has sufficient information to be computed, and the result is exposed in `value`.
* When `status` is `ValueStatus.Unavailable`, then the dynamic value does not have such information such as when a parent Data view’s Data source has returned nothing. The `value` is then always `undefined`.
* When `status` is `ValueStatus.Loading`, then the dynamic value is awaiting for the required information to arrive. This happens when a parent Data view is either waiting for its object to load or is reloading it due to a [refresh in client](/refguide/change-object#refresh-in-client).
	* In case a dynamic value was previously in a `ValueStatus.Available` state, then the previous `value` is still returned. This is done so that a component can keep showing the previous value if it doesn’t need to handle `Loading` explicitly. This prevents flickering: a state when a displayed value rapidly changes between loading and not loading several times.
	* In other cases, the `value` is `undefined`. This is a common situation while a page is still being loaded.

### 4.3 EditableValue {#editable-value}

EditableValue is used to represent values that can be changed by a pluggable widget client component and is passed only to [attribute properties](property-types-pluggable-widgets#attribute). It is defined as follows:

```ts
export interface EditableValue<T extends AttributeValue> {
    readonly status: ValueStatus;
    readonly readOnly: boolean;
    
    readonly value: T | undefined;
    setValue(value: T | undefined): void;
    readonly validation: string | undefined;
    setValidator(validator?: (value: T | undefined) => string | undefined): void;
    
    readonly displayValue: string;
    setTextValue(value: string): void;
    
    readonly formatter: ValueFormatter<T>;
    setFormatter(formatter: ValueFormatter<T> | undefined): void;
    
    readonly universe?: T[];
}
```

A component will receive `EditableValue<X>` where `X` depends on the configured `attributeType`.

`status` is similar to one exposed for `DynamicValue`. It indicates if the value's loading has finished and if loading was successful. Similarly to `DynamicValue`, `EditableValue` keeps returning the previous `value` when `status` changes from `Available` to `Loading` to help a widget avoid flickering.

The flag `readOnly` indicates whether a value can actually be edited. It will be true, for example, when a widget is placed inside a Data view that is not [editable](/refguide/data-view#editable), or when a selected attribute is not editable due to [access rules](/refguide/access-rules). The `readOnly` flag is always true when a `status` is not `ValueStatus.Available`. Any attempt to edit a value set to read-only will have no affect and incur a debug-level warning message.

The value can be read from the `value` field and modified using `setValue` function. Note that `setValue` returns nothing and does not guarantee that the value is changed synchronously. But when a change is propagated, a component receives a new prop reflecting the change.

When setting a value, a new value might not satisfy certain validation rules — for example a value might be bigger that the underlying attribute allows. In this case, your change will affect only `value` and `displayValue` received through a prop. Your change will not be propagated to an object’s attribute and will not be visible outside of your component. The component will also receive a validation error text through the `validation` field of `EditableValue`.

It is possible for a component to extend the defined set of validation rules. A new validator — a function that checks a passed value and returns a validation message string if any — can be provided through the `setValidator` function. A component can have only a single custom validator. The Mendix platform ensures that custom validators are executed whenever necessary, for example when a page is being saved by an end-user. It is best practice to call `setValidator` early in a component's lifecycle — specifically in the [componentDidMount](https://en.reactjs.org/docs/react-component.html#componentdidmount) function.

In practice, many client components present values as nicely formatted strings which take locale-specific settings into account. To facilitate such cases `EditableValue` exposes a field `displayValue` formatted version of `value`, and a method `setTextValue` — a version of `setValue` that takes care of parsing. `setTextValue` also validates that a passed value can be parsed and assigns the target attribute’s type. Similarly to `setValue`, a change to an invalid value will not be propagated further that the prop itself, but a `validation` is reported. Note that if a value cannot be parsed, the prop will contain only a `displayValue` string and `value` will become undefined.

There is a way to use more the convenient `displayValue`  and `setTextValue` while retaining control over the format. A component can use a `setFormatter` method passing a formatter object: an object with `format` and `parse` methods. The Mendix platform provides a convenient way of creating such objects for simple cases. An existing formatter exposed using a `EditableValue.formatter` field can be modified using its `withConfig` method. For complex cases formatters still can be created manually. A formatter can be reset back to default settings by calling `setFormatter(undefined)`.

The optional field `universe` is used to indicate the set of all possible values that can be passed to a `setValue` if a set is limited. Currently, `universe` is provided only when the edited attribute is of the Boolean or enumeration [types](/refguide/attributes#type).

### 4.4 IconValue {#icon-value}

`DynamicValue<IconValue>` is used to represent icons: small pictograms in the Mendix platform. Those can be static or dynamic file- or font-based images. An icon can only be configured through an [icon](property-types-pluggable-widgets#attribute) property. `IconValue` is defined as follows:

```ts
interface GlyphIcon {
    readonly type: "glyph";
    readonly iconClass: string;
}
    
interface WebImageIcon {
    readonly type: "image";
    readonly iconUrl: string;
}
    
interface NativeImageIcon {
    readonly type: "image";
    readonly iconUrl: Readonly<ImageURISource>;
}
    
export type WebIcon = GlyphIcon | WebImageIcon | undefined;
export type NativeIcon = GlyphIcon | NativeImageIcon | undefined;
export type IconValue = WebIcon | NativeIcon;
```

In practice, `WebIcon` and `NativeIcon` are usually passed to a `Icon` component provided by Mendix, since this provides a convenient way of handling all types of icons at once. For more information on `Icon`, see the [Icon](#icon) section below.

### 4.5 ImageValue{#imagevalue}

`DynamicValue<ImageValue>` is used to represent static or dynamic images. An image can be configured only through an [image](property-types-pluggable-widgets#image) property. `ImageValue` is defined as follows:

```ts
export interface WebImage {
    readonly uri: string;
    readonly name: string;
    readonly altText?: string;
}
export type NativeImage = Readonly<ImageURISource & { name?: string; } | string | number>;
export type ImageValue = WebImage | NativeImage;
```

`NativeImage` can be passed to a `mendix/components/native/Image` component provided by Mendix for native widgets. `WebImage` can be passed to react-dom’s `img` component.

### 4.6 FileValue {#filevalue}

`DynamicValue<FileValue>` is used to represent files. A file can be configured only through a [file](property-types-pluggable-widgets#file) property. `FileValue` is defined as follows:

```ts
export interface FileValue {
    readonly uri: string;
    readonly name: string;
}
```

### 4.7 ListValue{#listvalue}

`ListValue` is used to represent a list of objects for the [datasource](property-types-pluggable-widgets#datasource) property.

```ts
export interface ObjectItem {
    id: GUID;
}

export interface ListValue {
    status: ValueStatus;
    offset: number;
    limit: number;
    setOffset(offset: number): void;
    setLimit(limit: Option<number>): void;
    requestTotalCount(needTotalCount: boolean): void;
    items?: ObjectItem[];
    hasMoreItems?: boolean;
    totalCount?: number;
}
```

When a `datasource` property with `isList="true"` is configured for a widget, the client component gets a list of objects represented as a `ListValue`. This type allows detailed access and control over the data source.

The `offset` and `limit` properties specify the range of objects retrieved from the datasource. The `offset` is the starting index and the `limit` is the number of requested items. By default, the `offset` is *0* and the `limit` is `undefined` which means all the datasource's items are requested. You can control these properties with the `setOffset` and `setLimit` methods. This allows a widget to not show all data at once. Instead it can show only a single page when you set the proper offset and limit, or the widget will load additional data whenever it is needed if you increase the limit.

The following code sample sets the offset and limit to load datasource items for a specific range:

```ts
this.props.myDataSource.setOffset(20);
this.props.myDataSource.setLimit(10);
```

You can use the `setOffset` and `setLimit` methods to create a widget with pagination support (assuming widget properties are configured as follows):

```ts
interface MyListWidgetsProps {
    myDataSource: ListValue;
    pageSize: number;
}
```

To set the number of items requested by the datasource you can use the `setLimit` in the constructor of the widget:

```ts
export default class PagedWidget extends Component<PagedWidgetProps> {
    constructor(props: PagedWidgetProps) {
        super(props);

        props.myDataSource.setLimit(props.pageSize);
    }
}
```

To switch to a different page you can change the offset with the `setOffset` method:

```tsx
const ds = this.props.myDataSource;
const current = this.props.myDataSource.offset;
<button onClick={() => ds.setOffset(current - this.props.pageSize)}>
    Previous
</button>
<button onClick={() => ds.setOffset(current + this.props.pageSize)}>
    Next
</button>
```

The `hasMoreItems` property indicates whether there are more objects beyond the limit of the most recent list. When a widget does not show all the records immediately by setting a limit with `setLimit` and allows the user to load additional data, you can use this property to make it clear in the user interface that the user has reached the end of the list.

The following code sample shows a 'load more' button only when there is more data available, and loads additional data when the user clicks the button:

```tsx
const currentLimit = this.props.myDataSource.limit;
this.props.myDataSource.hasMoreItems &&
<button 
    onClick={() => this.props.myDataSource.setLimit(currentLimit + 10)}
>
    Load more
</button>
```

When a `limit` is set to *0*, that case is handled in a special way. In this case `ListValue` will avoid sending a request to the server to retrieve data and will immediately return an empty result. This property can be used to build widgets that load their data "lazily": only when and if a specific condition is met.

The following code sample loads the data only if a button is pressed:

```tsx
export default const LazyWidget = (props: LazyWidgetProps) => {
    useMemo(() => props.myDataSource.setLimit(0), []);
    return props.myDataSource.items?.length ? (
        props.myDataSource.items.map((i) => <div key={i.id}>Item</div>)
    ) : (
        <button onClick={() => props.myDataSource.setLimit(undefined)}>Load data</button>
    );
}
```

The `totalCount` property is the total number of objects the datasource can return. Calculating a total count might consume significant resources and is only returned when the widget indicated needs a total count by calling the `requestTotalCount(true)` method. When possible, please use the `hasMoreItems` instead of the `totalCount` property.

The following code sample shows how to request the total count to be returned:

```ts
export default class PagedWidget extends Component<PagedWidgetProps> {
    constructor(props: PagedWidgetProps) {
        super(props);
    
        props.myDataSource.requestTotalCount(true);
    }
}
```

The `items` property contains all the requested data items of the datasource. However, it is not possible to access domain data directly from `ListValue`, as every object is represented only by GUID in the `items` array. Instead, a list of items may be used in combination with other properties, for example with a property of type [`attribute`](property-types-pluggable-widgets#attribute), [`action`](property-types-pluggable-widgets#action), or [`widgets`](property-types-pluggable-widgets#widgets).
 

### 4.8 ListActionValue {#listactionvalue}

`ListActionValue` represents actions that may be applied to items from `ListValue`. The `ListActionValue` is an object and its definition is as follows:

```ts
export interface ListActionValue {
    get: (item: ObjectItem) => ActionValue;
}
```

<a name="get-function"></a>In order to call an action on a particular item of a `ListValue` first an instance of `ActionValue` should be obtained by calling `ListActionValue.get` with the item (assuming widget properties are configured as follows):

```ts
interface MyListWidgetsProps {
    myDataSource: ListValue;
    myListAction: ListActionValue;
}
```

The following code sample shows how to call `myListAction` on the first element from the `myDataSource`.

```ts
const actionOnFirstItem = this.props.myDataSource.myListAction.get(this.props.myDataSource.item[0]);

actionOnFirstItem.execute();
```

In this code sample, checks of status `myDataSource` and availability of items are omitted for simplicity. See the [ActionValue section](#actionvalue) for more information about the usage of `ActionValue`.

{{% alert type="info" %}}
The `get` method was introduced in Mendix 9.0.

You can obtain an instance of `ActionValue` by using the `ListActionValue` as a function and calling it with an item. This is deprecated, will be removed in Mendix 10, and should be replaced by a call to the `get` function as described [above](#get-function).
{{% /alert %}}

### 4.9 ListAttributeValue {#listattributevalue}

`ListAttributeValue` represents an [attribute property](property-types-pluggable-widgets#attribute) that is linked to a data source.
This allows the client component to access attribute values on individual items from a `ListValue`. `ListAttributeValue` is an object and its definition is as follows:

```ts
export interface ListAttributeValue<T extends AttributeValue> {
    get: (item: ObjectItem) => EditableValue<T>; // NOTE: EditableValue obtained from ListAttributeValue always readonly
}
```

The type `<T>` depends on the allowed value types as configured for the attribute property.

{{% alert type="warning" %}}
Due to a technical limitation it is not yet possible to edit attributes obtained via `ListAttributeValue`. `EditableValue`s returned by `ListAttributeValue` are always **readonly**.
{{% /alert %}}

In order to work with the attribute value of a particular item of a `ListValue` first an instance of `EditableValue` should be obtained by calling `ListAttributeValue.get` with the item (assuming widget properties are configured as follows with an attribute of type `string`):

```ts
interface MyListWidgetsProps {
    myDataSource: ListValue;
    myAttributeOnDatasource: ListAttributeValue<string>;
}
```

The following code sample shows how to get an `EditableValue` that represents a read-only value of an attribute of the first element from the `myDataSource`.

```ts
const attributeValue = this.props.myAttributeOnDatasource.get(this.props.myDataSource.items[0]);
```

Note: in this code sample checks of status of `myDataSource` and availability of items are omitted for simplicity. See [EditableValue section](#editable-value) for more information about usage of `EditableValue`.

### 4.10 ListWidgetValue {#listwidgetvalue}

`ListWidgetValue` represents a [widget property](property-types-pluggable-widgets#widgets) that is linked to a data source. 
This allows the client component to render child widgets with items from a `ListValue`.
`ListWidgetValue` is an object and its definition is as follows:

```ts
export interface ListWidgetValue {
    get: (item: ObjectItem) => ReactNode;
}
```

For clarity, consider the following example using `ListValue` together with the `widgets` property type. When the `widgets` property named `myWidgets` is configured to be tied to a `datasource` named `myDataSource`, the client component props appear as follows:

```ts
interface MyListWidgetsProps {
    myDataSource: ListValue;
    myWidgets: (i: ObjectItem) => ReactNode;
}
```

Because of the above configurations, the client component may render every instance of widgets with a specific item from the list like this:

```ts
this.props.myDataSource.items.map(i => this.props.myWidgets.get(i));
```

{{% alert type="info" %}}
The `get` method was introduced in Mendix 9.0.

You can obtain an instance of `ReactNode` by using the `ListWidgetValue` as a function and calling it with an item. This is deprecated, will be removed in Mendix 10, and should be replaced by a call to the `get` function as described [above](#get-function).
{{% /alert %}}


### 4.11 ListExpressionValue {#listexpressionvalue}

`ListExpressionValue` represents an [expression property](property-types-pluggable-widgets#expression) or [text template property](property-types-pluggable-widgets#texttemplate) that is linked to a data source. This allows the client component to access expression or text template values for individual items from a `ListValue`. `ListExpressionValue` is an object and its definition is as follows:

```ts
export interface ListExpressionValue<T extends AttributeValue> {
    get: (item: ObjectItem) => DynamicValue<T>
};
```

The type `<T>` depends on the return type as configured for the expression property. For a text template property, this type is always `string`.

In order to work with the expression or text template value of a particular item of a `ListValue`, first an instance of `DynamicValue` should be obtained by calling `ListExpressionValue.get` with the item (assuming widget properties are configured as follows with an expression of type `boolean`):

```ts
interface MyListWidgetsProps {
    myDataSource: ListValue;
    myExpressionOnDatasource: ListExpressionValue<boolean>;
    myTextTemplateOnDatasource: ListExpressionValue<string>;
}
```

The following code sample shows how to get a `DynamicValue` that represents the value of an expression for the first element from the `myDataSource`.

```ts
const expressionValue = this.props.myDataSource.myExpressionOnDatasource.get(this.props.myDataSource.item[0]);
```

## 5 Exposed Modules

### 5.1 Icon {#icon}

Mendix platform exposes two versions of an `Icon` react component: `mendix/components/web/Icon` and `mendix/components/native/Icon`. Both components are useful helpers to render `WebIcon` and `NativeIcon` values respectively. They should be passed through an `icon` prop. The native `Icon` component additionally accepts `color` (`string`) and `size` (`number`) props.

## 6 Exposed Libraries {#exposed-libraries}

### 6.1 React and React Native {#exposed-react}

The Mendix Platform re-exports [react](https://www.npmjs.com/package/react), [react-dom](https://www.npmjs.com/package/react-dom), and [react-native](https://www.npmjs.com/package/react-native) packages to pluggable widgets. `react` is available to all components. `react-dom` is available only to components running in web or hybrid mobile apps. `react-native` is available only to components running in native mobile apps.

Mendix provides you with `react` version `17.*.*` (in npm terms `^17.0.1`) and a matching version of `react-dom`. For `react-native` Mendix exposes version `0.63.*` (in npm terms `~0.63.3`).

Patch versions might change from one minor release of Mendix to another. 

### 6.2 Big.js

The Mendix Platform uses [big.js](https://www.npmjs.com/package/big-js) to represent and operate on numbers. Mendix 9.0 re-exports version 6.0.

## 7 Native Dependencies

Sometimes for widgets it is necessary to rely on the existing community libraries of `react` and `react-native`. With widgets targeting a web platform it is easy to include those libraries as they can be shipped together with a widget by bundling them into the widget's package. That is often not the case with libraries targeting a native platform, as some of them require a setup of Android- and iOS-specific code into a Mendix native app or [Make It Native](/refguide/getting-the-make-it-native-app) app. For more information, see [Declaring Native Dependencies](native-dependencies).

## 8 Read More

* [Pluggable Widgets API Documentation](pluggable-widgets)
* [Pluggable Widget Property Types Documentation](property-types-pluggable-widgets)
* [How to Build Pluggable Widgets](/howto/extensibility/pluggable-widgets)
* [Declaring Native Dependencies](native-dependencies)
