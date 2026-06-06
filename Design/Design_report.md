# Hugo - Design Report

## 1. Dependencies

### 1.1 Intro

#### Methodology and Tools

To analyze Hugo's design, we combined two complementary approaches.

For the code dependency task we used custom PowerShell scripts to analyze Go import statements, identifying files and their respective packages with the highest and lowest number of dependencies. Since Go dependencies are declared explicitly through imports at the package level, this approach provides a direct approximation of code dependency between software modules.

Parallelly, to analyze the software’s knowledge dependencies, we performed a repository mining process on Hugo’s official Git repository. This was done by extracting a log file containing Hugo’s complete commit history. We then utilized **Code Maat**, a specialized command-line tool designed for analyzing a repository’s code’s evolution over time. Specifically, we performed a **Change Coupling analysis** that calculated the temporal co-changes across the codebase.

### 1.2 Code Dependencies

To measure code dependencies, we used two metrics. **Fan-out** refers to the number of other modules that a given file imports, indicating how many external dependencies it relies on. Conversely, **fan-in** counts how many other modules import a given file, indicating how central or reused it is across the codebase. A file with high fan-out is a heavy consumer of other components; a file with high fan-in is a widely shared dependency.

The most highly connected files belong to the `hugolib` package, confirming its role as the core of Hugo. Files such as `site.go`, `hugo_sites.go`, and `hugo_sites_build.go` coordinate multiple subsystems involved in site generation, rendering, resource handling and publishing.

In particular, the file with the highest number of dependencies is `site.go`, containing 62 imports. This file acts as a central orchestration component responsible for coordinating the site generation workflow. Its very high fan-out reflects the need to interact with rendering, configuration, template management, resources, publishing, and multilingual features. The related `Hugo_sites.go` (35 imports) introduces the `HugoSites` structure, a container for multiple Site instances that provides explicit support for multilingual websites, holding the logic core of Hugo.

Several highly connected files also belong to the `commands` package, including `server.go` (containing 51 imports), `hugobuilder.go`, and `commandeer.go`. These files form the CLI (Command Line Interface) layer of Hugo, handling server startup, command execution, build coordination, filesystem watching, terminal interaction, and command parsing. Consequently, they require interaction with other subsystems including configuration, logging, filesystem services, and the build engine, resulting in high fan-out and moderate fan-in.

Another critical component is `templatestore.go` (containing 42 imports), inside the `tplimpl` package, which represents the core of Hugo’s template subsystem. As for `site.go`, it has a high fan-out and a moderate fan-in. It imports `hugofs` for filesystem access and uses the `output` and `media` packages to manage file generation. Also, it interacts with `page` for rendering logic, `metrics` to measure performance, `identity` for dependency tracking (`getIdentity`), and `sitesmatrix` to handle multilingual, version. The `tplimpl` package implements the internal template engine, including parsing, inheritance handling, template composition, as can be inferred from functions such as `parseTemplate`, `applyBaseTemplate`, and `needsBaseTemplate` in `templates.go`.

While the files discussed above are characterised mainly by high fan-out, two other critical components stand out for combining both high fan-out and high fan-in: `allconfig` package with `allconfig.go` (42 imports) and `deps` package with `deps.go` (33 imports). The `allconfig` package centralises all configuration options in Hugo, such as output formats, languages, module settings, and serves as the foundation for the entire system. The `deps` structure acts as a shared dependency container, centralising access to shared services such as filesystem access, caches, logging, resource handling, configuration and templates to the rest of the application. Their high fan-in reflects how broadly they are consumed across the codebase.

By analyzing the flow of dependencies, we can observe that Hugo is built around three major layers:

* CLI Layer (commands package)

* Core (hugolib package)

* Infrastructure and Shared Services (deps, allconfig, helpers, tplimpl, resources, hugofs)

The CLI layer imports the core, which in turn orchestrates the Infrastructure Services.

At the opposite end, several files have zero or very few imports (low fan-out) such as `constants.go`, `compare.go`, and `doc.go`.
These files are at the absolute base of the dependency hierarchy and generally fall into three categories:

* Utility Logic: Files like `compare.go` implement isolated mathematical or sorting algorithms.

* Domain Dictionaries: Files like `constants.go` provide constant definitions and interface.

* Documentation: Files like `doc.go` contain documentation strings.

### 1.3 Knowledge Dependencies

To isolate the repository’s evolutionary behaviors, we executed a Change Coupling analysis via Code Maat, yielding a dataset that maps the system’s Knowledge Dependencies, by showing how often two distinct files are modified within the same commit.
These relationships are evaluated through two metrics:

