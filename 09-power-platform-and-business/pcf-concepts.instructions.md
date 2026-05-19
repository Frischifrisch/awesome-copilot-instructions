---
description: 'PCF (Power Apps Component Framework): overview, limitations, community and samples'
applyTo: '**'
---


# Power Apps Component Framework Overview

Power Apps component framework empowers professional developers and app makers to create code components for model-driven and canvas apps. These code components can be used to enhance the user experience for users working with data on forms, views, dashboards, and canvas app screens.

## Key Capabilities

You can use PCF to:
- Replace a column on a form that displays a numeric text value with a `dial` or `slider` code component
- Transform a list into an entirely different visual experience bound to the dataset, like a `Calendar` or `Map`

## Important Limitations

- Power Apps component framework works only on Unified Interface and not on the legacy web client
- Power Apps component framework is currently not supported for on-premises environments

## How PCF Differs from Web Resources

Unlike HTML web resources, code components are:
- Rendered as part of the same context
- Loaded at the same time as any other components
- Provide a seamless experience for the user

Code components can be:
- Used across the full breadth of Power Apps capabilities
- Reused many times across different tables and forms
- Bundled with all HTML, CSS, and TypeScript files into a single solution package
- Moved across environments
- Made available via AppSource

## Key Advantages

### Rich Framework APIs
- Component lifecycle management
- Contextual data and metadata access
- Seamless server access via Web API
- Utility and data formatting methods
- Device features: camera, location, microphone
- User experience elements: dialogs, lookups, full-page rendering

### Development Benefits
- Support for modern web practices
- Optimized for performance
- High reusability
- Bundle all files into a single solution file
- Handle being destroyed and reloaded for performance reasons while preserving state

## Licensing Requirements

Power Apps component framework licensing is based on the type of data and connections used:

### Premium Code Components
Code components that connect to external services or data directly via the user's browser client (not through connectors):
- Considered premium components
- Apps using these become premium
- End-users require Power Apps licenses

Declare as premium by adding to manifest:
```xml
<external-service-usage enabled="true">
  <domain>www.microsoft.com</domain>
</external-service-usage>
```

### Standard Code Components
Code components that don't connect to external services or data:
- Apps using these with standard features remain standard
- End-users require minimum Office 365 license

**Note**: If using code components in model-driven apps connected to Microsoft Dataverse, end users will require Power Apps licenses.

## Related Resources

