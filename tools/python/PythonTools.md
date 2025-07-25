# Python Helpers

## Authentication method for Azure DevOps

Generally speaking Azure DevOps services has three ways of doing authentication:

1. Microsoft Entra ID(with an Entra token)
2. Personal Access Tokens(PATs)
3. Azure DevOps OAuth

In this project all tools should only use the first one. An Entra token is very long compared to PATs. There are two ways of getting such a token:

1. Use a service connection with Workload Identity Federation, which is for scripts that run in pipelines in Azure cloud.
2. Get it from AzCli, which is for scripts that runs on Microsoft developers machines.  

Some tools need to support the both scenarios, for example, nuget.exe. We are still exploring ways to make it work well with Entra ID auth(need to find a way to support cross-org nuget publishing).

## delete_ado_pipeline.py
Prerequisites: Install AzCli and get logged in.

This script is designed to completely delete an Azure DevOps pipeline definition and all of its associated data.

The process is as follows:
1. Asynchronously finds and deletes ALL retention leases associated with the pipeline.
2. Asynchronously deletes ALL build runs (history) for the pipeline.
3. Deletes the pipeline definition itself.


## upload_symbol.py

This script downloads ONNX Runtime Windows release artifacts for a specified version from GitHub. Extracts PDB symbol files and uploads them to your Azure DevOps symbol server, simplifying debugging for teams hosting symbols there.

This script requires symbols.exe, which can be downloaded by using the following Azure REST API: https://learn.microsoft.com/en-us/rest/api/azure/devops/symbol/client/get?view=azure-devops-rest-7.1

symbols.exe handles authentication.

## run_packaging_pipelines.py

The script is for triggering Azure DevOps pipelines for a specific Github pull request or git branch. ONNX Runtime's release managers should use this script to trigger packaging pipelines.
You may also use this script to trigger pull request pipelines for external PRs. 

### Prerequisites

1. Install AzCli and get logged in.
2. pip install pyyaml

### Usages

It supports two modes:

1. CI Build Mode (Default):
   - Triggers pipelines in the 'Lotus' project.
   - Filters pipelines based on the following criteria:
     - Repository Association: Must be 'https://github.com/microsoft/onnxruntime'.
     - Recent Activity: Must have run in the last 30 days.
     - Pipeline Type: Must be YAML-based.
     - Trigger Type: Must NOT be triggered by another pipeline resource.
     - Template Requirement: Must extend from 'v1/1ES.Official.PipelineTemplate.yml@1esPipelines'.
2. Pull Request (PR) Mode:
   - Activated by using the '--pr <ID>' argument.
   - Triggers pipelines in the 'PublicPackages' project.
   - Filters pipelines based on a simplified criteria:
     - Repository Association: Must be 'https://github.com/microsoft/onnxruntime'.
     - Recent Activity: Must have run in the last 30 days.
     - Pipeline Type: Must be YAML-based.

The script also includes a feature to cancel any currently running builds for a matching
pipeline on the target branch/PR before queuing a new one.

## ort_test_dir_utils.py

Provides helpers for creating ONNX test directories that can be run using onnx_test_runner and onnxruntime_perf_test.

In order to import ort_test_dir_utils you need to either
  - run python from the `<onnxruntime root dir`>/tools/python directory
  - add the directory to your PYTHONPATH
  - add the directory to sys.path prior to importing

e.g. to add to sys.path
```python
import sys
sys.path.append('<onnxruntime root dir>/tools/python')

import ort_test_dir_utils
```

### Creating a test directory for a model.

The create_test_dir helper can create the input and output pb files in various ways.

Often a support request will only provide a problematic model and no input data. create_test_dir can be used to create input to allow the model to be debugged more easily. Random input can be generated if not provided. If expected output is not provided, the model will be run with the input, and the output from that will be saved as the expected output.

```python
def create_test_dir(model_path, root_path, test_name,
                    name_input_map=None, symbolic_dim_values_map=None,
                    name_output_map=None):
    """
    Create a test directory that can be used with onnx_test_runner, onnxruntime_perf_test.
    Generates random input data for any missing inputs.
    Saves output from running the model if name_output_map is not provided.

    :param model_path: Path to the onnx model file to use.
    :param root_path: Root path to create the test directory in.
    :param test_name: Name for test. Will be added to the root_path to create the test directory name.
    :param name_input_map: Map of input names to numpy ndarray data for each input.
    :param symbolic_dim_values_map: Map of symbolic dimension names to values to use for the input data if creating
                                    using random data.
    :param name_output_map: Optional map of output names to numpy ndarray expected output data.
                            If not provided, the model will be run with the input to generate output data to save.
    :return: None
    """
```

Example usage:

