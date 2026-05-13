# Software Architecture Report – Hugo

Project repository: https://github.com/SwDA-Team-04/hugo  
Report part: Architecture  
Diagram tool: C4-PlantUML (PlantUML text diagrams embedded below).  
Scope: the Hugo static site generator, focusing on the main Hugo CLI binary and its internal package-level components.

## 1. Context level

```plantuml
@startuml Hugo_Context
!include <C4/C4_Context>

Person(author, "Content author / site developer", "Creates content, templates, assets, and configuration; runs Hugo locally or in CI.")
Person(themeDev, "Theme or module developer", "Publishes reusable Hugo themes, modules, templates, and assets.")
Person(visitor, "Website visitor", "Consumes the generated static website.")

System(hugo, "Hugo", "Go-based static site generator that builds a complete static website from content, templates, assets, modules, and configuration.")

System_Ext(workspace, "Local project workspace", "Content, configuration, themes, layouts, assets, static files, and generated public output.")
System_Ext(moduleRepos, "Git / module repositories", "Remote sources for Hugo Modules, themes, and dependencies.")
System_Ext(browser, "Web browser", "Used with the embedded development server and LiveReload.")
System_Ext(hosting, "Static hosting / cloud storage", "GitHub Pages, Netlify, S3, GCS, Azure Storage, or another static hosting target.")

Rel(author, hugo, "Runs CLI commands: build, server, new, mod, deploy")
Rel(themeDev, moduleRepos, "Publishes modules/themes")
Rel(hugo, workspace, "Reads content/config/templates/assets; writes generated site")
Rel(hugo, moduleRepos, "Downloads or resolves modules and dependencies")
Rel(hugo, browser, "Serves generated site and LiveReload during development")
Rel(hugo, hosting, "Uploads or provides generated static files")
Rel(visitor, hosting, "Requests static HTML, CSS, JS, images")

@enduml
```

At context level, Hugo is a developer-facing build tool rather than an online business service. Its primary stakeholders are content authors/site developers, theme/module developers, and deployment operators. Website visitors normally do not interact with Hugo at runtime; they interact with the static output produced by Hugo. This is important architecturally because most quality concerns concentrate on fast local builds, configurability, extensibility, and correctness of generated files, rather than high availability of a production server.

## 2. Container level

```plantuml
@startuml Hugo_Container
!include <C4/C4_Container>

Person(author, "Content author / site developer")
System_Boundary(hugoSystem, "Hugo") {
  Container(cli, "Hugo CLI binary", "Go executable", "Single deployable application. Parses commands, loads configuration, builds sites, serves local development previews, processes assets, and publishes output.")
}

ContainerDb_Ext(workspace, "Project workspace / file system", "Files", "Content, config, layouts, assets, static files, modules cache, resources cache, and public output.")
System_Ext(moduleRepos, "Git/module repositories", "External repositories for modules and themes.")
System_Ext(browser, "Browser", "Receives local development server responses and LiveReload events.")
System_Ext(cloud, "Cloud/static hosting", "Optional deployment target.")

Rel(author, cli, "Executes commands", "CLI")
Rel(cli, workspace, "Reads/writes", "OS filesystem abstraction")
Rel(cli, moduleRepos, "Resolves modules", "Git/Go module mechanisms")
Rel(cli, browser, "Serves pages and reload events", "HTTP/WebSocket in server mode")
Rel(cli, cloud, "Deploys generated site when deploy edition is used", "Cloud provider APIs")

@enduml
```

Hugo is best described as a modular monolith packaged as a single Go executable. At C4 container level there is therefore one internal deployable container: the Hugo CLI binary. The project also contains optional behavior controlled by build tags/editions, for example deployment-related features, but these are still compiled into a binary rather than deployed as independent services. The local project workspace, module repositories, browser, and cloud targets are shown as external systems because they are not independently deployable containers owned by Hugo.

This container decision also explains why microservice-specific architectural concerns such as distributed transactions, service discovery, and runtime inter-service consistency are not central in Hugo. Hugo’s main communication style is local call-based interaction within the binary plus file-system state exchange with the project workspace.

## 3. Component level – Hugo CLI binary

```plantuml
@startuml Hugo_Component
!include <C4/C4_Component>

Container_Boundary(cli, "Hugo CLI binary") {
  Component(cmd, "Command interface", "Go packages: main, commands", "Defines CLI commands and command-specific flows such as build, server, deploy, mod, new, and config.")
  Component(conf, "Configuration and dependency assembly", "Go packages: config, deps, helpers, hugofs, modules", "Loads configuration, builds file-system abstractions, initializes shared services, caches, loggers, template/resource providers, and module settings.")
  Component(build, "Build orchestrator", "Go package: hugolib", "Coordinates sites, languages, page trees, rendering, rebuilds, and high-level site generation.")
  Component(content, "Content/source model", "Go packages: source, parser, markup, resources/page", "Reads content and data, parses front matter/markup, and represents pages and site data.")
  Component(tpl, "Template engine", "Go packages: tpl, tplimpl", "Loads and executes templates and template functions.")
  Component(assets, "Resource and asset pipeline", "Go packages: resources, transform, minifiers, media, internal/js", "Processes images, CSS, Sass, JS, minification, media types, and resource transformations.")
  Component(pub, "Publisher", "Go package: publisher", "Writes generated pages and transformed resources to the publish directory.")
  Component(server, "Development server and watcher", "Go packages: commands/server, watcher, livereload", "Serves local output, watches file changes, and triggers rebuilds/LiveReload.")
  Component(deploy, "Deployment adapter", "Go package: deploy", "Optional direct publishing to cloud/static hosting targets.")
}

ContainerDb_Ext(fs, "Project file system", "Files")
System_Ext(mods, "Git/module repositories")
System_Ext(browser, "Browser")
System_Ext(cloud, "Cloud/static hosting")

Rel(cmd, conf, "initializes")
Rel(cmd, build, "executes build/server use cases")
Rel(conf, mods, "resolves modules")
Rel(conf, fs, "creates filesystem views")
Rel(build, content, "loads and organizes pages")
Rel(build, tpl, "renders content with templates")
Rel(build, assets, "processes resources")
Rel(build, pub, "publishes output")
Rel(server, build, "triggers rebuilds")
Rel(server, browser, "serves local preview and LiveReload")
Rel(pub, fs, "writes public output")
Rel(deploy, cloud, "uploads generated site")

@enduml
```

