# Splice Experiment

This is planning for the [spliced](https://github.com/buildsi/spliced) experiment
that we plan to run for the BUILDSI project.

## Pre-requisites

1. We need to finish Smeagle, and add as a tester to [spliced](https://github.com/buildsi/spliced)
2. This experiment will need to be setup to run on slurm / a cluster, meaning installing spack and other dependencies (trying to get the current container base into a working environment).
3. Probably something with [libtree](https://github.com/haampie/libtree) TBA.

## Plan

Once the above are done, here is the plan! 

### Matrix Automated Tests

We will first need to choose a set of compilers appropriate for a scoped analysis (likely related to what Smeagle can parse). The [experiments/e4s.yaml](experiments/e4s.yaml) will be run with a custom Python script
that reads in each e4s package, and:

```bash
For every package in e4s P with tests:
  For every dependency of package P, D that is also in e4s:
    Perform splice of package P and dependency D
```

This means that we also need to know the sample of the e4s packages that have tests.
I wrote a script to do that.

```bash
$ spack python has_tests.py experiments/e4s.yaml
44 packages out of 90 have tests in e4s.yaml
```

This will also generate a file with the subset of packages that have tests:

```bash
$ cat experiments/has_tests.yaml 
experiments:
- swig
...
```

The original spack.yaml for e4s is in [ref](ref) for reference. Finally, once you  have
this list, you can do a run on your host to determine which tests might be broken (my first
run 14/44 failed, or a little less than 1/3, where 1/3 is about half of the libraries in the e4s set.

```bash
$ python run_tests.py
qthreads             test bug or failure
hypre                test bug or failure
superlu              test bug or failure
kokkos               test bug or failure
bolt                 test bug or failure
parsec               test bug or failure
superlu-dist         test bug or failure
heffte               test bug or failure
aml                  install failure
arborx               test bug or failure
tasmanian            test bug or failure
slepc                test bug or failure
ginkgo               test bug or failure
caliper              test bug or failure
```

For full details and error messages you can see [experiments/has_tests_results.json](experiments/has_tests_results.json).


### Manual Tests

Since we can't splice in things that aren't dependencies in spack (and we need the tests above) we also need a manual approach
that picks/chooses some manual cases that we know will cause issue (e.g. mpich and openmpi) to run those.
We will want to do these "by hand" and perhaps run a manual experiment (not developed yet in spliced but should be fairly
easy to do) to run the predictions without needing to spack install them.


## Running the Experiment

Let's clone the experiment repository to get the examples and scripts.
We have space in `/p/vast1/build`

```bash
cd /p/vast1/build
git clone https://github.com/buildsi/spliced-experiment
```

Let's install anaconda to avoid pain.

```bash
wget https://repo.anaconda.com/archive/Anaconda3-5.3.1-Linux-x86_64.sh
chmod +x Anaconda3-5.3.1-Linux-x86_64.sh
./Anaconda3-5.3.1-Linux-x86_64.sh -p /p/vast1/build/anaconda3
```

And we need spack.

```bash
git clone -b vsoch/db-17-splice https://github.com/vsoch/spack
. spack/share/spack/setup-env.sh 

# always build with debug (this is in template script too)
export SPACK_ADD_DEBUG_FLAGS=true

# add anaconda (or your favorite python install) to the path to install spliced
export PATH=/p/vast1/build/anaconda3/bin:$PATH
spack compiler find
```

Install [spliced](https://github.com/buildsi/spliced) and [symbolator](https://github.com/buildsi/symbolator)

```bash
pip install spliced symbolator-python
```

Note that @vsoch is testing the branch `add/spack-tests` that will run spack tests
with a splice for the command. You can see example splices in [splices](splices) and we are going to be generating them programatically
based on tests we have.

## Generating experiments

To generate new experiment files we can do the following:

```bash
$ mkdir -p splices
$ spack python generate_experiments.py splices/
```

Note that @vsoch has probably already run this if there is a "splices" directory in the
repository. Then you can have spliced generate the commands for you.  Here is an example
to run on your own to see:

```bash
spliced command splices/swig/pcre/pcre/experiment.yaml
```

Take a look at the commands generated above if you are interested. But let's do this in Python. Make a root output directory alongside spack
```bash
$ mkdir -p results
```

The script [submit_jobs.py](submit_jobs.py) will do exactly that.

**important** I've hard coded the template for the submission script at the bottom of that script, please
change this to be where your spack install is, etc. It's hard coded for mine because *reasons*.

The above will submit a bunch of jobs for all versions of the input parameters on the cluster,
and keep scripts in `$PWD/scripts`

```bash
                        # experiment                           # output directory
$ python submit_jobs.py splices/swig/pcre/pcre/experiment.yaml results
```

Note that @vsoch needs to add a boolean to spliced to say "run spack tests for this splice"
and then it will be ready to go.
