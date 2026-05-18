# Hugo - Design Report

## 1. Dependencies

### 1.1 Intro
#### Methodology and Tools
To analyze Hugo's design, we combined two complementary approaches.

For the code dependency task we used custom PowerShell scripts to analyze Go import statements, identifying files and their respective packages with the highest and lowest number of dependencies. Since Go dependencies are declared explicitly through imports at the package level, this approach provides a direct approximation of code dependency between software modules.

Parallelly, to analyze the software’s knowledge dependencies, we performed a repository mining process on Hugo’s official Git repository. This was done by extracting a log file containing Hugo’s complete commit history. We then utilized **Code Maat**, a specialized command-line tool designed for analyzing a repository’s code’s evolution over time. Specifically, we performed a **Change Coupling analysis** that calculated the temporal co-changes across the codebase. 

### 1.2 Code Dependencies
Code dependencies: based on imports in source code
• Which files have most or least dependencies? Why?

The most highly connected files belong to the hugolib package, confirming its role as the core of Hugo. Files such as site.go, hugo_sites.go, and hugo_sites_build.go coordinate multiple subsystems involved in site generation, rendering, resource handling and publishing.

In particular, the file with the highest number of dependencies is site.go, containing 62 imports. This file acts as a central orchestration component responsible for coordinating the site generation workflow. Its very high fan-out reflects the need to interact with rendering, configuration, template management, resources, publishing, and multilingual features. The presence of the HugoSites (Hugo_Site.go has itself 35 imports) structure, which acts as a container for multiple Site instances, suggests explicit support for multilingual websites. It creates the final web site and it contains the logic core of Hugo.

Several highly connected files also belong to the commands package, including server.go containing 51 imports, hugobuilder.go, and commandeer.go. These files form the CLI (Command Line Interface) layer of Hugo. These are responsible for server startup, command execution, build coordination, filesystem watching, terminal interaction, and command parsing. Consequently, they require interaction with other subsystems including configuration, logging, filesystem services, and the build engine. In fact, they show high fan-out and moderate fan-in.

Another critical component is templatestore.go containg 42 imports, inside the tplimpl package. It represents the core of Hugo’s template subsystem. As for site.go, it has an high fan-out and a low moderate fan-in, it is the core of the template subsystem. This subsystem imports some packages: hugofs for filesystem, and uses the output and media packages to manage file generation (for instance, resolveOutputFormatAndOrMediaType to resolve the output format suffix and media type). Also, it interacts with page for rendering logic, metrics to measure performance, identity for dependency tracking (getIdentity), and sitesmatrix to handle multilingual, version. The tplimpl package implements the internal template engine, including parsing, inheritance handling, template composition and execution preparation. This can be inferred from functions such as parseTemplate, applyBaseTemplate and needsBaseTemplate inside the templates.go file.. (serve questo livello di approfondimento?)

Two examples of a highly critical components is the allconfig package with allconfig.go (42 imports) and deps package with deps.go(33 imports). they have both high fan-out and high fan-in. Package allconfig which contains all configuration options in Hugo (output formats, languages, module settings) and serves as the foundation for the system. while Deps acts as a shared dependency container (Deps holds dependencies used by many). The Deps structure centralizes shared services such as filesystem access, caches, logging, resource handling, configuration and templates to the rest of the application.

By analyzing the flow of dependencies, we can observe that Hugo is built around three major layers:

•	CLI Layer (commands package)

•	Core (hugolib package)

•	Infrastructure and Shared Services (deps, allconfig, helpers, tplimpl, resources, hugofs)

The CLI layer uses(imports) the core, which in turn uses (orchestrates) the Infrastructure Services.

At the opposite side, several files have zero or very few imports (low fan-out) such as constants.go, compare.go, and doc.go.
These files are at the absolute base of the dependency hierarchy and generally fall into three categories:

•	Utility Logic: Files like compare.go implement isolated mathematical or sorting algorithms.

•	Domain Dictionaries: Files like constants.go provide constant definitions and interface.

•	Documentation: Files like doc.go contain documentation strings.


