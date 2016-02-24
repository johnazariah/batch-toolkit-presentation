- title : (Azure) Batch Toolkit
- description : An introduction to the Batch Toolkit
- author : John Azariah
- theme : sky
- transition : default

***

### What is the Batch Toolkit?

- Not a replacement for Batch SDK or Client Library
- User-centric Code Flow
- Idiomatic F# where possible

***

### What does it mean to be user-centric?

- Writing batch-sdk apps is hard!
    - Lots of secret sauce
        - Read/Write model is mixed and inconsistent
        - Have to submit items to the cloud in a given order
        - Cannot read or write *some* properties at various times
    - Focus is on *how* to do something, not *what* to do
    - Limited re-use of existing work
- We need primitives to represent our intent
- We need to design for composition and re-use.

***

### Why F#?

- To build the toolkit
    - Extremely succinct
    - Mathematical foundation for composition
    - Easy to build DSLs
- To use the toolkit
    - Cute DSL

_C# can be used also. Would you like contribute to a C# wrapper?_

***

### Programming Model
' Represents a single command-line executable invocation - can be parametrized
' Represents a command which can recognize runtime failure and optionally handle it
' Represents a set of recoverable commands which can be executed sequentially followed by a set of commands which are executed at the end - to support clean-up or data-upload tasks

    [lang=cs]
    Command : Simple | Parametrized

    CommandWithErrorHandler
    {
        Try : Command
        OnError : Command?
    }

    CommandSet
    {
        MainBlock : List<CommandWithErrorHandler>
        FinalBlock : List<Command>       
    }

***

### Programming Model     
' Represents a template of related work - including a commandset, some local files to upload, and a flag to say if admin privileges are required
' Represents a job - including a set of templates, some local files which are common to all the templates and their instances, and the range of values for each parameter

    [lang=cs]
    WorkloadUnitTemplate
    {
        Commands : CommandSet
        LocalFiles : List<FileInfo>
        RunAsAdmin : bool
    }

    WorkloadArguments : Dictionary<string, List<string>>

    WorkloadSpecification
    {
        UnitTemplates : List<WorkloadUnitTemplate>
        LocalFiles : List<FileInfo>
        Arguments : WorkloadArguments
    }

***

### Programming Model
' Represents a single instance of a WorkloadSpecificationtemplate, with associated arguments (one for each parameter), and the files to be uploaded for the task  

    [lang=cs]    
    TaskArguments : Dictionary<string, string>

    TaskSpecification
    {
        // *Unified Schema* for CloudTask/JobPreparationTask/JobManagerTask...
        // Closure of all the settable properties of xxxTask variants
        ...

        TaskCommandSet : CommandSet
        TaskArguments : TaskArguments
        TaskLocalFiles : LocalFiles
    }

***

### Programming Model

' Represents a job - can be constructed as an in-memory object and associated with a set of task specifications
' Also allows for modelling files that are used by all the task instances

    [lang=cs]
    JobSpecification
    {
        // All the settable cloudJob properties
        ...

        JobManagerTask : TaskSpecification?
        JobPreparationTask : TaskSpecification?
        JobReleaseTask : TaskSpecification?

        JobTasks : TaskSpecification list
        JobSharedLocalFiles : LocalFiles        
    }


***
### Programming Model

* Unified Pool Specification
    * Named Pool
    * Auto Pool

***
### Toolkit Supports

Composing Primitives (Monoidally)

    [lang=cs]
    CommandSet ⊕ CommandSet ⊕ ...
    LocalFiles ⊕ LocalFiles ⊕ ...
    UploadedFiles ⊕ UploadedFiles ⊕ ...
    WorkloadUnitTemplate ⊕ WorkloadUnitTemplate ⊕ ...
    WorkloadArguments ⊕ WorkloadArguments ⊕ ...
    WorkloadSpecification ⊕ WorkloadSpecification ⊕ ...
    TaskArguments ⊕ TaskArguments ⊕ ...

***
### Toolkit Supports

* Parametric Sweep
    - Generate all combinations of parameter-value pairs
    - Construct a task for each set of parameter-values

* Shared Files for Job
    - Associated SharedFiles with JobPreparationTask (creating one if necessary)
    - Introducing a FILE_COPY command in each Task's CommandSet so these files are copied from the JobPrepTask folder to the Task folder

***
### Job Submission workflow
    [lang=haskell]
    Workload -> JobSpecification
        Generate TaskSpecification for each parameter set

    JobSpecification -> submitJob    
        For each TaskSpecification
            Generate script file for command-set;
            Append script file to the LocalFiles collection
            Set the CommandLine of the task to execute script file
        For each LocalFile
            Upload the file
            Obtain SAS and associate with Task
        Ensure the NamedPool exists (if not Auto)
        Submit All Tasks
        Submit Job            

***
### Development workflow
    [lang=haskell]
    Define set of commands, files, arguments
    Build a Workload

    Submit Workload
        Workflow gets converted to  

---
### DSL example
    [lang=haskell]
    let CopyOutputToAzure = ``:cmd`` "echo 'Copying output to Azure'"
    let SayHelloWorld = ``:cmd`` "echo 'Hello, world!'"
    let SayGoodbye = ``:cmd`` "echo 'Goodbye'"

---    
    [lang=haskell]
    let workload =
        ``:workload``
        ``:units``
            [
                (``:unit``
                    ``:admin`` true
                    ``:main``
                        [
                            ``:do`` SayHelloWorld
                            ``:try``
                                (``:cmd_with_params`` "echo 'Hello %user%'" ``:params`` ["%user%"])
                                ``:with`` ``:exit``
                        ]
                    ``:finally``
                        [
                            CopyOutputToAzure
                            SayGoodbye
                        ]
                    ``:files`` [])
            ]
        ``:files``
            []
        ``:arguments``
            [
                ``:range`` "%user%" ``:over`` ["John"; "Ivan"; "Pablo"]
            ]
---
    [lang=haskell]
    workload
    |> WorkloadOperations.SubmitWorkloadToPoolAsync batchConfiguration storageConfiguration pool
    |> Async.AwaitTask

***
### Where is it?

* Github : https://github.com/johnazariah/batch-toolkit
