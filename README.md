# Quick Start

Running many experiments while minimizing idle GPU time requires a lot of manual efforts.
This light-weight tool helps you currently run a **LOT** of experiments with simple commands and configurations.

## Install

```
pip install mlrunner
```

dependencies:

```
python >= 3.7
pyyaml
```

## Usage

Edit `params.yaml` and then hit `run`.

See `run -h` for available command-line args. See comments in `params.yaml` for available of configurations.

For typical use cases see `examples`.

## Example

Suppose we develop a new normalization layer "newnorm" and want to compare it to batchnorm. Both have a
hyperparameter `--moment`. We also want to see how early stop affects our model, which is
specified by a boolean flag `--early-stop`. Each run involves training, checkpoint average and test with the averaged
checkpoint. The `params.yaml` can be specified as:

```yaml
---
# All commands for each experiment with params to be filled specified as `{param}` or `[param]`
# `{_output}` is a reserved param for the automatically generated output directory
template:
  train: >
    python train.py data-bin/{data} --save-dir {_output} --norm {norm} [moment] [early-stop]

  avg: >
    python checkpoint_avg.py --inputs {_output} --num 5 --output {_output}/avg.pt

  test: >
    python generate.py data-bin/{data} --beam 5 --path {_output}/avg.pt

# default values for all params
default:
  data: iwslt14
  norm: batch
  moment: 0.1
  early-stop: False

# GPU indices to be filled in `CUDA_VISIBLE_DEVICES={}`, each corresponds to a worker.
resource: [ 0, 1, 2, 3 ]

---
# compare the effect of different normalization layer and moment 
norm: [ new, batch ]
moment: [ 0.1, 0.05 ]

---
# examine the effect of early stopping
norm: [ batch ]
early-stop: [ True, False ]

```

We simply hit `run` for the `6 = 2 * 2 + 2` experiments. Since  `norm=batch,moment=0.1` in the second yaml doc and `norm=batch,early-stop=False` in the third doc share the same params, the latter is skipped. As we specify 4 workers each with only one
gpu, there are 4 tasks running concurrently:

```
$ run
Orphan params: set()
Tasks: 5, Commands: 15
START   gpu: 0, train: 1/ 4, output/Norm_new-Moment_0.1
START   gpu: 1, train: 2/ 4, output/Norm_new-Moment_0.05
START   gpu: 2, train: 3/ 4, output/Norm_batch-Moment_0.1
START   gpu: 3, train: 4/ 4, output/Norm_batch-Moment_0.05
START   gpu: 0, avg  : 1/ 4, output/Norm_new-Moment_0.1
FAIL    gpu: 0, avg  : 1/ 4, output/Norm_new-Moment_0.1
...
```

The logs are redirected to directories (referred with `{_output}`) of each experiment (named with parameters):

```
$ ls output/Norm_batch-Moment_0.1
checkpoint51.pt
checkpoint52.pt
averaged_model.pt
log.train.20220316.030151
log.avg.20220316.030151
log.test.20220316.030151
param
stat
```

We provide `Examiner` as a container to iteratively apply a metric parser to all experiments and aggregate the results. In this example we simply parse the test log for the test BLEU:

```python
from runner.examine import Examiner, latest_log


# define a metric parser for each directory (experiment)
def add_bleu(output_dir, experiment, caches):
    # Each parser follows the same signature
    # It can read/write to a global cache dict `caches`, 
    # and read/write each experiment: 
    # collections.namedtuple("Experiment", ["cache", "metric", "param"])
    latest_test_log = latest_log("test", output_dir)
    bleu = parse_bleu(latest_test_log)  # a user-defined log parser
    experiment.metric["bleu"] = bleu


examiner = Examiner()  # container for parsed results
# register parser for each directory (experiment)
examiner.add(add_bleu)
# run all parsers for directories matched by regex 
examiner.exam(output="output", regex=".*")
# print the tsv table with all (different) params and metrics of each experiment
examiner.table()
```
which results in
```commandline
norm	moment	early-stop	bleu
new	0.1	FALSE	11.0
new	0.05	FALSE	12.3
batch	0.1	FALSE	14.4
batch	0.05	FALSE	16.5
batch	0.1	TRUE	15.0
```
The table can be readily copied to excel or numbers for further analysis. 

# Under the Hood
The tool edits each command template with the following steps:

1. Substitute the param placeholders (`{param}` and `[param]`) in the templates of the first doc with a sweep of param combinations specified in later docs
2. Append shell environment variable `CUDA_VISIBLE_DEVICES={resource}` as the prefix
3. Append shell redirect `> output_dir/log.{command}.{time} 2>&1` as the suffix

# Workflow

Manually scheduling a **LOT** of experiments can quickly lead to frustrations:

1. Efficiency. During early phase we experiment on small models and datasets which are not resource hungry. One can find it hard to fully utilize the gpu times on modern multi-gpu machines.
2. Cognitive load. There are lengthy pipelines and numerous parameters to tune: data, model architecture, hyperparams, training regimes and test regimes. These knots are typically scatter in code, data or command-line args, making the experiment process error-prone and cognitively draining.
3. Accessibility. How to distinguish artifacts of different experiments in the file system while maintaining readability? How to quickly obtain insights from tens of hundreds of results? How to quickly set up the workflow for new projects?
4. Robustness: What if your machine is temporally down or some bug happened in your code? Which experiment needs rerun?

This tool tightly integrates into a more effective workflow. In a nutshell:

1. Make every modification (architecture, params, training regime, etc.) adjustable by command line args. This
   interface is consistent to most code base.
    1. For structural change of models, use if/else or switch/case
    2. For datasets, specify the directory
2. Specify irrelevant params in the command template. Make relevant params to the experiment variables (`[param]` or `{param}`) and list values you want to test in a configuration file. Specify default values of these params for reference.
3. Use a pool of workers to concurrently run all your experiments. Track progress with tools like tensorboard.
4. Apply the same processing code for each run to parse results you need, and aggregate them for visualization: use tensorboard hyperparams, jupyter notebook or simply a spreadsheet.

