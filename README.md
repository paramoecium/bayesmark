# Installation

This Bayesian optimization (BO) benchmark framework requires a few easy steps for setup. It can be run either on a local machine (in serial) or prepare a *commands file* to run on a cluster as parallel experiments (dry run mode).

The following instructions have been tested with Python 3.6.8 on Ubuntu (16.04.5 LTS).

The core package itself can be installed with:

```
pip install bo-benchmark
```
However, to also require installation of all the "built in" optimizers for evaluation, run:

```
pip install bo-benchmark[optimizers]
```
It is also possible to use the same pinned dependencies we used in testing by [installing from the repo](#install-in-editable-mode).

Building an environment to run the included notebooks can be done with:

```
pip install bo-benchmark[notebooks]
```
Or, `bo-benchmark[optimizers,notebooks]` can be used.

## Non-pip dependencies

To be able to install `opentuner` some system level (non-pip) dependencies must be installed. This can be done with:

```
sudo apt-get install libsqlite3-0
sudo apt-get install libsqlite3-dev
```
On Ubuntu, this results in:

```
ii  libsqlite3-0:amd64    3.11.0-1ubuntu1  amd64  SQLite 3 shared library
ii  libsqlite3-dev:amd64  3.11.0-1ubuntu1  amd64  SQLite 3 development files
```
On Ubuntu, one can run `dpkg -l | grep libsqlite` to see which `libsqlite` packages are installed.

The environment should now all be setup to run the BO benchmark.

# Running

Now we can run each step of the experiments. First, we run all combinations and then run some quick commands to analyze the output.

## Launch the experiments

The experiments are run using the experiment launcher, which has the following interface:

```
usage: bob-launch [-h] [-dir DB_ROOT] [-odir OPTIMIZER_ROOT] [-v] [-u UUID]
                  [-dr DATA_ROOT] [-b DB] [-o OPTIMIZER [OPTIMIZER ...]]
                  [-d DATA [DATA ...]]
                  [-c [{DT,MLP-adam,MLP-sgd,RF,SVM,ada,kNN,lasso,linear} ...]]
                  [-m [{acc,mae,mse,nll} ...]] [-n N_CALLS]
                  [-p N_SUGGEST] [-r N_REPEAT] [-nj N_JOBS] [-ofile JOBS_FILE]
```
The arguments are:

```
  -h, --help            show this help message and exit
  -dir DB_ROOT, -db-root DB_ROOT
                        root directory for all benchmark experiments output
  -odir OPTIMIZER_ROOT, --opt-root OPTIMIZER_ROOT
                        Directory with optimization wrappers
  -v, --verbose         print the study logs to console
  -u UUID, --uuid UUID  length 32 hex UUID for this experiment
  -dr DATA_ROOT, --data-root DATA_ROOT
                        root directory for all custom csv files
  -b DB, --db DB        database ID of this benchmark experiment
  -o OPTIMIZER [OPTIMIZER ...], --opt OPTIMIZER [OPTIMIZER ...]
                        optimizers to use
  -d DATA [DATA ...], --data DATA [DATA ...]
                        data sets to use
  -c, --classifier [{DT,MLP-adam,MLP-sgd,RF,SVM,ada,kNN,lasso,linear} ...]
                        classifiers to use
  -m, --metric [{acc,mae,mse,nll} ...]
                        scoring metric to use
  -n N_CALLS, --calls N_CALLS
                        number of function evaluations
  -p N_SUGGEST, --suggestions N_SUGGEST
                        number of suggestions to provide in parallel
  -r N_REPEAT, --repeat N_REPEAT
                        number of repetitions of each study
  -nj N_JOBS, --num-jobs N_JOBS
                        number of jobs to put in the dry run file, the default
                        0 value disables dry run (real run)
  -ofile JOBS_FILE, --jobs-file JOBS_FILE
                        a jobs file with all commands to be run
```
The output files will be placed in `[DB_ROOT]/[DBID]`. If `DBID` is not specified, it will be a randomly created subdirectory with a new name to avoid overwriting previous experiments. The path to `DBID` is shown at the beginning of `stdout` when running `bob-launch`. In general, let the launcher create and setup `DBID` unless you are appending to a previous experiment, in which case, specify the existing `DBID`.

### Selecting the experiments

A list of optimizers, classifiers, data sets, and metrics can be listed using the `-o`/`-c`/`-d`/`-m` commands, respectively. If not specified, the program launches all possible options.

### Selecting the optimizer

The data argument `-o` allows a list containing the "built-in" optimizers:

```
"HyperOpt", "Nevergrad-OnePlusOne", "OpenTuner-BanditA", "OpenTuner-GA", "OpenTuner-GA-DE", "PySOT", "RandomSearch", "Scikit-GBRT-Hedge", "Scikit-GP-Hedge", "Scikit-GP-LCB"
```
or, one can specify a user-defined optimizer. The class containing an optimizer conforming to the API must be found in in the folder specified by `--opt-root`. Additionally, a configuration defining each optimizer must be defined in `[OPT_ROOT]/config.json`. The `--opt-root` and `config.json` may be omitted if only built-in optimizers are used.

Additional details for providing a new optimizer are found in [adding a new optimizer](#adding-a-new-optimizer).

### Selecting the data set

The data argument `-d` allows a list containing the "built-in" data sets:

```
"breast", "digits", "iris", "wine", "boston", "diabetes"
```
or, it can refer to a custom `csv` file, which is the name of file in the folder specified by `--data-root`. It also follows the convention that regression data sets start with `reg-` and classification data sets start with `clf-`. For example, the classification data set in `[DATA_ROOT]/clf-foo.csv` is specified with `-d clf-foo`.

The `csv` file can be anything readable by pandas, but we assume the final column is the target and all other columns are features. The target column should be integer for classification data and float for regression. The features should float (or `str` for categorical variable columns). See `bo_benchmark.data.load_data` for more information.

### Dry run for cluster jobs

It is also possible to do a "dry run" of the launcher by specifying a value for `--num-jobs` greater than zero. For example, if `--num-jobs 50` is provided, a text file listing 50 commands to run is produced, with one command (job) per line. This is useful when preparing a list of commands to run later on a cluster.

### The UUID argument

The `UUID` is a 32-char hex string used as a master random seed which we use to draw random seeds for the experiments. If `UUID` is not specified a version 4 UUID is generated. The used UUID is displayed at the beginning of `stdout`. In general, the `UUID` should not specified/re-used except for debugging because it violates the assumption that the experiment UUIDs are unique.

## Aggregate results

Next to aggregate all the experiment files into combined (json) files we need to run the aggregation command:

```
usage: bob-agg [-h] [-dir DB_ROOT] [-odir OPTIMIZER_ROOT] [-v] -b DB [-rv]
```
The arguments are:

```
  -h, --help            show this help message and exit
  -dir DB_ROOT, -db-root DB_ROOT
                        root directory for all benchmark experiments output
  -odir OPTIMIZER_ROOT, --opt-root OPTIMIZER_ROOT
                        Directory with optimization wrappers
  -v, --verbose         print the study logs to console
  -b DB, --db DB        database ID of this benchmark experiment
  -rv, --ravel          ravel all studies to store batch suggestions as if
                        they were serial
```
The `DB_ROOT` must match the folder from the launcher `bob-launch`, and `DBID` must match that displayed from the launcher as well. The aggregate files are found in `[DB_ROOT]/[DBID]/derived`.

## Analyze and summarize results

Finally, to run a statistical analysis presenting a summary of the experiments we run

```
usage: bob-anal [-h] [-dir DB_ROOT] [-odir OPTIMIZER_ROOT] [-v] -b DB
```
The arguments are:

```
  -h, --help            show this help message and exit
  -dir DB_ROOT, -db-root DB_ROOT
                        root directory for all benchmark experiments output
  -odir OPTIMIZER_ROOT, --opt-root OPTIMIZER_ROOT
                        Directory with optimization wrappers
  -v, --verbose         print the study logs to console
  -b DB, --db DB        database ID of this benchmark experiment
```
The `DB_ROOT` must match the folder from the launcher `bob-launch`, and `DBID` must match that displayed from the launcher as well. The aggregate files are found in `[DB_ROOT]/[DBID]/derived`.

The `bob-anal` command looks for a `baseline.json` file in `[DB_ROOT]/[DBID]/derived`, which states the best possible and random search performance. If no such file is present, `bob-anal` automatically calls `bob-baseline` to build it. The baselines are inferred from the random search performance in the logs. The baseline values are considered fixed (not random) quantities when `bob-anal` builds confidence intervals. Therefore, we allow the user to leave them fixed and do not rebuild them when `bob-anal` is called if a baselines file is already present.

## Example

After finishing the setup (environment) a small-scale serial can be run as follows:

```
# setup
> DB_ROOT=./notebooks  # path/to/where/you/put/results
> DBID=bo_example_folder
> mkdir $DB_ROOT
# experiments
> bob-launch -n 15 -r 3 -dir $DB_ROOT -b $DBID -o RandomSearch PySOT -c SVM DT -v
Supply --uuid 3adc3182635e44ea96969d267591f034 to reproduce this run.
Supply --dbid bo_example_folder to append to this experiment or reproduce jobs file.
User must ensure equal reps of each optimizer for unbiased results
-c DT -d boston -o PySOT -u a1b287b450385ad09b2abd7582f404a2 -m mae -n 15 -p 1 -dir /notebooks -b bo_example_folder
-c DT -d boston -o PySOT -u 63746599ae3f5111a96942d930ba1898 -m mse -n 15 -p 1 -dir /notebooks -b bo_example_folder
-c DT -d boston -o RandomSearch -u 8ba16c880ef45b27ba0909199ab7aa8a -m mae -n 15 -p 1 -dir /notebooks -b bo_example_folder
...
0 failures of benchmark script after 144 studies.
done
# aggregate
> bob-agg -dir $DB_ROOT -b $DBID
# analyze
> bob-anal -dir $DB_ROOT -b $DBID -v
...
median score @ 15:
optimizer
PySOT_0.2.3_9b766b6           0.330404
RandomSearch_0.0.1_9b766b6    0.961829
mean score @ 15:
optimizer
PySOT_0.2.3_9b766b6           0.124262
RandomSearch_0.0.1_9b766b6    0.256422
normed mean score @ 15:
optimizer
PySOT_0.2.3_9b766b6           0.475775
RandomSearch_0.0.1_9b766b6    0.981787
done
```
The aggregate result files (i.e., `summary.json`) will now be available in `$DB_ROOT/$DBID/derived`. However, this will be high variance since it was from only 3 trials and only to 15 function evaluations.

## Plotting and notebooks

Plotting the quantitative results found in `$DB_ROOT/$DBID/derived` can be done using the notebooks found in the `notebooks/` folder of the git repository. The notebook `plot_mean_score.ipynb` generates plots for aggregate scores averaging over all problems. The notebook `plot_test_case.ipynb` generates plots for each test problem.

To use the notebooks, first copy over the `notebooks/` folder from git repository.

To setup the kernel for running the notebooks use:

```
virtualenv bobm_ipynb --python=python3.6
source ./bobm_ipynb/bin/activate
pip install bo-benchmark[notebooks]
python -m ipykernel install --name=bobm_ipynb --user
```
Now, the notebooks for plotting can be run with the command `jupyter notebook` and selecting the kernel `bobm_ipynb`.

It is also possible to convert the notebooks to an HTML report at the command line using `nbconvert`. For example, use the command:

```
jupyter nbconvert --to html --execute notebooks/plot_mean_score.ipynb
```
The output file will be in `./notebooks/plot_mean_score.html`. See the `nbconvert` [documentation](https://nbconvert.readthedocs.io/en/latest/usage.html#supported-output-formats) for more output formats. By default, the notebooks look in `./notebooks/bo_example_folder/` for the `summary.json` from `bob-anal`.

To run `plot_test_case.ipynb` use the command:

```
jupyter nbconvert --to html --execute notebooks/plot_test_case.ipynb --ExecutePreprocessor.timeout=600
```
The `--ExecutePreprocessor.timeout=600` timeout increase is needed due to the large number of plots being generated. The output will be in `./notebooks/plot_test_case.html`.

# Adding a new optimizer

All optimizers in this benchmark are required to follow the interface specified of the `AbstractOptimizer` class in `bo_benchmark.abstract_optimizer`. In general, this requires creating a wrapper class around the new optimizer. The wrapper classes must all be placed in a folder referred to by the `--opt-root` argument. This folder must also contain the `config.json` folder.

The interface is simple, one must merely implement the `suggest` and `observe` functions. The `suggest` function generates new guesses for evaluating the function. Once evaluated, the function evaluations are passed to the `observe` function. The objective function is *not* evaluated by the optimizer class. The objective function is evaluated on outside and results are passed to `observe`. This is the correct setup for Bayesian optimization because:

- We can observe/try inputs that were never suggested
- We can ignore suggestions
- The objective function may not be something as simple as a Python function

So passing the function as an argument as is done in `scipy.optimization` is artificially restrictive.

The implementation of the wrapper will look like the following:

```
from bo_benchmark.abstract_optimizer import AbstractOptimizer
from bo_benchmark.experiment import experiment_main


class NewOptimizerName(AbstractOptimizer):
    # Used for determining the version number of package used
    primary_import = "name of import used e.g, opentuner"

    def __init__(self, api_config, optional_arg_foo=None, optional_arg_bar=None):
        """Build wrapper class to use optimizer in benchmark.

        Parameters
        ----------
        api_config : dict-like of dict-like
            Configuration of the optimization variables. See API description.
        """
        AbstractOptimizer.__init__(self, api_config)
        # Do whatever other setup is needed
        # ...

    def suggest(self, n_suggestions=1):
        """Get suggestion from the optimizer.

        Parameters
        ----------
        n_suggestions : int
            Desired number of parallel suggestions in the output

        Returns
        -------
        next_guess : list of dict
            List of `n_suggestions` suggestions to evaluate the objective
            function. Each suggestion is a dictionary where each key
            corresponds to a parameter being optimized.
        """
        # Do whatever is needed to get the parallel guesses
        # ...
        return x_guess

    def observe(self, X, y):
        """Feed an observation back.

        Parameters
        ----------
        X : list of dict-like
            Places where the objective function has already been evaluated.
            Each suggestion is a dictionary where each key corresponds to a
            parameter being optimized.
        y : array-like, shape (n,)
            Corresponding values where objective has been evaluated
        """
        # Update the model with new objective function observations
        # ...
        # No return statement needed


if __name__ == "__main__":
    # This is the entry point for experiments, so pass the class to experiment_main to use this optimizer.
    # This statement must be included in the wrapper class file:
    experiment_main(NewOptimizerName)
```
Depending on the API of the optimizer being wrapped, building this wrapper class may only or require a few lines of code, or be a total pain.

## The config file

Each optimizer wrapper can have multiple configurations, which is each referred to as a different optimizer in the benchmark. For example, the JSON config file will have entries as follows:

```
    "OpenTuner-BanditA-New": [
        "opentuner_optimizer.py",
        {"techniques": ["AUCBanditMetaTechniqueA"]}
    ],
    "OpenTuner-GA-DE-New": [
        "opentuner_optimizer.py",
        {"techniques": ["PSO_GA_DE"]}
    ],
    "OpenTuner-GA-New": [
        "opentuner_optimizer.py",
        {"techniques": ["PSO_GA_Bandit"]}
    ],
```
Basically, the entries are `"name_of_strategy": ["file_with_class", {kwargs_for_the_constructor}]`. Here, `OpenTuner-BanditA`, `OpenTuner-GA-DE`, and `OpenTuner-GA` are all treated as different optimizers by the benchmark even though the all use the same class from `opentuner_optimizer.py`.

This `config.json` must be in the same folder as the optimizer classes (e.g., `opentuner_optimizer.py`).

## Running with a new optimizer

To run the benchmarks using a new optimizer, simply provide its name (from `config.json`) in the `-o` list. The `--opt-root` argument must be specified in this case. For example, the launch command from the [example](#example) becomes:

```
bob-launch -n 15 -r 3 -dir $DB_ROOT -b $DBID -o RandomSearch PySOT-New -c SVM DT --opt-root ./example_opt_root -v
```
Here, we are using the example `PySOT-New` wrapper from the `example_opt_root` folder in the git repo. It is equivalent to the builtin `PySOT`, but gives an example of how to provide a new custom optimizer.

# Contributing

## Install in editable mode

First, define the variables for the paths we will use:

```
GIT=/path/to/where/you/put/repos
ENVS=/path/to/where/you/put/virtualenvs
```
Then clone the repo in your git directory `$GIT`:

```
cd $GIT
git clone https://github.com/uber/bo-benchmark.git
```
Inside your virtual environments folder `$ENVS`, make the environment:

```
cd $ENVS
virtualenv bo_benchmark --python=python3.6
source $ENVS/bo_benchmark/bin/activate
```
Now we can install the pip dependencies. Move back into your git directory and run

```
cd $GIT/bo-benchmark
pip install -r requirements/base.txt
pip install -r requirements/optimizers.txt
pip install -e .  # Install the benchmark itself
```
You may want to run `pip install -U pip` first if you have an old version of `pip`. The file `optimizers.txt` contains the dependencies for all the optimizers used in the benchmark. The analysis and aggregation programs can be run using only the requirements in `base.txt`.

## Contributor tools

First, we need to setup some needed tools:

```
cd $ENVS
virtualenv bo_benchmark_tools --python=python3.6
source $ENVS/bo_benchmark_tools/bin/activate
pip install -r $GIT/bo-benchmark/requirements/tools.txt
```

To install the pre-commit hooks for contributing run (in the `bo_benchmark_tools` environment):

```
cd $GIT/bo-benchmark
pre-commit install
```

To rebuild the requirements, we can run:

```
cd $GIT/bo-benchmark
# Get py files from notebooks to analyze
jupyter nbconvert --to script notebooks/*.ipynb
# Generate the .in files (but pins to latest, which we might not want)
pipreqs bo_benchmark/ --ignore bo_benchmark/builtin_opt/ --savepath requirements/base.in
pipreqs test/ --savepath requirements/test.in
pipreqs bo_benchmark/builtin_opt/ --savepath requirements/optimizers.in
pipreqs notebooks/ --savepath requirements/ipynb.in
pipreqs docs/ --savepath requirements/docs.in
# Regenerate the .txt files from .in files
pip-compile-multi --no-upgrade
```

## Generating the documentation

First setup the environment for building with `Sphinx`:

```
cd $ENVS
virtualenv bo_benchmark_docs --python=python3.6
source $ENVS/bo_benchmark_docs/bin/activate
pip install -r $GIT/bo-benchmark/requirements/docs.txt
```
Then we can do the build:

```
cd $GIT/bo-benchmark/docs
make all
open _build/html/index.html
```
Documentation will be available in all formats in `Makefile`. Use `make html` to only generate the HTML documentation.

## Running the tests

The tests for this package can be run with:

```
cd $GIT/bo-benchmark
./test.sh
```
The script creates a conda environment using the requirements found in `requirements/test.txt`.

The `test.sh` script *must* be run from a *clean* git repo.

Or if we only want to run the unit tests and not check the adequacy of the requirements files, one can use

```
# Setup environment
cd $ENVS
virtualenv bo_benchmark_test --python=python3.6
source $ENVS/bo_benchmark_test/bin/activate
pip install -r $GIT/bo-benchmark/requirements/test.txt
pip install -e $GIT/bo-benchmark
# Now run tests
cd $GIT/bo-benchmark/
pytest test/ -s -v --hypothesis-seed=0 --disable-pytest-warnings --cov=bo_benchmark --cov-report html
```
A code coverage report will also be produced in `$GIT/bo-benchmark/htmlcov/index.html`.

## Deployment

The wheel (tar ball) for deployment as a pip installable package can be built using the script:

```
cd $GIT/bo-benchmark/
./build_wheel.sh
```

# License

This project is licensed under the Apache 2 License - see the LICENSE file for details.
