### Overview
This document provides a brief explanation on how to create Sitefinity widgets in a secondary Visual Studio project so that they may be consumed in multiple Sitefinity web projects. Using this approach will allow a developer to reuse widgets by including a reference to a common Sitefinity widgets project and importing precompiled front-end widgets by way of front-end build tools such as grunt or gulp.

#### File Organization
The organization of files as described in this document do not strictly adhere to how Sitefinity organizes code in a base Sitefinity web project. Rather, the files are organized in such a way as to make it easier for a developer to find files for a particular component as they generally will be located beneath a different folder for each component.

At present this does not interfere with the operation of Sitefinity but may be subject to breaking in future versions.

#### Content Types and the Module Builder
This document is strictly concerned with reusing widgets and front-end components. Widgets that rely on custom content types will require those content types to be exported from the module builder and imported via the module building into your new Sitefinity site.

That may present its own set of challenges with respect to content type updates, versioning and deployment.

### Content Types
The first thing to consider for your module is the content types. It will be necessary to export content types from Sitefinity and import them into your new Sitefinity sites.

### Organizing the Code
Consider a Site Alerts widget that contains a controller, a couple views, and a model. In the common project we set up the files as follows:

```
Modules
    +-- [ other modules ]
    +-- SiteAlerts
        +-- Views
            +-- DefaultView.cshtml
            +-- DesignerView.Simple.cshtml
        +-- SiteAlertsController.cs
        +-- SiteAlertsModel.cs
    +-- [ other modules ]
```

Let's assume that the controller handles any required data fetching and mapping of that data out to the view model.

Let's also assume that the namespace for your common project and your Sitefinity project will respectively be:
- MyProject.Common
- MyProject.Sitefinity

### Embedding the Views
We will need to embed the view files so that they may be found inside the .dll of the compiled common project. This will require the following steps.

1. Right click on each view, select Properties and change the Build Action to _Embedded Resource_. Save the project.
2. Right click on the project and select Unload Project. Once the project is unloaded, right click on it again and select Edit. Search for the first view file in the project, "Modules\SiteAlerts\Views\DefaultView.cshtml". You should find an xml entry that looks like this:
```
<EmbeddedResource Include="Modules\SiteAlerts\Views\DefaultView.cshtml" />
```
3. 


### VirtualPathProvider 
Sitefinity needs to know how to get to the views for your components considering a) the views are going to be located in an external project and b) the views will not be located in the standard folder format that Sitefinity expects.

In the common project we have a "Constants" file that contains constant values that may be used by the common project and by the other Sitefinity projects. In it we will define virtual paths that Sitefinity may use to help find views that are d