- **`degree`**: the coupling strength, expressed as a percentage. It represents the probability that a modification in one file necessitates a simultaneous change in the coupled file within the same commit.
- **`average-revs`**: The absolute volume of shared revisions, indicating the total number of times the two files were modified together across the repository's history.

In order to leave out noise, the results were filtered by excluding from the analysis irrelevant entities, such as documentation (`docs/`) and testing suites (`*_test.go`). Additionally, infrastructure dependencies were evaluated separately. For instance, the pair `go.mod` ↔ `go.sum` exhibits an extremely high evolutionary coupling (97% degree across almost 1000 commits), but that extremely high co-change is due to them being tied to Go’s package manager, rather than representing an actual architectural flaw, so it was therefore omitted from the analysis.

The results were sorted decreasingly by `average-revs` and then filtered, only showing file pairs with a `degree` **above 30%**. This specific filtering strategy was chosen because, even though a high coupling percentage is a strong indicator of dependency, it becomes more meaningful when supported by a significant amount of commits.

In the following section, we briefly analyse the most meaningful pairs that resulted from our analysis, ordered by their historical volume:

- **`hugolib/page.go` ↔ `hugolib/site.go` (32% degree, 598 revs):** this was expected, given that a website is essentially a collection of pages, modifying how the global site behaves naturally requires updating how individual pages are processed.
- **`commands/new.go` ↔ `helpers/hugo.go` (34% degree, 171 revs):** this shows that modifying the content creation command often forces developers to update general utility functions in the helpers folder.
- **`commands/commandeer.go` ↔ `commands/server.go` (32% degree, 153 revs):** this highlights that evolutions in command line interface initialization often require a parallel adjustment to the local development server configuration.
- **`hugofs/rootmapping_fs.go` ↔ `hugolib/filesystems/basefs.go` (34% degree, 41 revs):** this indicates that changing how virtual folders are mapped requires synchronous updates to the core filesystem structures.
- **`resources/page/page.go` ↔ `resources/page/page_nop.go` (61% degree, 33 revs):** here we register a very high co-change rate due to the symmetrical nature of the two files, where updating the real page forces a structural alignment of its non-operational fallback counterpart.
- **`hugolib/page__common.go` ↔ `resources/page/page.go` (40% degree, 30 revs)** and **`hugolib/site_new.go` ↔ `resources/page/site.go` (41% degree, 29 revs):** both these pairs reveal a tight coupling between core logical entities and their abstract representations inside the shared resources module.
- **`parser/pageparser/item.go` ↔ `parser/pageparser/pagelexer.go` (57% degree, 25 revs):** this pairing is easily justified by the fact that the `pagelexer.go` file scans the text for special Hugo commands, while `item.go` defines the structure of what was found. If the parsing rules change, the way the pieces are catalogued must change accordingly.

Some of these pairs display a co-change that is easily explained due to the nature of the software’s domain; however, this analysis also highlights some continuous historical bonds between files belonging to conceptually distant packages.

### 1.4 Comparison between Code and Knowledge Dependencies

Comparing the results obtained by analyzing the static dependencies and the repository history, two main cases emerged in which the Change Coupling is inconsistent with the theoretical structure of the software levels:

* Level inconsistency between the pair `commands/new.go` and `helpers/hugo.go`: the static analysis suggests that Hugo is divided into three hierarchical levels (CLI —> Core —> Infrastructure). The `helpers/` folder seems to be part of the Infrastructural Level, collecting general purpose utilities intended to remain independent from specific CLI commands. The data, however, show a strong evolutionary coupling: it is therefore legitimate to deduce that often, as commands to generate new content are modified, it is also necessary to change the utilities. This behavior is compatible with an information leakage phenomenon: design concerns appear to have leaked into a generic utility module. Through a further analysis of the commit logs and the repository history, we noticed that the file `helpers/hugo.go` was the subject of a massive refactoring, likely aimed at improving information hiding.

* Abstraction inconsistencies between the Core (`hugolib/`) and the Resources (`resources/`): the static analysis shows an apparently unidirectional and clean flow, in which the Core modules (such as `page__common.go` and `site_new.go`) import the package `resources/page` to access its abstractions, complying with the hierarchy. From an evolutionary point of view, however, these pairs register a systemic Change Coupling (around 40%). Despite the fact that the static structure introduces dependencies that are supposedly consistent with the hierarchical architecture, the high Change Coupling observed suggests a tight co-evolution between Core and Resources. This could indicate that the separation of concerns is less clear-cut than the static analysis suggests.

