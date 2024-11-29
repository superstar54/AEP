# AEP 010: Introducing the `PythonJob` in AiiDA

| AEP number | 010                                                          |
|------------|--------------------------------------------------------------|
| Title      | Introducing the `PythonJob` in AiiDA                         |
| Authors    | [Xing Wang](mailto:xingwang1991@gmail.com) (superstar54)     |
| Champions  | [Xing Wang](mailto:xingwang1991@gmail.com) (superstar54)     |
| Type       | S - Standard Track AEP                                       |
| Created    | 28-Nov-2024                                                  |
| Status     | Draft                                                        |

## Background

AiiDA is a powerful tool for managing complex computational workflows and ensuring data provenance. However, despite its capabilities, AiiDA is sometimes perceived as having a steep initial learning curve, especially for new users or those who have existing computational workflows developed outside of AiiDA. This perception is primarily because users are required to develop AiiDA plugins to execute their codes on remote computers. This process involves creating input generators, output parsers, and configuring codes for execution within the AiiDA framework, which can be time-consuming and requires maintaining additional codebases.

In many computational science communities, software packages like ASE (Atomic Simulation Environment), Pymatgen, and others are widely used and already provide APIs to manage inputs, execute computations, and parse outputs. Requiring developers to adapt their existing packages to meet AiiDA-specific requirements introduces significant overhead, including the need to maintain dual codebases and potentially duplicate functionality. Moreover, AiiDA's emphasis on data provenance demands that data be transformed to fit its unique database format, which can differ from formats used by other packages.

These challenges can hinder the adoption of AiiDA, particularly among researchers who may not have the resources or expertise to develop custom plugins or who wish to leverage their existing codebases without significant modification.

## Proposed Enhancement

Introduce a new component called `PythonJob` in AiiDA that allows users to deploy their existing Python functions and package APIs to operate jobs on remote computers seamlessly. Users can write standard Python functions using their preferred packages, and `PythonJob` will manage the execution on remote computers, handle data serialization and deserialization, manage data transformations, and ensure comprehensive data provenance within the AiiDA framework.
### Key Features

- **Seamless Integration**: Users can directly use existing Python functions and packages without modification.
- **Remote Execution Management**: `PythonJob` handles data transfer, environment setup, and execution on remote computers.
- **Automatic Data Serialization**: Inputs and outputs are automatically serialized and deserialized, integrating with AiiDA's data types.
- **Data Provenance Tracking**: Full provenance of the computation is recorded in AiiDA's graph, ensuring reproducibility.
- **Flexibility**: Supports custom outputs, dynamic outputs (namespaces), handling of files and folders, and exit codes.


## Implementation Details

### Execution Mechanism

- **Function Serialization**: The user-defined Python function is inspected and the source code transferred to the remote computer, including any necessary lines to import required packages.
- **Environment Management**: In the `metadata`, one activate the environment, load modules.
- **Remote Execution**: On the remote machine, the function is executed within the prepared environment. A runner script deserializes inputs, executes the function, and serializes outputs.
- **Data Transfer**: Input data is transferred to the remote machine before execution, and output data is retrieved after execution completes.

### User Interface

- **Input Preparation**: Users use a helper function `prepare_pythonjob_inputs` to package the function, its inputs (`function_inputs`), and other setting parameters (e.g., `computer`, `parent_folder`, `upload_files`, `function_outputs`, and `metadata` etc.).
- **Execution Functions**: Users can execute the `PythonJob` using AiiDA's standard execution functions, such as `run`, `run_get_node`, or `submit`.

### Data Serialization and Deserialization

`PythonJob` searches for data serializers from the `aiida.data` entry point using the module and class names of the data types (e.g., `ase.atoms.Atoms`). To let `PythonJob` find the serializer, users must register the AiiDA data type in their plugin.

```ini
[project.entry-points."aiida.data"]
myplugin.ase.atoms.Atoms = "myplugin.data:AtomsData"
```

This registers a data serializer for `ase.atoms.Atoms` data. Standard Python types (e.g., `int`, `float`, `str`, `list`, `dict`) are automatically serialized and deserialized.

#### Avoiding Duplicate Serializers

If multiple plugins register serializers for the same data type, `PythonJob` may raise an error due to ambiguity. To resolve this, users can specify which serializer to use in a configuration file (e.g., `pythonjob.json` in the AiiDA configuration directory):

```json
{
    "serializers": {
        "ase.atoms.Atoms": "myplugin.ase.atoms.Atoms"
    }
}
```

### Handling Files and Folders

- **Upload Files or Folders**: Users can specify files or folders to be uploaded to the remote working directory using the `upload_files` parameter.
- **parent_folder**: Users can specify a parent folder to access files from a previous task, allowing data reuse and chaining of tasks.
- **Retrieve Additional Files**: Users can specify additional files to retrieve from the remote machine after execution using standard `CalcJob` parameter:  `metadata['options']['additional_retrieve_list']`.

