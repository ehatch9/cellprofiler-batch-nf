# cellprofiler-batch-nf

Batch execution of CellProfiler analyses using Nextflow

# Purpose

Parallelize the execution of CellProfiler analysis using batch computing or HPC services using the Nextflow workflow management system. This relies upon the free and open-source tool for image analysis: [CellProfiler](https://cellprofiler.org/).

The user starts by setting up an analysis in the CellProfiler GUI, and then selecting the "Create Batch File" option to create a batch file which can be used as the input to this workflow.

When this workflow is run on that batch file, the analyses specified by the user is split up and distributed across multiple nodes for more rapid execution.

# Usage

```
    Usage:

    nextflow run FredHutch/cellprofiler-batch-nf <ARGUMENTS>

    Required Arguments:
      --input_h5            Batch file created by the CellProfiler GUI interface defining the analysis to run
      --input_csv           CSV file containing the parameters + groupings for this run
      --output              Path to output directory

    Optional Arguments:
      --n                   Number of images to analyze in each batch (default: 1000)
      --concat_n            Number of tabular results to combine/concatenate in the first round (default: 100)
      --version             Software version CellProfiler (default: 4.1.3)
                            Must correspond to tag available at hub.docker.com/r/cellprofiler/cellprofiler/tags
      --group_col           The name of the grouping column in the CSV file (default: Group_Number)
      --folder_col          The name of the folder column in the CSV file (default: PathName_Orig)
      --file_col            The name of the file column in the CSV file (default: FileName_Orig)
      --shard_col           The name of the column being added that has the shard (default: Shard_Id)
      --file_prefix_in      The value of the folder path prefix, in the folder column of the CSV 
                               that needs to be changed (no default, ignored when empty)
      --file_prefix_out     When using --file_prefix_in, set this to be the value of the folder path
                               to change into

    CellProfiler Citations: See https://cellprofiler.org/citations
    Workflow: https://github.com/FredHutch/cellprofiler-batch-nf
```


## Example

Here's an example of how to run this workflow, in seven steps.

1. Make/find a source folder. Put the images there.

```bash
mkdir -p WORK_DIR/EXAMPLE_IMAGES/
```

2. Make the output folder. This is where the workflow results will go.

```bash
mkdir -p WORK_DIR/EXAMPLE_OUTPUT/
```

3. Make a 'project' folder to put the workflow & code. 
  * Put your project file (a `.cppipe` file) there. This file is created by using the CellProfiler app, usually on your desktop/laptop. 

```bash
mkdir -p WORK_DIR/EXAMPLE_PROJECT/
```

You also need to put the `parse_csv.py` script in there, into a `templates` folder:

```bash
mkdir -p WORK_DIR/EXAMPLE_PROJECT/templates/
```

Copy the `parse_csv.py` file so it is located at `WORK_DIR/EXAMPLE_PROJECT/templates/parse_csv.py`. You *must* do this, or else the script will not run.


4. Make a CSV file with the list of images to run. This is created in CellProfiler.


5. Make sure the file paths in the CSV will end up working

If you're not running CellProfiler in the same way that you will run the workflow (e.g. 'On the compute cluster'), then you may need to use the `FILE_PREFIX_IN` and `FILE_PREFIX_OUT` fields.

In the compute environment where you'll be running this workflow, find the location of one of the input files. Copy that location down.

Open up the CSV file from step 4, and look at the `PathName_Orig` column. Find the same image, and copy its location down

Does the location (e.g. `/compute/storage/example/location/image-1.tiff`) match what's in the CSV file (e.g. `/desktop/volume/example/location/image-1.tiff`)?

If they are different, we'll need to swap them. We can do that by setting the `FILE_PREFIX_IN` value to be what to remove, and `FILE_PREFIX_OUT` to be what to replace it with.

Using the example above, we would set the values to be:

```bash
FILE_PREFIX_IN="/desktop/volume"
FILE_PREFIX_OUT="/compute/storage"
```


6. Make a 'run' script. This has all of the variables needed to run Nextflow. It also makes reproducibility easy: just re-run the script.

Here's an example run script, which we'll call `run.sh`. Save it to the project folder (e.g. `WORK_DIR/EXAMPLE_PROJECT/`)

```bash
#!/bin/bash

# INPUT
INPUT_H5='EXAMPLE_PROJECT.cppipe'
INPUT_CSV='filelist.csv'   # the location of the CSV file from step 4
INPUT_CONCAT_N=100
INPUT_BATCH_SIZE=1000

# CSV
FILE_PREFIX_IN="/EXAMPLE_INPUT_PREFIX"   # see step 5
FILE_PREFIX_OUT="/EXAMPLE_OUTPUT_PREFIX"

#OUTPUT
PROJECT=EXAMPLE_PROJECT
OUTPUT_DIR=WORK_DIR/EXAMPLE_OUTPUT/

# Profile (used to allocate resources)
PROFILE=standard

# Run the workflow
NXF_VER=20.10.0 \
nextflow \
    run \
    -profile $PROFILE \
    FredHutch/cellprofiler-batch-nf \
    --input_h5 "${INPUT_H5}" \
    --input_csv "${INPUT_CSV}" \
    --file_prefix_in "${FILE_PREFIX_IN}" \
    --file_prefix_out "${FILE_PREFIX_OUT}" \
    --n $INPUT_BATCH_SIZE \
    --output ${OUTPUT_DIR} \
    -with-report $PROJECT.report.html \
    -with-trace $PROJECT.trace.tsv \
    -resume \
    -latest

```
The script is creating bash variables for the workflow parameters and then running the Nextflow workflow.




7. Finally, run the script.

```bash
cd WORK_DIR/EXAMPLE_PROJECT/
bash run.sh
```

The script will run and put the output in the output folder/directory. In testing, processing ~480 images took 10-18 minutes.



### Using Multiple Configs

If you have a Nextflow config file that is used to execute workflows in your environment, Nextflow [can combine it](https://www.nextflow.io/docs/latest/config.html) with the GitHub repo's `nextflow.config`. 

For example, let's say you have a `default.nextflow.config` file that is used to run in your environment (e.g. to support AWS Batch, or Slurm, or Singularity). You'd update the script to include a `BASE_CONFIG` variable, and update the `nextflow run` command parameters to include it.


```bash

BASE_CONFIG=WORK_DIR/default.nextflow.config

# Run the workflow
NXF_VER=20.10.0 \
nextflow \
    run \
    -c $BASE_CONFIG \
    -profile $PROFILE \
    FredHutch/cellprofiler-batch-nf \
    --input_h5 "${INPUT_H5}" \
    --input_csv "${INPUT_CSV}" \
    --file_prefix_in "${FILE_PREFIX_IN}" \
    --file_prefix_out "${FILE_PREFIX_OUT}" \
    --n $INPUT_BATCH_SIZE \
    --output ${OUTPUT_DIR} \
    -with-report $PROJECT.report.html \
    -with-trace $PROJECT.trace.tsv \
    -resume \
    -latest

```

### Shard and Groups

If we have a large number of images, we will split up the work into multiple smaller runs of CellProfiler, so the process completes more quickly. The default 'split' size is set by `INPUT_BATCH_SIZE`. We do this by assigning images into smaller collections called *shards*.

Since CellProfiler can work on groups of images, we need to make sure that all of the images in a group end up in the same CellProfiler run. 

We do this by processing the CSV, identifying groups, and assigning all of images in a group into the same shard. If there are no groups, images are assigned in the order they show up in the CSV file (e.g. 'the first 10 images go into shard 1 if the `INPUT_BATCH_SIZE` is 10).


## CellProfiler pipelines

The workflow requires a CellProfiler pipeline (`.cppipe`) file. This can be created from a project by going to *File -> Export -> Pipeline*. See the [CellProfiler manual](https://cellprofiler-manual.s3.amazonaws.com/CellProfiler-4.1.3/help/projects_introduction.html?highlight=cppipe#) for details. Look for the 'Saving a project' section.