```python
import sys
import ort_test_dir_utils
import onnx_test_data_utils

# example model with two float32 inputs called 'input1' (dims: {2, 1}) and 'input2' (dims: {'dynamic', 4})
model_path = '<onnxruntime root dir>/onnxruntime/test/testdata/transform/expand_elimination.onnx'

# when using the default data generation any symbolic dimension values must be provided
symbolic_vals = {'dynamic':2} # provide value for symbolic dim named 'dynamic' in 'input2'

# let create_test_dir create random input in the (arbitrary) default range of -10 to 10.
# it will create data of the correct type based on the model.
ort_test_dir_utils.create_test_dir(model_path, 'temp/examples', 'test1', symbolic_dim_values_map=symbolic_vals)

# alternatively some or all input can be provided directly. any missing inputs will have random data generated.
# symbolic dimension values are only required for input data that is randomly generated,
# so we don't need to provide that in this case as we're explicitly providing all inputs.
inputs = {'input1': np.array([100, 200]).reshape((2,1)).astype(np.float32),
          'input2': np.random.randn(2,4).astype(np.float32)} # use 2 for the 'dynamic' dimension so shape is {2, 4}

ort_test_dir_utils.create_test_dir(model_path, 'temp/examples', 'test2', name_input_map=inputs)

# can easily dump the input and output to visually check it's as expected
onnx_test_data_utils.dump_pb('temp/examples/test2/test_data_set_0')
```

### Running the test using python

To execute the test once the directory is created you can use the onnx_test_runner or onnxruntime_perf_test executables if you have built onnxruntime from source, or the run_test_dir helper. Input can be either the test directory, or the model in case there are multiple in the test directory.

```python
def run_test_dir(model_or_dir):
    """
    Run the test/s from a directory in ONNX test format.
    All subdirectories with a prefix of 'test' are considered test input for one test run.

    :param model_or_dir: Path to onnx model in test directory,
                         or the test directory name if the directory only contains one .onnx model.
    :return: None
    """
```

Example usage:

```python
import sys
import ort_test_dir_utils

try:
    ort_test_dir_utils.run_test_dir('temp/examples/test1')
    ort_test_dir_utils.run_test_dir('temp/examples/test2/expand_elimination.onnx')
except Exception:
    print("Exception:", sys.exc_info()[1])
```

## onnx_test_data_utils.py

Provides helpers for generating/reading protobuf files containing ONNX TensorProto data.

```
usage: onnx_test_data_utils.py [-h] --action {dump_pb,numpy_to_pb,image_to_pb,random_to_pb,update_name_in_pb}
                               [--input INPUT] [--name NAME] [--output OUTPUT] [--resize RESIZE] [--channels_last] [--add_batch_dim]
                               [--shape SHAPE] [--datatype DATATYPE] [--min_value MIN_VALUE] [--max_value MAX_VALUE] [--seed SEED]

        Utilities for working with the input/output protobuf files used by the ONNX test cases and onnx_test_runner.
        These are expected to only contain a serialized TensorProto.

        dump_pb: Dumps the TensorProto data from an individual pb file, or all pb files in a directory.
        numpy_to_pb: Convert numpy array saved to a file with numpy.save() to a TensorProto, and serialize to a pb file.
        image_to_pb: Convert data from an image file into a TensorProto, and serialize to a pb file.
        random_to_pb: Create a TensorProto with random data, and serialize to a pb file.
        update_name_in_pb: Update the TensorProto.name value in a pb file.
                           Updates the input file unless --output <filename> is specified.


optional arguments:
  -h, --help            show this help message and exit
  --action {dump_pb,numpy_to_pb,image_to_pb,random_to_pb,update_name_in_pb}
                        Action to perform
  --input INPUT         The input filename or directory name
  --name NAME           The value to set TensorProto.name to if creating/updating one.
  --output OUTPUT       Filename to serialize the TensorProto to.

image_to_pb:
  image_to_pb specific options

  --resize RESIZE       Provide the height and width to resize to as comma separated values. e.g. --shape 200,300 will resize to height 200 and width 300.
  --channels_last       Transpose image from channels first to channels last.
  --add_batch_dim       Prepend a batch dimension with value of 1 to the shape. i.e. convert from CHW to NCHW

random_to_pb:
  random_to_pb specific options

  --shape SHAPE         Provide the shape as comma separated values e.g. --shape 200,200
  --datatype DATATYPE   numpy dtype value for the data type. e.g. f4=float32, i8=int64. See: https://docs.scipy.org/doc/numpy/reference/arrays.dtypes.html
  --min_value MIN_VALUE
                        Limit the generated values to this minimum.
  --max_value MAX_VALUE
                        Limit the generated values to this maximum.
  --seed SEED           seed to use for the random values so they're deterministic.
```

## dump_subgraphs.py

If you're investigating a model with control flow nodes (Scan/Loop/If) the subgraphs won't be displayed in Netron. Run dump_subgraphs to dump the subgraphs as .onnx files that can be viewed individually.

```
usage: dump_subgraphs.py [-h] -m MODEL [-o OUT]

Dump all subgraphs from an ONNX model into separate onnx files.

optional arguments:
  -h, --help                show this help message and exit
  -m MODEL, --model MODEL   model file
  -o OUT, --out OUT         output directory (default: <current dire)
```

## onnx2tfevents.py

Convert ONNX model to TensorBoard events file so that we can visualize the model in TensorBoard. This is especially useful for debugging large models that cannot be visualized in Netron.

```
usage: onnx2tfevents.py [-h] [--logdir LOGDIR] [--model MODEL]

optional arguments:
  -h, --help       show this help message and exit
  --logdir LOGDIR  Tensorboard log directory
  --model MODEL    ONNX model path
```

Once the events file is generated, start the tensorboard and visualize the model graph.

Note: This tool requires torch and tensorboard to be installed.