Despite the two cases discussed above, the comparison shows an overall coherence between Code and Knowledge Dependencies. In most cases, the evolutionary coupling analysis confirmed the static dependency analysis, showing that files belonging to the same subsystems also tend to evolve together over time and the majority of co-change pairs are also explained by explicit import relationships. These last two findings highlight the value of combining both static and evolutionary analysis: Code Dependencies alone would have suggested a clean, well-layered architecture, while the Knowledge Dependencies expose where that architecture has been or still is undergoing some improvements.

## 2. Patterns

### Adapter Pattern

The Adapter pattern is used to expose the `page.SitesProvider` API without changing the public structure of `HugoSites`. In `hugolib/hugo_sites.go`, `hugoSitesSitesProvider` acts as the adapter by wrapping `*HugoSites` and implementing `Sites()` and `Data()` methods. The adapter is then passed when `HugoInfo` is created in `hugolib/site.go`, where a provider compatible with `page.SitesProvider` and `DataProvider` is required.

This pattern solves an API conflict: `HugoSites` already exposes a public field named `Sites []*Site`, so adding a `Sites()` method directly on the same type would impact the public API and risk breaking existing callers. The adapter allows to solve the problem without changing the source code of `HugoSites`.

An alternative is to use the Facade pattern to introduce a facade that exposes `Sites()` and `Data()` by composing and delegating to `HugoSites`. This would provide a single, simplified object implementing both `page.SitesProvider` and `page.DataProvider`, reducing direct coupling with `HugoSites` internals.
The downside is that this adds an extra abstraction layer and moves consumers to depend on a new API. For this specific naming conflict, the Adapter remains the better choice because it resolves the issue with minimal structural change.

### Builder Pattern

The Builder pattern is used to construct a table of contents by setting parameters incrementally and building it once all parameters are set. In `tablesofcontents/tableofcontents.go`, the `Builder` struct plays the role of the builder, providing methods like `SetIdentifiers()` and `AddAt()` to configure the table step by step. The `Transform()` function in `markup/goldmark/toc.go` uses this pattern to set identifiers (line 62), add headings (line 71), and finally build the complete table (line 132).

This approach is useful when the construction process involves multiple steps and intermediate configurations, allowing the code to remain readable and flexible without requiring a single method with many parameters. 

An alternative would be a factory function like `func Build(ids []string, h *Heading, row, level int) ToC`, which would be simpler for cases with few parameters and would eliminate the need to distribute construction logic across multiple function calls. However, the Builder pattern is more appropriate here because it provides clearer separation of concerns and scales better if more configuration steps are added in the future.


### Decorator Pattern

The Decorator pattern wraps an object to add behavior without changing its original implementation. In `hugofs/decorators.go` this is used to extend an `afero.Fs`: `NewBaseFileDecorator` returns a `baseFileDecoratorFs` that transparently decorates files and directories, supplying extra metadata, an `Opener` function and `JoinStatFunc` function for directories while leaving the underlying `afero` filesystem unchanged.

This pattern solves the problem of attaching project-specific behavior and metadata to third‑party types without forking or modifying the external library. It keeps the extension localized: callers can open files via the provided opener and access enriched file info, and the core `afero` implementation remains untouched.

One alternative is to modify the function to open filesystems directly through the native interface. While this removes an external dependency and allows for unlimited implementation freedom, it requires replacing the current `afero` library. This transition would demand substantial effort to both reimplement existing features and maintain the new custom solution.


### Prototype Pattern

The Prototype pattern is used to create new objects by cloning an existing instance and then modifying the clone.  In `tpl/internal/go_templates/htmltemplate/template.go`, the Template type plays the role of the prototype, and `Template.Clone()` creates a copy that can be extended with variant definitions.

This approach is useful when creating an object from scratch would be more expensive or more verbose than duplicating an existing one. It also makes it easy to reuse a common base template while customizing only the parts that change.

An alternative could be a factory function that creates a new template with a predefined configuration and then adds the variants to it. This would be simpler and clearer if the goal were only to centralize object creation. However, the Prototype pattern is a better fit here because the code needs to duplicate an already initialized template and then customize the copy.


## 3. Summary

To summarize, the Static Dependency analysis and the Change Coupling results are largely consistent, with the `hugolib` package (`site.go` and `hugo_sites.go`) and the template subsystem central to Hugo's structure, while the CLI and infrastructure layers play supporting roles. Nevertheless, evolutionary coupling highlights recurring co-changes, suggesting areas where strict layering is softened by practical concerns.

The pattern inspection confirms pragmatic design choices. The design patterns analysed were Adapter, Builder, Decorator and Prototype. These patterns solved real‑world problems and favoured stability over radical refactoring.

Overall, Hugo balances robustness and pragmatism, opting for compatibility and developer ergonomics when making design trade-offs.