### 1.3 Knowledge Dependencies 
To isolate the repository’s evolutionary behaviors, we executed a Change Coupling analysis via Code Maat, yielding a dataset that maps the system’s Knowledge Dependencies, by showing how often two distinct files are modified within the same commit. 
These relationships are evaluated through two metrics: 
* **`degree`**: the coupling strength, expressed as a percentage. It represents the probability that a modification in one file necessitates a simultaneous change in the coupled file within the same commit.
* **`average-revs`**: The absolute volume of shared revisions, indicating the total number of times the two files were modified together across the repository's history.
  
In order to leave out noise, the results were filtered by excluding from the analysis irrelevant entities, such as documentation (`docs/`) and testing suites (`*_test.go`). Additionally, infrastructure dependencies were evaluated separately. For instance, the pair `go.mod` ↔ `go.sum` exhibits an extremely high evolutionary coupling (97% degree across almost 1000 commits), but that extremely high co-change is due to them being tied to Go’s package manager, rather than representing an actual architectural flaw, so it was therefore omitted from the analysis. 

The results were sorted decreasingly by `average-revs` and then filtered, only showing file pairs with a `degree` **above 30%**. This specific filtering strategy was chosen because, even though a high coupling percentage is a strong indicator of dependency, it becomes more meaningful when supported by a significant amount of commits.

• Which are not consistent with code dependencies?

### 1.4 Comparison between Code and Knowledge Dependencies


## 2. Patterns
[Assess pattern usage in the code:
• Find at least 4 instances of patterns
• Which classes play which role?
• Why is the pattern used? Which problem does solve?
• Is there an alternative, what would be pros & cons?]


### Decorator Pattern
Decorator pattern allows to wrap an element in a decorator object to add extra features without modify the original element.
#### Who and where?
In ```hugofs``` package, ```decorators.go``` contains a lot of decorators. One of them is  ```NewBaseFileDecorator```,which is a decorator for Filesystems. The Filesystem is wrapped in a ```baseFileDecoratorFs``` struct, this allows to add an Opener function without modifying the underlying Fs.
#### Why?
Thanks to this operation, it's possibile to open a Filesystem using the function specified in ```NewBaseFileDecorator```, that means that a new behavior is been added but the original Filesystem structure is not been modified.
#### A possible alternative
One possible alternative could be to modify the function to open the Fs already provided in ```Filesystem``` interface but in the project Fs are handled using an external library (```afero```), is not possible to do it unless ```afero``` library is no more used.
##### Pros:
The method in the interface can be freely implemented without limitation. Additionally, there wolud be one less dependency.
##### Cons:
It will require a substantial effert to reimplement all the feaures alredy provided by the library and it would require maintenance effort too.


### Builder Pattern
This pattern is used to generate a table of contents (toc) by initially setting some paramethers and build it when all parameters are setted.
#### Who and where?
In ```tablesofcontents/tableofcontents.go``` file, the function to build the toc is specified as well as the funcions to set paramethers. The ```Transform``` function in ```markup/goldmark/toc.go``` file use the pattern to set the identifiers (line 62), then adds the heading to the table (line 71) and after these operations, it builds the table at line 132.
#### Why?
This pattern allows to build the object one step at time, without use a giant builder at which all the paramethers are provided.
#### A possible alternative
As said, this pattern avoid to user big builder method with a lot of parameters. In this case the parmeters setted are only two so the table of contents could as be build using a "traditional" builer like ```func Build(ids []string, h *Heading, row, level int) {//set params and return the ToC}```
##### Pros:
In this example the params are only a few so they are easy to manage and could be helpful to not distribute the constrution of the table through multiple functions.
##### Cons:
The cons are that you have to prepare all the parameters in and then call the builder after.


### Prototype Pattern
#### Who and where?
TO DO
```func (t *Template) Clone() (*Template, error) ``` in ```tpl\internal\go_templates\template.go``` file...
#### Why?
#### A possible alternative
##### Pros:
##### Cons:


## 3. Summary
Final conclusions