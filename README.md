# Condor Job Sender

`condor-job-sender` is a small Python library for submitting Python scripts, batches of jobs, and dependency-based DAG workflows to an [HTCondor](https://htcondor.org/) cluster.

The package creates HTCondor submit descriptions and executable shell scripts, submits jobs through the HTCondor Python bindings, and uses NetworkX graphs to describe dependencies between DAG jobs.

> [!IMPORTANT]
> Actual job submission requires the HTCondor Python bindings and therefore must run on Linux, such as a cluster login node or WSL. The package can be installed and imported on Windows, but submission functions will report that HTCondor is unavailable.

## Installation before the PyPI release

Until the package is published on PyPI, install the latest version directly from GitHub.

With `pip`:

```bash
pip install "git+https://github.com/MountainMeigui/CondorJobSender.git"
```

With [`uv`](https://docs.astral.sh/uv/):

```bash
uv add "git+https://github.com/MountainMeigui/CondorJobSender.git"
```

After a PyPI release, the installation command will be:

```bash
pip install condor-job-sender
```

The distribution name uses hyphens (`condor-job-sender`), while the Python import name uses underscores (`condor_job_sender`).

## Requirements

- Python 3.10 or newer
- Linux for actual HTCondor submission
- Access to a working HTCondor scheduler from the machine running the package
- A Python script and output directories accessible from the submission machine

## Configuration

Create a file named `configurations.yml` in the project from which you will submit jobs. If you cloned this repository, you can start by copying the included example:

```bash
cp configurations.example.yml configurations.yml
```

PowerShell equivalent:

```powershell
Copy-Item .\configurations.example.yml .\configurations.yml
```

Edit the new file for your cluster. For example:

```yaml
directories:
  python_path_remote: /usr/bin/python3
  std_error_dir: /home/username/condor/error/
  std_output_dir: /home/username/condor/output/
  log_dir: /home/username/condor/log/
  working_dir: /home/username/project/
  condor_executables_dir: /home/username/condor/executables/

username: username

parameters:
  universe: vanilla
```

The current implementation constructs filenames by appending them to the configured directories, so directory values should end with `/`. Make sure the directories already exist and are writable.

Load the configuration before creating or submitting jobs:

```python
from condor_job_sender import configure

configure("configurations.yml")
```

Configuration is loaded explicitly rather than during import, allowing different projects and machines to use different settings.

## Submit one Python job

```python
from condor_job_sender import configure, send_job_to_condor

configure("configurations.yml")

send_job_to_condor(
    py_script_path="/home/username/project/simulation.py",
    job_name="simulation",
    request_cpus="4",
    requestMemory="4096",
    Arguments="10 0.25",
)
```

The package creates a shell executable in `condor_executables_dir`, constructs an HTCondor submit description, and queues the job.

Supported keyword arguments currently include:

| Argument | Meaning |
| --- | --- |
| `Arguments` | Space-separated arguments passed to the Python script |
| `request_cpus` | Number of requested CPU cores |
| `requestMemory` | Requested memory in MB |
| `log` | Optional directory for HTCondor log files |
| `output` | Optional directory for standard-output files |
| `error` | Optional directory for standard-error files |

If `log`, `output`, or `error` is omitted, the corresponding directory from `configurations.yml` is used.

## Submit a batch of jobs

Use `send_batch_of_jobs_to_condor` when the same script should run with multiple parameter sets:

```python
from condor_job_sender import configure, send_batch_of_jobs_to_condor

configure("configurations.yml")

batch_parameters = [
    {"temperature": "0.1", "seed": "1"},
    {"temperature": "0.2", "seed": "2"},
    {"temperature": "0.3", "seed": "3"},
]

send_batch_of_jobs_to_condor(
    py_script_path="/home/username/project/simulation.py",
    job_name="temperature_scan",
    batch_parameters=batch_parameters,
    Arguments="$(temperature) $(seed)",
    request_cpus="1",
    requestMemory="2048",
)
```

Each dictionary supplies values for the corresponding HTCondor variables in `Arguments`.

## Submit a DAG workflow

A directed NetworkX graph can describe dependencies between jobs. An edge from `prepare` to `simulate` means that `prepare` must finish before `simulate` starts.

```python
import networkx as nx

from condor_job_sender import configure, send_dag_job

configure("configurations.yml")

workflow = nx.DiGraph()
workflow.add_edges_from(
    [
        ("prepare", "simulate"),
        ("simulate", "analyze"),
    ]
)

job_information = {
    "prepare": {
        "py_script_path": "/home/username/project/prepare.py",
        "kargs_dict": {
            "request_cpus": "1",
            "requestMemory": "1024",
        },
    },
    "simulate": {
        "py_script_path": "/home/username/project/simulate.py",
        "batch_parameters": [
            {"seed": "1"},
            {"seed": "2"},
        ],
        "kargs_dict": {
            "Arguments": "$(seed)",
            "request_cpus": "4",
            "requestMemory": "4096",
        },
    },
    "analyze": {
        "py_script_path": "/home/username/project/analyze.py",
        "kargs_dict": {
            "request_cpus": "1",
            "requestMemory": "2048",
        },
    },
}

send_dag_job(
    dag_graph=workflow,
    dag_dir_name="example_workflow",
    information_dict=job_information,
)
```

Every graph node must have a matching entry in `job_information`. If `batch_parameters` is omitted, the job runs once.

## Using a configuration file stored elsewhere

The configuration file does not need to be in the current directory. Pass an absolute or relative path:

```python
from condor_job_sender import configure

configure("/home/username/.config/condor-job-sender.yml")
```

## Development

Clone the repository and install its dependencies:

```bash
git clone https://github.com/MountainMeigui/CondorJobSender.git
cd CondorJobSender
uv sync
```

Verify that the package imports:

```bash
uv run python -c "import condor_job_sender; print('Import successful')"
```

Build the wheel and source distribution:

```bash
uv build
```

The generated files will appear under `dist/`.

## Current limitations

- HTCondor's Python bindings are not installed on Windows, so jobs must be submitted from Linux, WSL, or a Linux cluster login node.
- Configured directory paths should end with `/`.
- Paths and individual values in `Arguments` should not contain spaces in the current implementation.
- This package assumes that the configured Python executable, scripts, and generated executable directory are accessible from the HTCondor submission machine.

## Contributing

Bug reports, suggestions, and pull requests are welcome through the [GitHub repository](https://github.com/MountainMeigui/CondorJobSender).
