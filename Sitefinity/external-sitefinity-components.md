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
3. Edit the EmbeddedResource tag and add a LogicalName tag so that the code looks like this:
```
    <EmbeddedResource Include="Modules\SiteAlerts\Views\DefaultView.cshtml">
      <LogicalName>MyProject.Common.Mvc.Views.SiteAlerts.DefaultView.cshtml</LogicalName>
    </EmbeddedResource>
```
4. Repeat for each view in the module.
5. Save the project file, close it, and reopen it.

Step 3 in this sequence instructs the compiler to make the embedded view available in a different namespace than Visual Studio's default scheme. The scheme we are replacing it with is a scheme that Sitefinity uses to find embedded views in a .dll.

Unfortunately as of Visual Studio 2017 there isn't a way to change the logical name of an embedded file through the Properties interface.

### VirtualPathProviderViewEngine

We will be providing a VirtualPathProviderViewEngine which will be invoked on Sitefinity startup. The engine itself will be in it's own class file while configuration for the engine will be provided via a class file that stores application constants.

The class file for the engine is as follows.

```
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
using System.Web.Mvc;

namespace MyProject.Common.Startup
{
    public class VirtualPathProviderViewEngine : System.Web.Mvc.VirtualPathProviderViewEngine
    {
        private readonly string[] _rootModuleNamespaces;

        private VirtualPathProviderViewEngine(string[] rootModuleNamespaces)
        {
            _rootModuleNamespaces = rootModuleNamespaces;
        }

        public static VirtualPathProviderViewEngine Create(string[] rootModuleNamespaces, string[] locationFormats)
        {
            var engine = new VirtualPathProviderViewEngine(rootModuleNamespaces);

            // find all the namespaces created by the /Application/Modules folder structure
            // and pluck the top-level names out to create new search locations for views;
            // we do not want to search the actual folder structure via system.io because we do 
            // not want to deploy that folder.

            var moduleTypes = Assembly.GetExecutingAssembly().GetTypes().Where(engine.IsAssemblyMatchRootNamespace).Select(t => t.Namespace).Distinct();
            var classNames = moduleTypes.Select(t => t.Split('.')[3]).ToList();
            var additionalLocations = new List<string>();

            classNames.ForEach(t => additionalLocations.AddRange(locationFormats.Where(f => f.Contains("{1}")).Select(format => format.Replace("{1}", t))));

            var allLocationFormats = locationFormats.Concat(additionalLocations).ToArray();

            engine.FileExtensions = new[] { "cshtml", "vbhtml", "aspx", "ascx" };
            engine.AreaViewLocationFormats = allLocationFormats;
            engine.AreaMasterLocationFormats = allLocationFormats;
            engine.AreaPartialViewLocationFormats = allLocationFormats;
            engine.ViewLocationFormats = allLocationFormats;
            engine.MasterLocationFormats = allLocationFormats;
            engine.PartialViewLocationFormats = allLocationFormats;

            return engine;
        }

        protected override IView CreatePartialView(ControllerContext controllerContext, string partialPath)
        {
            if (partialPath.EndsWith(".cshtml") || partialPath.EndsWith(".vbhtml"))
            {
                return new RazorView(controllerContext, partialPath, null, false, null);
            }

            return new WebFormView(controllerContext, partialPath);
        }

        protected override IView CreateView(ControllerContext controllerContext, string viewPath, string masterPath)
        {
            if (viewPath.EndsWith(".cshtml") || viewPath.EndsWith(".vbhtml"))
            {
                return new RazorView(controllerContext, viewPath, masterPath, false, null);
            }

            return new WebFormView(controllerContext, viewPath, masterPath);
        }

        private bool IsAssemblyMatchRootNamespace(System.Type assemblyType)
        {
            return _rootModuleNamespaces.Any(ns => (assemblyType.Namespace?.StartsWith(ns) ?? false) && assemblyType.Namespace != ns);
        }
    }
}
```

### VirtualPathProvider 
Sitefinity needs to know how to get to the views for your components considering a) the views are going to be located in an external project and b) the views will not be located in the standard folder format that Sitefinity expects.

In the common project we will have a "Constants" file that contains constant values that may be used by the common project and by the other Sitefinity projects. In it we will define virtual paths that Sitefinity may use to help find views that are embedded in the compiled project.

The stub of the constants file may look as follows.

```
using MyProject.Common.Startup

namespace MyProject.Common
{
    public class Constants
    {
        private struct VirtualPathsParameters
        {
            public static string[] LocationFormats => new[]
            {
                "~/Frontend-Assembly/MyProject.Common/Modules/{1}/{0}.cshtml",
                "~/Frontend-Assembly/MyProject.Common/Modules/{1}/Views/{0}.cshtml",
                "~/Frontend-Assembly/MyProject.Common/MVC/{1}/{0}.cshtml",
                "~/Frontend-Assembly/MyProject.Common/MVC/Views/{1}/{0}.cshtml",
                "~/Frontend-Assembly/MyProject.Common/Modules/Views/{0}.cshtml",
            };
            
            public static string[] RootModuleNamespaces => new[]
            {
                "MyProject.Common.Modules",
            };
        }        
        
        public static VirtualPathProviderViewEngine VirtualPageProviderViewEngine => VirtualPathProviderViewEngine.Create(VirtualPathsParameters.RootModuleNamespaces, VirtualPathsParameters.LocationFormats);

    }
}
```

Note that this will provide Sitefinity locations to look for views within the common project. If you want to use this same technique for embedding views in the Sitefinity project itself, repeat the inclusion of the VirtualPathsParameters struct and the VirtualPageProviderViewEngine static property in the Sitefinity project constants file, changing namespaces as necessary.

### 
