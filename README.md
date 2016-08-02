# variant-calling-benchmarks
Automated and curated variant calling benchmarks for Guacamole

This repository was created to aid development of the [Guacamole](https://github.com/hammerlab/guacamole) somatic variant caller. Eventually we may support benchmarking other variant callers.

The sequencing data for most benchmarks comes from publically available datasets but cannot be directly released by us. To run these benchmarks you will have to acquire the BAMs yourself first.

## Installation

From a checkout run:
```
pip install -e .
```

Also checkout and build guacamole.

## Config files

The benchmark defintions and guacamole configurations are given in JSON. They are split into multiple files for clarity but the 'configuration' is just the union of all the files passed. The code doesn't care what the files are or which file has a particular property.

A simple substitution mechanism is supported where strings like "foo-${NAME}" will have $NAME expanded according to substitutions defined in the config files (under the 'substitutions' keys). The variable `$THIS_DIR` always expands to the absolute path of the directory the current JSON file is found in.

## Benchmark definition

The [local-wgs1 benchmark](benchmarks/local-wgs1/benchmark.json) is a small benchmark that can be run locally using BAMs that are checked into the repo.

A benchmark consists of a set of *patients* and a set of *variants collections*. Each patient has at least two BAMs (tumor and normal) but can have more, including RNA. A variant collection is any set of variants we want to compare guacamole against, such as validated calls or calls generated by another caller. Variant collections are given as a single CSV file containing variants for all patients in the benchmark (see [here](benchmarks/local-wgs1/published_calls.csv) for an example).

When a benchmark is run, recall, precision, and other metrics are computed across patients by comparing the guacamole calls to each of the variant collections. A CSV file of merged calls from guacamole and the variant collections is also written so further comparisons of these call sets may be performed. The guacamole invocation force calls the sites in the benchmark variant collections so guacamole likelihood and filter information is available for all sites in the benchmark whether or not they triggered a guacamole call.

Note that variant calls are associated with patients, not samples. This is the "joint calling" approach: we do not call variants on individual BAMs but rather on the entire patient using all BAMs.

## Running Locally

Change `$GUACAMOLE_HOME` here to point to your checkout of the guacamole repository.

```
GUACAMOLE_HOME=~/sinai/git/guacamole
vcb-guacamole-local \
    base_config.json \
    infrastructure-configs/local.json \
    guacamole-configs/default.json \
    benchmarks/local-wgs1/benchmark.json \
    --guacamole-jar $GUACAMOLE_HOME/target/guacamole-0.0.1-SNAPSHOT.jar \
    --guacamole-dependencies-jar $GUACAMOLE_HOME/target/guacamole-deps-only-0.0.1-SNAPSHOT.jar \
    --out-dir results
```

## Running on a cluster

We override `TMPDIR` because the default (`/tmp`) is not mounted on our cluster nodes. The node you call the script from and the spark driver node (not the executors) must be able to share the temporary files. The driver node must also be able to write to the directory specified in `--out-dir`.

```
mkdir -p tmp
GUACAMOLE_HOME=~/sinai/git/guacamole
TMPDIR=$(pwd)/tmp vcb-guacamole-cluster \
    base_config.json \
    infrastructure-configs/demeter.json \
    guacamole-configs/default.json \
    benchmarks/aocs/benchmark.json \
    --guacamole-jar $GUACAMOLE_HOME/target/guacamole-0.0.1-SNAPSHOT.jar \
    --guacamole-dependencies-jar $GUACAMOLE_HOME/target/guacamole-deps-only-0.0.1-SNAPSHOT.jar \
    --out-dir results
```

## Submitting results to google cloud storage (Hammer Lab users only)

We're using Google Cloud Storage to store benchmark results. First, install the [Google Cloud SDK](https://cloud.google.com/sdk/docs/) and login.

After you've run a benchmark you can copy the results to GCS with a command like:

```
gsutil rsync -d results gs://variant-calling-benchmarks-results/tim-$(date +"%m-%d-%y")
```

You can also pull down all submitted benchmarks to your local machine with a command like:

```
gsutil -m rsync -r \
    gs://variant-calling-benchmarks-results \
    ~/sinai/data/gcs/variant-calling-benchmarks-results
```

