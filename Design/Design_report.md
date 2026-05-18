# Hugo - Design Report

## 1. Dependencies

### 1.1 Intro
Methodology and Tools

### 1.2 Code Dependencies
Code dependencies: based on imports in source code
• Which files have most or least dependencies? Why?

### 1.3 Knowledge Dependencies 
Knowledge dependencies: based on co-change (how often two files are
changed together in the same commit)
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