- [What are code components?](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/custom-controls-overview)
- [Code components for canvas apps](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/component-framework-for-canvas-apps)
- [Create and build a code component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/create-custom-controls-using-pcf)
- [Learn Power Apps component framework](https://learn.microsoft.com/en-us/training/paths/use-power-apps-component-framework)
- [Use code components in Power Pages](https://learn.microsoft.com/en-us/power-apps/maker/portals/component-framework)

## Training Resources

- [Create components with Power Apps Component Framework - Training](https://learn.microsoft.com/en-us/training/paths/create-components-power-apps-component-framework/)
- [Microsoft Certified: Power Platform Developer Associate](https://learn.microsoft.com/en-us/credentials/certifications/power-platform-developer-associate/)

---


# Limitations

With Power Apps component framework, you can create your own code components to improve the user experience in Power Apps and Power Pages. Even though you can create your own components, there are some limitations that restrict developers implementing some features in the code components. Below are some of the limitations:

## 1. Dataverse Dependent APIs Not Available for Canvas Apps

Microsoft Dataverse dependent APIs, including WebAPI, are not available for Power Apps canvas applications yet. For individual API availability, see [Power Apps component framework API reference](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/reference/).

## 2. Bundle External Libraries or Use Platform Libraries

Code components should either use [React controls & platform libraries](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/react-controls-platform-libraries) or bundle all the code including external library content into the primary code bundle. 

To see an example of how the Power Apps command line interface can help with bundling your external library content into a component-specific bundle, see [Angular flip component](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/sample-controls/angular-flip-control) example.

## 3. Do Not Use HTML Web Storage Objects

Code components should not use the HTML web storage objects, like `window.localStorage` and `window.sessionStorage`, to store data. Data stored locally on the user's browser or mobile client is not secure and not guaranteed to be available reliably.

## 4. Custom Auth Not Supported in Canvas Apps

Custom auth in code components is not supported in Power Apps canvas applications. Use connectors to get data and take actions instead.

## Related Topics

- [Power Apps component framework API reference](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/reference/)
- [Power Apps component framework overview](https://learn.microsoft.com/en-us/power-apps/developer/component-framework/overview)

---


# PCF Community Resources

The Power Apps Component Framework has a vibrant community that creates and shares resources, tools, and knowledge. This guide provides links to key community resources.

## PCF Gallery

**[PCF Gallery](https://pcf.gallery)**

The PCF Gallery is the central hub for discovering, sharing, and learning about PCF components.

**What You'll Find:**
- Community-created components
- Component demonstrations and screenshots
- Source code links
- Installation instructions
- Component ratings and reviews
- Search and filtering capabilities

**How to Use:**
- Browse components by category
- Search for specific functionality
- Download components for your projects
- Submit your own components to share with the community
- Learn from real-world component implementations

## Community Videos

Learn from expert developers through these comprehensive video tutorials:

### Getting Started
- **Getting started with code components with OOB React and Fluent UI by PowerfulDevs** - Introduction to building components with React and Fluent UI
- **Getting Started With Power Apps Component Framework by April Dunnam** - Beginner-friendly introduction to PCF

### Deep Dives
- **Power Apps Component Framework Manifest File Explained by April Dunnam** - Detailed explanation of the manifest structure
- **Easier Development with React Controls and Platform Libraries by Scott Durow** - Using React and platform-provided libraries
- **Understanding the Power Apps Component Framework by PowerfulDevs** - Comprehensive overview of the framework

### Debugging & Development
- **How to Debug Power Apps Component Framework Components by April Dunnam** - Debugging techniques and tools
- **Using React and the Fluent UI in Power Apps Component Framework by Microsoft** - Official guidance on React/Fluent UI integration

### Advanced Topics
- **Power Apps Component Framework: Datasets with React and Azure Maps by Nishant Rana** - Working with datasets and external APIs
- **How to Upload and Display Images with Power Apps Component Framework by April Dunnam** - Image handling in components
- **Deep Dive: Power Apps Component Framework API by PowerfulDevs** - Comprehensive API exploration

### Styling & Theming
- **Using Fluent UI Components in Power Apps Component Framework by Sancho Harker** - Styling with Fluent UI
- **Power Apps Component Framework: Styling and Theming by Microsoft** - Official theming guidance

### Additional Resources
- **Power Apps Component Framework End to End Series by April Dunnam** - Complete walkthrough series
- More videos available through community channels and Microsoft's official documentation

## Community Blogs

Stay updated with these excellent community blogs:

1. **Sancho Harker** - Advanced PCF techniques and best practices
2. **Benedikt Bergmann** - Component architecture and patterns
3. **Andrew Butenko** - PCF development tips and tools
4. **Nishant Rana** - Integration scenarios and advanced features
5. **OlivierFlying** - Performance optimization and debugging
6. **Ramakrishnan Raman** - Real-world implementation examples
7. **Temmy Wahyu Raharjo** - Component design patterns
8. **Scott Durow** - Platform libraries and React components
9. **Guido Preite** - Enterprise PCF development
10. **Ulrikke Akerbæk** - Canvas apps and PCF integration

**Topics Covered:**
- Component development tutorials
- Best practices and patterns
- Performance optimization
- Integration with external services
- Troubleshooting common issues
- New feature announcements
- Real-world use cases

## Community Tools

### PCF Builder for XrmToolBox

**What It Does:**
- Simplifies PCF component creation
- Provides visual manifest editor
- Generates boilerplate code
- Streamlines component testing

**Key Features:**
- Visual manifest designer
- Property configuration UI
- Resource management
- Quick component scaffolding
- Integration with XrmToolBox ecosystem

**Best For:**
- Rapid prototyping
- Learning PCF structure
- Quick component setup
- Manifest validation

### PCF Builder for VS Code

**What It Does:**
- Integrates PCF development into Visual Studio Code
- Provides IntelliSense and code completion
- Simplifies workflow without leaving the editor

**Key Features:**
- VS Code extension
- Command palette integration
- Manifest schema validation
- Code snippets for common patterns
- Integrated terminal commands

**Best For:**
- Developers who prefer VS Code
- Streamlined workflow
- Modern development experience
- Built-in debugging support

## How to Engage with the Community

### Contribute Components
- Share your components on PCF Gallery
- Publish source code on GitHub
- Write blog posts about your implementation

### Learn from Others
- Browse PCF Gallery for inspiration
- Watch community videos for tutorials
- Read blogs for best practices and tips

### Get Help
- Microsoft Learn Q&A forums
- Power Apps Community forums
- GitHub repository issues and discussions
- Twitter/LinkedIn Power Platform community

### Stay Updated
- Follow community bloggers
- Subscribe to YouTube channels
- Join Power Platform user groups
- Attend community calls and events

## Community Best Practices

1. **Share Your Work**: Contribute components and knowledge back to the community
2. **Provide Feedback**: Report issues and suggest improvements
3. **Document Well**: Include clear documentation with your components
4. **Test Thoroughly**: Ensure components work across platforms before sharing
5. **Follow Standards**: Use established patterns and naming conventions
6. **Be Helpful**: Answer questions and help other developers

## Additional Resources

- **Microsoft Learn**: Official documentation and tutorials
- **Power Platform Community**: Forums and discussion boards
- **GitHub**: Source code repositories and samples
- **Power CAT (Customer Advisory Team)**: Enterprise guidance and patterns
- **User Groups**: Local and virtual meetups

## Contributing to PCF Gallery

To add your component to PCF Gallery:

1. Create a well-documented component
2. Test across target platforms
3. Prepare screenshots and demos
4. Submit to pcf.gallery
5. Include source code link (GitHub recommended)
6. Provide clear installation instructions

## Finding the Right Resource

- **Just Starting?** → Watch April Dunnam's "Getting Started" video
- **Need a Component?** → Browse PCF Gallery
- **Learning Best Practices?** → Read community blogs
- **Want Quick Setup?** → Use PCF Builder tools
- **Debugging Issues?** → Watch debugging videos and read troubleshooting blogs
- **Advanced Techniques?** → Follow Scott Durow and PowerfulDevs content

The PCF community is welcoming and eager to help. Don't hesitate to reach out, ask questions, and share your own experiences!

---


# How to Use the Sample Components

All the sample components listed under this section are available to download from [github.com/microsoft/PowerApps-Samples/tree/master/component-framework](https://github.com/microsoft/PowerApps-Samples/tree/master/component-framework) so that you can try them out in your model-driven or canvas apps.

The individual sample component topics under this section provide you an overview of the sample component, its visual appearance, and a link to the complete sample component.

## Before You Can Try the Sample Components

To try the sample components, you must first:

- [Download](https://docs.github.com/repositories/working-with-files/using-files/downloading-source-code-archives#downloading-source-code-archives-from-the-repository-view) or [clone](https://docs.github.com/repositories/creating-and-managing-repositories/cloning-a-repository) this repository [github.com/microsoft/PowerApps-Samples](https://github.com/microsoft/PowerApps-Samples).
- Install [Install Power Platform CLI for Windows](https://learn.microsoft.com/en-us/power-platform/developer/cli/introduction#install-power-platform-cli-for-windows).

## Try the Sample Components

Follow the steps in the [README.md](https://github.com/microsoft/PowerApps-Samples/blob/master/component-framework/README.md) to generate solutions containing the controls so you can import and try the sample components in your model-driven or canvas app.

## How to Run the Sample Components

Use the following steps to import and try the sample components in your model-driven or canvas app.

### Step-by-Step Process

1. **Download or clone the repository**
   - [Download](https://docs.github.com/repositories/working-with-files/using-files/downloading-source-code-archives#downloading-source-code-archives-from-the-repository-view) or [clone](https://docs.github.com/repositories/creating-and-managing-repositories/cloning-a-repository) [github.com/microsoft/PowerApps-Samples](https://github.com/microsoft/PowerApps-Samples).

2. **Open Developer Command Prompt**
   - Open a [Developer Command Prompt for Visual Studio](https://learn.microsoft.com/visualstudio/ide/reference/command-prompt-powershell) and navigate to the `component-framework` folder.
   - On Windows, you can type `developer command prompt` in Start to open a developer command prompt.

3. **Install dependencies**
   - Navigate to the component you want to try, for example `IncrementControl`, and run:
   ```bash
   npm install
   ```

4. **Restore project**
   - After the command has completed, run:
   ```bash
   msbuild /t:restore
   ```

5. **Create solution folder**
   - Create a new folder inside the sample component folder:
   ```bash
   mkdir IncrementControlSolution
   ```

6. **Navigate to solution folder**
   ```bash
   cd IncrementControlSolution
   ```

7. **Initialize solution**
   - Inside the folder you created, run the `pac solution init` command:
   ```bash
   pac solution init --publisher-name powerapps_samples --publisher-prefix sample
   ```
   > **Note**: This command creates a new file named `IncrementControlSolution.cdsproj` in the folder.

8. **Add component reference**
   - Run the `pac solution add-reference` command with the `path` set to the location of the `.pcfproj` file:
   ```bash
   pac solution add-reference --path ../../IncrementControl
   ```
   or
   ```bash
   pac solution add-reference --path ../../IncrementControl/IncrementControl.pcfproj
   ```
   > **Important**: Reference the folder that contains the `.pcfproj` file for the control you want to add.

9. **Build the solution**
   - To generate a zip file from your solution project, run the following three commands:
   ```bash
   msbuild /t:restore
   msbuild /t:rebuild /restore /p:Configuration=Release
   msbuild
   ```
   - The generated solution zip file becomes in the `IncrementControlSolution\bin\debug` folder.

10. **Import the solution**
    - Now that you have the zip file, you have two options:
      - Manually [import the solution](https://learn.microsoft.com/powerapps/maker/data-platform/import-update-export-solutions) into your environment using [make.powerapps.com](https://make.powerapps.com/).
      - Alternatively, to import the solution using Power Apps CLI commands, see the [Connecting to your environment](https://learn.microsoft.com/powerapps/developer/component-framework/import-custom-controls#connecting-to-your-environment) and [Deployment](https://learn.microsoft.com/powerapps/developer/component-framework/import-custom-controls#deploying-code-components) sections.

11. **Add components to apps**
    - Finally, to add code components to your model-driven and canvas apps, see:
      - [Add components to model-driven apps](https://learn.microsoft.com/powerapps/developer/component-framework/add-custom-controls-to-a-field-or-entity)
      - [Add components to canvas apps](https://learn.microsoft.com/powerapps/developer/component-framework/component-framework-for-canvas-apps#add-components-to-a-canvas-app)

## Available Sample Components

The repository contains numerous sample components including:

- AngularJSFlipControl
- CanvasGridControl
- ChoicesPickerControl
- ChoicesPickerReactControl
- CodeInterpreterControl
- ControlStateAPI
- DataSetGrid
- DeviceApiControl
- FacepileReactControl
- FluentThemingAPIControl
- FormattingAPIControl
- IFrameControl
- ImageUploadControl
- IncrementControl
- LinearInputControl
- LocalizationAPIControl
- LookupSimpleControl
- MapControl
- ModelDrivenGridControl
- MultiSelectOptionSetControl
- NavigationAPIControl
- ObjectOutputControl
- PowerAppsGridCustomizerControl
- PropertySetTableControl
- ReactStandardControl
- TableControl
- TableGrid
- WebAPIControl

Each sample demonstrates different aspects of the Power Apps component framework and can serve as a learning resource or starting point for your own components.