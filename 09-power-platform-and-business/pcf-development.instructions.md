---
description: 'PCF development: best practices, API, code components, events and React libraries'
applyTo: '**/*.{ts,tsx,js,json,xml,pcfproj,csproj,css,html}'
---


# Best Practices and Guidance for Code Components

Developing, deploying, and maintaining code components needs a combination of knowledge across multiple areas. This article outlines established best practices and guidance for professionals developing code components.

## Power Apps Component Framework

### Avoid Deploying Development Builds to Dataverse

Code components can be built in [production or development mode](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/code-components-alm#building-pcfproj-code-component-projects). Avoid deploying development builds to Dataverse since they adversely affect the performance and can even get blocked from deployment due to their size. Even if you plan to deploy a release build later, it can be easy to forget to redeploy if you don't have an automated release pipeline. More information: [Debugging custom controls](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/debugging-custom-controls).

### Avoid Using Unsupported Framework Methods

These include using undocumented internal methods that exist on the `ComponentFramework.Context`. These methods might work but, because they're not supported, they might stop working in future versions. Use of control script that accesses host application HTML Document Object Model (DOM) isn't supported. Any parts of the host application DOM that are outside the code component boundary, are subject to change without notice.

### Use `init` Method to Request Network Required Resources

When the hosting context loads a code component, the `init` method is first called. Use this method to request any network resources such as metadata instead of waiting for the `updateView` method. If the `updateView` method is called before the requests return, your code component must handle this state and provide a visual loading indicator.

### Clean Up Resources Inside the `destroy` Method

The hosting context calls the `destroy` method when a code component is removed from the browser DOM. Use the `destroy` method to close any `WebSockets` and remove event handlers that are added outside of the container element. If you're using React, use `ReactDOM.unmountComponentAtNode` inside the `destroy` method. Cleaning up resources in this way prevents any performance issues caused by code components being loaded and unloaded within a given browser session.

### Avoid Unnecessary Calls to Refresh on a Dataset Property

If your code component is of type dataset, the bound dataset properties expose a `refresh` method that causes the hosting context to reload the data. Calling this method unnecessarily impacts the performance of your code component.

### Minimize Calls to `notifyOutputChanged`

In some circumstances, it's undesirable for updates to a UI control (such as keypresses or mouse move events) to each call `notifyOutputChanged`, as more calls would result in many more events propagating to the parent context than needed. Instead, consider using an event when a control loses focus, or when the user's touch or mouse event completes.

### Check API Availability

When developing code components for different hosts (model-driven apps, canvas apps, portals), always check the availability of the APIs you're using for support on those platforms. For example, `context.webAPI` isn't available in canvas apps. For individual API availability, see [Power Apps component framework API reference](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/reference/).

### Manage Temporarily Null Property Values Passed to `updateView`

Null values are passed to the `updateView` method when data isn't ready. Your components should account for this situation and expect that the data could be null, and that a subsequent `updateView` cycle can include updated values. `updateView` is available for both [standard](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/reference/control/updateview) and [React](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/reference/react-control/updateview) components.

## Model-Driven Apps

### Don't Interact Directly with `formContext`

If you have experience working with client API, you might be used to interacting with `formContext` to access attributes, controls, and call API methods such as `save`, `refresh`, and `setNotification`. Code components are expected to work across various products like model-driven apps, canvas apps, and dashboards, therefore they can't have a dependency on `formContext`.

A workaround is to make the code component bound to a column and add an `OnChange` event handler to that column. The code component can update the column value, and the `OnChange` event handler can access the `formContext`. Support for the custom events will be added in the future, which will enable communicating changes outside of a control without adding a column configuration.

### Limit Size and Frequency of Calls to the `WebApi`

When using the `context.WebApi` methods, limit both the number of calls and the amount of data. Each time you call the `WebApi`, it counts towards the user's API entitlement and service protection limits. When performing CRUD operations on records, consider the size of the payload. In general, the larger the request payload, the slower your code component is.

## Canvas Apps

### Minimize the Number of Components on a Screen

Each time you add a component to your canvas app, it takes a finite amount of time to render. Render time increases with each component you add. Carefully measure the performance of your code components as you add more to a screen using the Developer Performance tools.

Currently, each code component bundles their own library of shared libraries such as Fluent UI and React. Loading multiple instances of the same library won't load these libraries multiple times. However, loading multiple different code components results in the browser loading multiple bundled versions of these libraries. In the future, these libraries will be able to be loaded and shared with code components.

### Allow Makers to Style Your Code Component

When app makers consume code components from inside a canvas app, they want to use a style that matches the rest of their app. Use input properties to provide customization options for theme elements such as color and size. When using Microsoft Fluent UI, map these properties to the theme elements provided by the library. In the future, theming support will be added to code components to make this process easier.

### Follow Canvas Apps Performance Best Practices

Canvas apps provide a wide set of best practices from inside the app and solution checker. Ensure your apps follow these recommendations before you add code components. For more information, see:

- [Tips to improve canvas app performance](https://learn.microsoft.com/en-us/powerapps/maker/canvas-apps/performance-tips)
- [Considerations for optimized performance in Power Apps](https://powerapps.microsoft.com/blog/considerations-for-optimized-performance-in-power-apps/)

## TypeScript and JavaScript

### ES5 vs ES6

By default, code components target ES5 to support older browsers. If you don't want to support these older browsers, you can change the target to ES6 inside your `pcfproj` folder's `tsconfig.json`. More information: [ES5 vs ES6](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/debugging-custom-controls#es5-vs-es6).

### Module Imports

Always bundle the modules that are required as part of your code component instead of using scripts that are required to be loading using the `SCRIPT` tag. For example, if you wanted to use a non-Microsoft charting API where the sample shows adding `<script type="text/javascript" src="somechartlibrary.js></script>` to the page, this isn't supported inside a code component. Bundling all of the required modules isolates the code component from other libraries and also supports running in offline mode.

> **Note**: Support for shared libraries across components using library nodes in the component manifest is not yet supported.

### Linting

Linting is where a tool can scan the code for potential issues. The template used by `pac pcf init` installs the `eslint` module to your project and configures it by adding an `.eslintrc.json` file.

To configure, at the command-line use:

```bash
npx eslint --init
```

Then answer the following questions when prompted:

- **How would you like to use ESLint?** Answer: To check syntax, find problems, and enforce code style
- **What type of modules does your project use?** Answer: JavaScript modules (import/export)
- **Which framework does your project use?** Answer: React
- **Does your project use TypeScript?** Answer: Yes
- **Where does your code run?** Answer: Browser
- **How would you like to define a style for your project?** Answer: Answer questions about your style
- **What format do you want your config file to be in?** Answer: JSON
- **What style of indentation do you use?** Answer: Spaces
- **What quotes do you use for strings?** Answer: Single
- **What line endings do you use?** Answer: Windows
- **Do you require semicolons?** Answer: Yes

Before you can use `eslint`, you need to add some scripts to the `package.json`:

```json
"scripts": {
   ...
   "lint": "eslint MY_CONTROL_NAME --ext .ts,.tsx",
   "lint:fix": "npm run lint -- --fix"
}
```

Now at the command-line, you can use:

```bash
npm run lint:fix
```

Additionally, you can add files to ignore by adding to the `.eslintrc.json`:

```json
"ignorePatterns": ["**/generated/*.ts"]
```

## HTML Browser User Interface Development

### Use Microsoft Fluent UI React

[Fluent UI React](https://developer.microsoft.com/fluentui#/get-started/web) is the official [open source](https://github.com/microsoft/fluentui) React front-end framework designed to build experiences that fit seamlessly into a broad range of Microsoft products. Power Apps itself uses Fluent UI, meaning you are able to create UI that's consistent with the rest of your apps.

#### Use Path-Based Imports from Fluent to Reduce Bundle Size

Currently, the code component templates used with `pac pcf init` won't use tree-shaking, which is the process where `webpack` detects modules imported that aren't used and removes them. If you import from Fluent UI using the following command, it imports and bundles the entire library:

```typescript
import { Button } from '@fluentui/react'
```

To avoid importing and bundling the entire library, you can use path-based imports where the specific library component is imported using the explicit path:

```typescript
import { Button } from '@fluentui/react/lib/Button';
```

Using the specific path reduces your bundle size both in development and release builds.

#### Optimize React Rendering

When using React, follow React specific best practices regarding minimizing rendering of components:

- Only make a call to `ReactDOM.render` inside the `updateView` method when a bound property or framework aspect change requires the UI to reflect the change. You can use `updatedProperties` to determine what has changed.
- Use `PureComponent` (with class components) or `React.memo` (with function components) where possible to avoid unnecessary re-renders.
- For large React components, deconstruct your UI into smaller components to improve performance.
- Avoid use of arrow functions and function binding inside the render function as these practices create a new callback closure with each render.

### Check Accessibility

Ensure that code components are accessible so that keyboard only and screen-reader users can use them:

- Provide keyboard navigation alternatives to mouse/touch events
- Ensure that `alt` and [ARIA](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA) (Accessible Rich Internet Applications) attributes are set so that screen readers announce an accurate representation of the code components interface
- Modern browser developer tools offer helpful ways to inspect accessibility

More information: [Create accessible canvas apps in Power Apps](https://learn.microsoft.com/en-us/powerapps/maker/canvas-apps/accessible-apps).

### Always Use Asynchronous Network Calls

When making network calls, never use a synchronous blocking request since this causes the app to stop responding and result in slow performance. More information: [Interact with HTTP and HTTPS resources asynchronously](https://learn.microsoft.com/en-us/powerapps/developer/model-driven-apps/best-practices/business-logic/interact-http-https-resources-asynchronously).

### Write Code for Multiple Browsers

Model-driven apps, canvas apps, and portals all support multiple browsers. Be sure to only use techniques that are supported on all modern browsers, and test with a representative set of browsers for your intended audience.

- [Limits and configurations](https://learn.microsoft.com/en-us/powerapps/maker/canvas-apps/limits-and-config)
- [Supported web browsers](https://learn.microsoft.com/en-us/power-platform/admin/supported-web-browsers-and-mobile-devices)
- [Browsers used by office](https://learn.microsoft.com/en-us/office/dev/add-ins/concepts/browsers-used-by-office-web-add-ins)

### Code Components Should Plan for Supporting Multiple Clients and Screen Formats

Code components can be rendered in multiple clients (model-driven apps, canvas apps, portals) and screen formats (mobile, tablet, web).

- Using `trackContainerResize` allows code components to respond to changes in the available width and height
- Using `allocatedHeight` and `allocatedWidth` can be combined with `getFormFactor` to determine if the code component is running on a mobile, tablet, or web client
- Implementing `setFullScreen` allows users to expand to use the entire available screen available where space is limited
- If the code component can't provide a meaningful experience in the given container size, it should disable functionality appropriately and provide feedback to the user

### Always Use Scoped CSS Rules

When you implement styling to your code components using CSS, ensure that the CSS is scoped to your component using the automatically generated CSS classes applied to the container `DIV` element for your component. If your CSS is scoped globally, it might break the existing styling of the form or screen where the code component is rendered.

For example, if your namespace is `SampleNamespace` and your code component name is `LinearInputComponent`, you would add a custom CSS rule using:

```css
.SampleNamespace\.LinearInputComponent rule-name
```

### Avoid Use of Web Storage Objects

Code components shouldn't use the HTML web storage objects, like `window.localStorage` and `window.sessionStorage`, to store data. Data stored locally on the user's browser or mobile client isn't secure and not guaranteed to be available reliably.

## ALM/Azure DevOps/GitHub

See the article on [Code component application lifecycle management (ALM)](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/code-components-alm) for best practices on code components with ALM/Azure DevOps/GitHub.

## Related Articles

- [What are code components](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/custom-controls-overview)
- [Code components for canvas apps](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/component-framework-for-canvas-apps)
- [Create and build a code component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/create-custom-controls-using-pcf)
- [Learn Power Apps component framework](https://learn.microsoft.com/en-us/training/paths/use-power-apps-component-framework)
- [Use code components in Power Pages](https://learn.microsoft.com/en-us/power-apps/maker/portals/component-framework)

---


# Power Apps Component Framework API Reference

The Power Apps component framework provides a rich set of APIs that enable you to create powerful code components. This reference lists all available interfaces and their availability across different app types.

## API Availability

The following table shows all API interfaces available in the Power Apps component framework, along with their availability in model-driven apps and canvas apps.

| API | Model-driven apps | Canvas apps |
|-----|------------------|-------------|
| AttributeMetadata | Yes | No |
| Client | Yes | Yes |
| Column | Yes | Yes |
| ConditionExpression | Yes | Yes |
| Context | Yes | Yes |
| DataSet | Yes | Yes |
| Device | Yes | Yes |
| Entity | Yes | Yes |
| Events | Yes | Yes |
| Factory | Yes | Yes |
| Filtering | Yes | Yes |
| Formatting | Yes | Yes |
| ImageObject | Yes | Yes |
| Linking | Yes | Yes |
| Mode | Yes | Yes |
| Navigation | Yes | Yes |
| NumberFormattingInfo | Yes | Yes |
| Paging | Yes | Yes |
| Popup | Yes | Yes |
| PopupService | Yes | Yes |
| PropertyHelper | Yes | Yes |
| Resources | Yes | Yes |
| SortStatus | Yes | Yes |
| StandardControl | Yes | Yes |
| UserSettings | Yes | Yes |
| Utility | Yes | Yes |
| WebApi | Yes | Yes |

## Key API Namespaces

### Context APIs

The `Context` object provides access to all framework capabilities and is passed to your component's lifecycle methods. It contains:

- **Client**: Information about the client (form factor, network status)
- **Device**: Device capabilities (camera, location, microphone)
- **Factory**: Factory methods for creating framework objects
- **Formatting**: Number and date formatting
- **Mode**: Component mode and tracking
- **Navigation**: Navigation methods
- **Resources**: Access to resources (images, strings)
- **UserSettings**: User settings (locale, number format, security roles)
- **Utils**: Utility methods (getEntityMetadata, hasEntityPrivilege, lookupObjects)
- **WebApi**: Dataverse Web API methods

### Data APIs

- **DataSet**: Work with tabular data
- **Column**: Access column metadata and data
- **Entity**: Access record data
- **Filtering**: Define data filtering
- **Linking**: Define relationships
- **Paging**: Handle data pagination
- **SortStatus**: Manage sorting

### UI APIs

- **Popup**: Create popup dialogs
- **PopupService**: Manage popup lifecycle
- **Mode**: Get component rendering mode

### Metadata APIs

- **AttributeMetadata**: Column metadata (model-driven only)
- **PropertyHelper**: Property metadata helpers

### Standard Control

- **StandardControl**: Base interface for all code components with lifecycle methods:
  - `init()`: Initialize component
  - `updateView()`: Update component UI
  - `destroy()`: Cleanup resources
  - `getOutputs()`: Return output values

## Usage Guidelines

### Model-Driven vs Canvas Apps

Some APIs are only available in model-driven apps due to platform differences:

- **AttributeMetadata**: Model-driven only - provides detailed column metadata
- Most other APIs are available in both platforms

### API Version Compatibility

- Always check the API availability for your target platform (model-driven or canvas)
- Some APIs may have different behaviors across platforms
- Test components in the target environment to ensure compatibility

### Common Patterns

1. **Accessing Context APIs**
   ```typescript
   // In init or updateView
   const userLocale = context.userSettings.locale;
   const isOffline = context.client.isOffline();
   ```

2. **Working with DataSet**
   ```typescript
   // Access dataset records
   const records = context.parameters.dataset.records;
   
   // Get sorted columns
   const sortedColumns = context.parameters.dataset.sorting;
   ```

3. **Using WebApi**
   ```typescript
   // Retrieve records
   context.webAPI.retrieveMultipleRecords("account", "?$select=name");
   
   // Create record
   context.webAPI.createRecord("contact", data);
   ```

4. **Device Capabilities**
   ```typescript
   // Capture image
   context.device.captureImage();
   
   // Get current position
   context.device.getCurrentPosition();
   ```

5. **Formatting**
   ```typescript
   // Format date
   context.formatting.formatDateLong(date);
   
   // Format number
   context.formatting.formatDecimal(value);
   ```

## Best Practices

1. **Type Safety**: Use TypeScript for type checking and IntelliSense
2. **Null Checks**: Always check for null/undefined before accessing API objects
3. **Error Handling**: Wrap API calls in try-catch blocks
4. **Platform Detection**: Check `context.client.getFormFactor()` to adapt behavior
5. **API Availability**: Verify API availability for your target platform before use
6. **Performance**: Cache API results when appropriate to avoid repeated calls

## Additional Resources

- For detailed documentation on each API, refer to the [Power Apps component framework API reference](https://learn.microsoft.com/power-apps/developer/component-framework/reference/)
- Sample code for each API is available in the [PowerApps-Samples repository](https://github.com/microsoft/PowerApps-Samples/tree/master/component-framework)

---


# Code Components

Code components are a type of solution component that can be included in a solution file and imported into different environments. They can be added to both model-driven and canvas apps.

## Three Core Elements

Code components consist of three elements:

1. **Manifest**
2. **Component implementation**
3. **Resources**

> **Note**: The definition and implementation of code components using Power Apps component framework is the same for both model-driven and canvas apps. The only difference is the configuration part.

## Manifest

The manifest is the `ControlManifest.Input.xml` metadata file that defines a component. It is an XML document that describes:

- The name of the component
- The kind of data that can be configured, either a `field` or a `dataset`
- Any properties that can be configured in the application when the component is added
- A list of resource files that the component needs

### Manifest Purpose

When a user configures a code component, the data in the manifest file filters the available components so that only valid components for the context are available for configuration. The properties defined in the manifest file are rendered as configuration columns so that users can specify values. These property values are then available to the component at runtime.

More information: [Manifest schema reference](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/)

## Component Implementation

Code components are implemented using TypeScript. Each code component must include an object that implements the methods described in the code component interface. The [Power Platform CLI](https://learn.microsoft.com/en-us/power-platform/developer/cli/introduction) auto-generates an `index.ts` file with stubbed implementations using the `pac pcf init` command.

### Required Methods

The component object implements these lifecycle methods:

- **init** (Required) - Called when the page loads
- **updateView** (Required) - Called when app data changes
- **getOutputs** (Optional) - Returns values when user changes data
- **destroy** (Required) - Called when the page closes

### Component Lifecycle

#### Page Load

When the page loads, the application creates an object using data from the manifest:

```typescript
var obj = new <"namespace on manifest">.<"constructor on manifest">();
```

Example:
```typescript
var controlObj = new SampleNameSpace.LinearInputComponent();
```

The page then initializes the component:

```typescript
controlObj.init(context, notifyOutputChanged, state, container);
```

**Init Parameters:**

| Parameter | Description |
|-----------|-------------|
| `context` | Contains all information about how the component is configured and all parameters. Access input properties via `context.parameters.<property name from manifest>`. Includes Power Apps component framework APIs. |
| `notifyOutputChanged` | Alerts the framework whenever the component has new outputs ready to be retrieved asynchronously. |
| `state` | Contains component data from the previous page load if explicitly stored using `setControlState` method. |
| `container` | An HTML div element to which developers can append HTML elements for the UI. |

#### User Changes Data

When a user interacts with your component to change data, call the `notifyOutputChanged` method passed in the `init` method. The platform responds by calling the `getOutputs` method, which returns values with the changes made by the user. For a `field` component, this would typically be the new value.

#### App Changes Data

If the platform changes the data, it calls the `updateView` method of the component and passes the new context object as a parameter. This method should be implemented to update the values displayed in the component.

#### Page Close

When a user navigates away from the page, the code component loses scope and all memory allocated for objects is cleared. However, some methods (like event handlers) may stay and consume memory based on browser implementation.

**Best Practices:**
- Implement the `setControlState` method to store information for the next time within the same session
- Implement the `destroy` method to remove cleanup code such as event handlers when the page closes

## Resources

The resource node in the manifest file refers to the resources that the component requires to implement its visualization. Each code component must have a resource file to construct its visualization. The `index.ts` file generated by the tooling is a `code` resource. There must be at least 1 code resource.

### Additional Resources

You can define additional resource files in the manifest:

- CSS files
- Image web resources
- Resx web resources for localization

More information: [resources element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/resources)

## Related Resources

- [Create and build a code component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/create-custom-controls-using-pcf)
- [Learn how to package and distribute extensions using solutions](https://learn.microsoft.com/en-us/power-platform/alm/solution-concepts-alm)

---


# Define Events (Preview)

[This topic is pre-release documentation and is subject to change.]

A common requirement when building custom components with the Power Apps Component Framework is the ability to react to events generated within the control. These events can be invoked either due to user interaction or programmatically via code. For example, an application can have a code component that lets a user build a product bundle. This component can also raise an event which could show product information in another area of the application.

## Component Data Flow

The common data flow for a code component is data flowing from the hosting application into the control as inputs and updated data flowing out of the control to the hosting form or page. This diagram shows the standard pattern of data flow for a typical PCF component:

![Shows that data update from the code component to the binding field triggers the OnChange event](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/media/component-events-onchange-example.png)

The data update from the code component to the bound field triggers the `OnChange` event. For most component scenarios, this is enough and makers just add a handler to trigger subsequent actions. However, a more complicated control might require events to be raised that aren't field updates. The event mechanism allows code components to define events that have separate event handlers.

## Using Events

The event mechanism in PCF is based on the standard event model in JavaScript. The component can define events in the manifest file and raise these events in the code. The hosting application can listen to these events and react to them.

### Define Events in Manifest

The component defines events using the [event element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/event) in the manifest file. This data allows the respective hosting application to react to events in different ways.

```xml
<property
  name="sampleProperty"
  display-name-key="Property_Display_Key"
  description-key="Property_Desc_Key"
  of-type="SingleLine.Text"
  usage="bound"
  required="true"
/>
<event
  name="customEvent1"
  display-name-key="customEvent1"
  description-key="customEvent1"
/>
<event
  name="customEvent2"
  display-name-key="customEvent2"
  description-key="customEvent2"
/>
```

### Canvas Apps Event Handling

Canvas apps react to the event using Power Fx expressions:

![Shows the custom events in the canvas apps designer](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/media/custom-events-in-canvas-designer.png)

### Model-Driven Apps Event Handling

Model Driven Apps use the [addEventHandler method](https://learn.microsoft.com/en-us/power-apps/developer/model-driven-apps/clientapi/reference/controls/addeventhandler) to associate event handlers to custom events for a component.

```javascript
const controlName1 = "cr116_personid";

this.onLoad = function (executionContext) {
  const formContext = executionContext.getFormContext();

  const sampleControl1 = formContext.getControl(controlName1);
  sampleControl1.addEventHandler("customEvent1", this.onSampleControl1CustomEvent1);
  sampleControl1.addEventHandler("customEvent2", this.onSampleControl1CustomEvent2);
}
```

> **Note**: These events occur separately for each instance of the code component in the app.

## Defining an Event for Model-Driven Apps

For model-driven apps you can pass a payload with the event allowing for more complex scenarios. For example in the diagram below the component passes a callback function in the event allowing the script handling to call back to the component.

![In this example, the component passes a callback function in the event allowing the script handling to call back to the component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/media/passing-payload-in-events.png)

```javascript
this.onSampleControl1CustomEvent1 = function (params) {
   //alert(`SampleControl1 Custom Event 1: ${params}`);
   alert(`SampleControl1 Custom Event 1`);
}.bind(this);

this.onSampleControl2CustomEvent2 = function (params) {
  alert(`SampleControl2 Custom Event 2: ${params.message}`);
  // prevent the default action for the event
  params.callBackFunction();
}
```

## Defining an Event for Canvas Apps

Makers configure an event using Power Fx on the PCF control in the properties pane.

## Calling an Event

See how to call an event in [Events](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/reference/events).

## Next Steps

[Tutorial: Define a custom event in a component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/tutorial-define-event)

---


# React Controls & Platform Libraries

When you use React and platform libraries, you're using the same infrastructure used by the Power Apps platform. This means you no longer have to package React and Fluent libraries individually for each control. All controls share a common library instance and version to provide a seamless and consistent experience.

## Benefits

By reusing the existing platform React and Fluent libraries, you can expect:

- **Reduced control bundle size**
- **Optimized solution packaging**
- **Faster runtime transfer, scripting, and control rendering**
- **Design and theme alignment with the Power Apps Fluent design system**

> **Note**: With GA release, all existing virtual controls will continue to function. However, they should be rebuilt and deployed using the latest CLI version (>=1.37) to facilitate future platform React version upgrades.

## Prerequisites

As with any component, you must install [Visual Studio Code](https://code.visualstudio.com/Download) and the [Microsoft Power Platform CLI](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/powerapps-cli#install-microsoft-power-platform-cli).

> **Note**: If you have already installed Power Platform CLI for Windows, make sure you are running the latest version by using the `pac install latest` command. The Power Platform Tools for Visual Studio Code should update automatically.

## Create a React Component

> **Note**: These instructions expect that you have created code components before. If you have not, see [Create your first component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/implementing-controls-using-typescript).

There's a new `--framework` (`-fw`) parameter for the `pac pcf init` command. Set the value of this parameter to `react`.

### Command Parameters

| Parameter | Value |
|-----------|-------|
| --name | ReactSample |
| --namespace | SampleNamespace |
| --template | field |
| --framework | react |
| --run-npm-install | true (default) |

### PowerShell Command

The following PowerShell command uses the parameter shortcuts and creates a React component project and runs `npm-install`:

```powershell
pac pcf init -n ReactSample -ns SampleNamespace -t field -fw react -npm
```

You can now build and view the control in the test harness as usual using `npm start`.

After you build the control, you can package it inside solutions and use it for model-driven apps (including custom pages) and canvas apps like standard code components.

## Differences from Standard Components

### ControlManifest.Input.xml

The [control element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/control) `control-type` attribute is set to `virtual` rather than `standard`.

> **Note**: Changing this value does not convert a component from one type to another.

Within the [resources element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/resources), find two new [platform-library element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/platform-library) child elements:

```xml
<resources>
  <code path="index.ts" order="1" />
  <platform-library name="React" version="16.14.0" />
  <platform-library name="Fluent" version="9.46.2" />
</resources>
```

> **Note**: For more information about valid platform library versions, see Supported platform libraries list.

**Recommendation**: We recommend using platform libraries for Fluent 8 and 9. If you don't use Fluent, you should remove the `platform-library` element where the `name` attribute value is `Fluent`.

### Index.ts

The [ReactControl.init](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/reference/react-control/init) method for control initialization doesn't have `div` parameters because React controls don't render the DOM directly. Instead [ReactControl.updateView](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/reference/react-control/updateview) returns a ReactElement that has the details of the actual control in React format.

### bundle.js

React and Fluent libraries aren't included in the package because they're shared, therefore the size of bundle.js is smaller.

## Sample Controls

The following controls are included in the samples. They function the same as their standard versions but offer better performance since they are virtual controls.

| Sample | Description | Link |
|--------|-------------|------|
| ChoicesPickerReact | The standard ChoicesPickerControl converted to be a React Control | ChoicesPickerReact Sample |
| FacepileReact | The ReactStandardControl converted to be a React Control | FacepileReact |

## Supported Platform Libraries List

Platform libraries are made available both at the build and runtime to the controls that are using platform libraries capability. Currently, the following versions are provided by the platform and are the highest currently supported versions.

| Library | Package | Build Version | Runtime Version |
|---------|---------|---------------|-----------------|
| React | react | 16.14.0 | 17.0.2 (Model), 16.14.0 (Canvas) |
| Fluent | @fluentui/react | 8.29.0 | 8.29.0 |
| Fluent | @fluentui/react | 8.121.1 | 8.121.1 |
| Fluent | @fluentui/react-components | >=9.4.0 <=9.46.2 | 9.68.0 |

> **Note**: The application might load a higher compatible version of a platform library at runtime, but the version might not be the latest version available. Fluent 8 and Fluent 9 are each supported but can not both be specified in the same manifest.

## FAQ

### Q: Can I convert an existing standard control to a React control using platform libraries?

A: No. You must create a new control using the new template and then update the manifest and index.ts methods. For reference, compare the standard and react samples described above.

### Q: Can I use React controls & platform libraries with Power Pages?

A: No. React controls & platform libraries are currently only supported for canvas and model-driven apps. In Power Pages, React controls don't update based on changes in other fields.

## Related Articles

- [What are code components?](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/custom-controls-overview)
- [Code components for canvas apps](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/component-framework-for-canvas-apps)
- [Create and build a code component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/create-custom-controls-using-pcf)
- [Learn Power Apps component framework](https://learn.microsoft.com/en-us/training/paths/use-power-apps-component-framework)
- [Use code components in Power Pages](https://learn.microsoft.com/en-us/power-apps/maker/portals/component-framework)

---


# Dependent Libraries (Preview)

[This topic is pre-release documentation and is subject to change.]

With model-driven apps, you can reuse a prebuilt library contained in another component that is loaded as a dependency to more than one component.

Having copies of a prebuilt library in multiple controls is undesirable. Reusing existing libraries improves performance, especially when the library is large, by reducing the load time for all components that use the library. Library reuse also helps reduce the maintenance overhead in build processes.

## Before and After

**Before**: Custom library files contained in each PCF component
![Diagram showing custom library files contained in each pcf component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/media/dependent-library-before-example.png)

**After**: Components calling a shared function from a Library Control
![Diagram showing components calling a shared function from a Library Control](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/media/dependent-library-after-example.png)

## Implementation Steps

To use dependent libraries, you need to:

1. Create a **Library component** that contains the library. This component can provide some functionality or only be a container for the library.
2. Configure another component to depend on the library loaded by the library component.

By default, the library loads when the dependent component loads, but you can configure it to load on demand.

This way you can independently maintain the library in the Library Control and the dependent controls don't need to have a copy of the library bundled with them.

## How It Works

You need to add configuration data to your component project so that the build process deploys your libraries the way you want. Set this configuration data by adding or editing the following files:

- **featureconfig.json**
- **webpack.config.js**
- Edit the manifest schema to **Register dependencies**

### featureconfig.json

Add this file to override the default feature flags for a component without modifying the files generated in the `node_modules` folder.

**Feature Flags:**

| Flag | Description |
|------|-------------|
| `pcfResourceDependency` | Enables the component to use a library resource. |
| `pcfAllowCustomWebpack` | Enables the component to use a custom web pack. This feature must be enabled for components that define a library resource. |

By default, these values are `off`. Set them to `on` to override the default.

**Example 1:**
```json
{ 
  "pcfAllowCustomWebpack": "on" 
} 
```

**Example 2:**
```json
{ 
   "pcfResourceDependency": "on",
   "pcfAllowCustomWebpack": "off" 
} 
```

### webpack.config.js

The build process for components uses [Webpack](https://webpack.js.org/) to bundle the code and dependencies into a deployable asset. To exclude your libraries from this bundling, add a `webpack.config.js` file to the project root folder that specifies the alias of the library as `externals`. [Learn more about the Webpack externals configuration option](https://webpack.js.org/configuration/externals/)

This file might look like the following when the library alias is `myLib`:

```javascript
/* eslint-disable */ 
"use strict"; 

module.exports = { 
  externals: { 
    "myLib": "myLib" 
  }, 
}  
```

### Register Dependencies

Use the [dependency element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/dependency) within [resources](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/resources) of the manifest schema.

```xml
<resources>
  <dependency
    type="control"
    name="samples_SampleNS.SampleStubLibraryPCF"
    order="1"
  />
  <code path="index.ts" order="2" />
</resources>
```

### Dependency as On-Demand Load of a Component

Rather than loading the dependent library when a component loads, you can load the dependent library on demand. Loading on demand provides the flexibility for more complex controls to only load dependencies when they're required, especially if the dependent libraries are large.

![Diagram showing the use of a function from a library where the library is loaded on demand](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/media/dependent-library-on-demand-load.png)

To enable on demand loading, you need to:

**Step 1**: Add these [platform-action element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/platform-action), [feature-usage element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/feature-usage), and [uses-feature element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/uses-feature) child elements to the [control element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/control):

```xml
<platform-action action-type="afterPageLoad" />
<feature-usage>
   <uses-feature name="Utility"
      required="true" />
</feature-usage>
```

**Step 2**: Set the `load-type` attribute of the [dependency element](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/manifest-schema-reference/dependency) to `onDemand`.

```xml
<dependency type="control"
      name="samples_SampleNamespace.StubLibrary"
      load-type="onDemand" />
```

## Next Steps

Try a tutorial that walks you through creating a dependent library:

[Tutorial: Use dependent libraries in a component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/tutorial-use-dependent-libraries)