### Exit Codes

If the function returns a dictionary containing an `exit_code`, `PythonJob` uses this to set the exit status and message of the process, allowing custom error handling.

Here is an example of a function that handles division by zero and returns an exit code:

```python
def safe_divide(x, y):
    if y == 0:
        return {"result": None, "exit_code": {"status": 400, "message": "Division by zero"}}
    return {"result": x / y}

inputs = prepare_pythonjob_inputs(
    safe_divide,
    function_inputs={"x": 10, "y": 0},
)
result, node = run_get_node(PythonJob, inputs=inputs)
print("Exit status:", node.exit_status)
print("Exit message:", node.exit_message)
```



#### Namespace Outputs
The PythonJob allows users to define namespace outputs. A namespace output is a dictionary with keys and values returned by a function. Each value in this dictionary will be serialized to AiiDA data, and the key-value pair will be stored in the database. Why use namespace outputs?

- Dynamic and Flexible: The keys and values in the namespace output are not fixed and can change based on the task’s execution.
- Querying: The data in the namespace output is stored as an AiiDA data node, allowing for easy querying and retrieval.
- Data Provenance: When the data is used as input for subsequent tasks, the origin of data is tracked.

```python
from ase.build import bulk

def generate_scaled_structures(structure, factors):
    scaled_structures = {}
    for i, factor in enumerate(factors):
        scaled = structure.copy()
        scaled.set_cell(scaled.cell * factor, scale_atoms=True)
        scaled_structures[f"scaled_{i}"] = scaled
    return {"scaled_structures": scaled_structures}

inputs = prepare_pythonjob_inputs(
    generate_scaled_structures,
    function_inputs={"structure": bulk("Si"), "factors": [0.95, 1.0, 1.05]},
    function_outputs=[{"name": "scaled_structures", "identifier": "namespace"}],
)
result, node = run_get_node(PythonJob, inputs=inputs)
for name, structure in result["scaled_structures"].items():
    print(f"{name}: {structure}")
```


### Environment and Dependency Management

- **Remote Environment Setup**: `PythonJob` can set up the required Python environment on the remote machine by activating virtual environments or using environment modules.

```python
from aiida_pythonjob.utils import create_conda_env
# create a conda environment on remote computer
create_conda_env(
       "merlin6",                # Remote computer
       "test_pythonjob",         # Name of the conda environment
       modules=["anaconda"],     # Modules to load (e.g., Anaconda)
       pip=["numpy", "matplotlib"],  # Python packages to install via pip
       conda={                   # Conda-specific settings
           "channels": ["conda-forge"],  # Channels to use
           "dependencies": ["qe"]       # Conda packages to install
       }
   )
```

### Usage Examples


A simple example of using `PythonJob` to add two numbers:

```python
from aiida.engine import run_get_node
from aiida_pythonjob import PythonJob, prepare_pythonjob_inputs

def add(x, y):
    return x + y

inputs = prepare_pythonjob_inputs(
    add,
    function_inputs={"x": 1, "y": 2},
    computer="localhost",
)
result, node = run_get_node(PythonJob, inputs=inputs)
print("result: ", result["result"])
```

### Design Considerations

#### Integration with AiiDA's Architecture

- **Reusing Existing Infrastructure**: `PythonJob` leverages AiiDA's `CalcJob` infrastructure, including transport, scheduling, and data management.
- **Compatibility with Workflows**: Can be used within AiiDA `WorkChains`, allowing composition of complex workflows.

#### Limitations and Constraints

- **Performance Overhead**: Serialization of the function and data may introduce overhead compared to native code execution.
- **Environment Management**: Environment setup may introduce overhead compared to optimized native code.

#### Future Extensions

- **Community Contributed Serializers**: Development of a registry of serializers for broader data type support.

## Pros and Cons

### Pros

- **Lower Entry Barrier**: Simplifies remote execution of computations, making AiiDA more accessible to new users and promoting wider adoption.
- **Leverage Existing Code**: Allows users to utilize their existing Python code and packages directly, reducing duplication and maintenance effort.
- **Maintain Data Provenance**: Ensures that all computations are fully tracked within AiiDA's provenance graph, supporting reproducibility.
- **Broad Accessibility**: Makes AiiDA appealing to a wider range of scientific and engineering communities that rely on Python-based tools.
- **Flexibility**: Users can execute a wide variety of tasks without developing specific plugins.

### Cons



## Conclusion

The introduction of `PythonJob` represents a significant advancement in making AiiDA more user-friendly and accessible. By allowing users to execute standard Python functions on remote computers while maintaining full data provenance, `PythonJob` reduces the initial setup overhead and encourages broader adoption of AiiDA across diverse computational science and engineering communities. It strikes a balance between flexibility and integration, empowering users to leverage their existing tools within AiiDA's robust workflow and data management framework.