The most central component is `hugolib`, which acts as the build orchestrator. It coordinates site/language variants, page trees, rendering, caching, rebuilds, and publishing. The command component is an outer adapter around this core: `main.go` delegates execution to the `commands` package, and the `commands` package wires the individual CLI commands. The `deps` package is an architectural assembly point that collects logger, filesystem, path/content/source specs, resource spec, template store, metrics, and build listeners. `resources` and `publisher` implement the asset/output side of the pipeline.

## 4. Relationship with Clean Architecture

Hugo partially resembles Clean Architecture, but it is not a strict implementation of it.

| Clean Architecture concept | Hugo mapping | Assessment |
|---|---|---|
| Entities / policies | Page/resource/site/configuration concepts, especially packages under `resources/page`, `resources/resource`, `hugolib`, `config` | Hugo has strong domain concepts: page trees, output formats, resources, languages, and site variants. |
| Use cases | Build site, render page, process resource, serve/rebuild, deploy | Use cases are implemented mainly inside `hugolib` and `commands`, not in a clearly isolated application layer. |
| Interface adapters | CLI commands, filesystem abstraction, template adapters, markup converters, publisher | Several adapters exist, and abstractions such as interfaces and filesystem wrappers improve testability. |
| Frameworks/drivers/details | Cobra/simplecobra, Go filesystem libraries, fsnotify, HTTP server, cloud SDKs, image/markup/minification libraries | Details are present at the edges, but some are also imported by central packages. |

The main Clean Architecture mismatch is dependency direction. In a strict Clean Architecture, source-code dependencies should point inward toward higher-level policies. Hugo’s central build packages import many concrete infrastructure/detail packages, including template implementation, publishing, filesystem watching, resource processing, and JavaScript processing. This is understandable for a high-performance static site generator, but it weakens the separation between policy and detail.

## 5. SOLID observations at component level

**SRP risk – build core as a broad orchestrator.** `hugolib.Site` and `hugolib.HugoSites` concentrate multiple reasons to change: language/site variants, page trees, rendering, caching, rebuild state, Git/CODEOWNERS metadata, and progress reporting. This is an acceptable orchestrator pattern in a monolith, but it increases change amplification.

**DIP risk – concrete details imported by central components.** The build core depends on concrete packages such as template implementation, publisher, filesystem events, resource processing, and JavaScript processing. A stricter DIP-oriented design would place more interfaces in the core and move concrete adapters outward.

**ISP risk – large dependency object.** `deps.Deps` works as a dependency assembly object, but it exposes many services and states to consumers. This can make components depend on more than they actually need, which is close to an Interface Segregation Principle smell. Smaller role-specific interfaces could reduce accidental coupling.

**OCP support and limitation.** Hugo is extensible through templates, modules, media types, markup converters, and resource transformations. However, adding a new first-class build concern may still require changes in central orchestration code. This is a common trade-off in a performance-sensitive modular monolith.

## 6. Architectural characteristics and metrics-based reasoning

**Performance.** Performance is a driving characteristic. Hugo’s architecture keeps the build process in one executable and avoids distributed runtime calls. Caches, resource specs, page trees, and parallel build mechanisms support fast generation and rebuilds.

**Configurability and extensibility.** Hugo supports many site shapes through configuration, templates, modules, output formats, languages, and asset pipelines. This is supported by separated packages for configuration, modules, templates, markup, resources, and output.

**Deployability.** Deployability is strong because Hugo is shipped as a single binary. Different editions are compiled with build tags, but deployment is still operationally simple compared with distributed systems.

**Maintainability and testability.** Maintainability is mixed. Package-level separation is visible and the use of interfaces/filesystem abstractions helps testing. However, the centrality of `hugolib` and `deps` creates high coupling hotspots. Suggested metrics to support this claim are: afferent/efferent coupling per Go package, instability `I = Ce / (Ca + Ce)`, number of internal imports, and cycle detection in the package graph.

**Architectural style.** Hugo is a modular monolith with some pipeline-like behavior. It is monolithic because all main behavior is deployed as one binary. It is modular because packages separate CLI, build orchestration, content parsing, template rendering, resource processing, publishing, server/watch mode, and deployment. It is pipeline-like because the typical build flow transforms source content and assets into static output through parsing, rendering, transformation, and publishing stages.

## 7. Summary

Hugo’s architecture is a good fit for its problem domain: a fast, flexible static site generator. The system is not a distributed web application; its architecture optimizes local build performance, configurability, static output generation, and simple deployment. The main architectural trade-off is centralization: `hugolib` and `deps` make orchestration efficient and practical, but they also become coupling hotspots and partially violate Clean Architecture dependency direction and some SOLID principles. For the project report, the most important conclusion is that Hugo should be evaluated as a modular monolith with a build pipeline, not as a microservice or layered enterprise application.
