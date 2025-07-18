# User Guide (immuno_lsf_wdl_compute1)
A tutorial for running immuno.wdl using LSF on WASHU compute1

## Before you start
### Get access to the WUSTL Oncology Enterprise

Before starting, make sure you are able to pull the [`ghcr.io`](http://ghcr.io) docker image using an interactive session: 

```bash
LSF_DOCKER_PRESERVE_ENVIRONMENT=false bsub -q oncology-interactive -G compute-oncology -n 1 -M 60G -R 'select[mem>60G] span[hosts=1] rusage[mem=60G]' -Is -a 'docker(ghcr.io/genome/genome_perl_environment:compute1-58)' /bin/bash
```

If not, you need to ask to be added into the WUSTL Oncology enterprise (ask in channel general (tom mooney, chris miller, and our PIs have the ability to add people to enterprise). 

More details on that here:

https://github.com/genome/genome/wiki/Pulling-ghcr.io-docker-images

Follow instructions on creating the personal access token and logging into it on the server using that token. 

### Set up SSH key between GitHub and your RIS account.

Setting up SSH for GitHub allows you to securely connect and authenticate to GitHub without needing to repeatedly enter your username and personal access token. It simplifies access to repositories and enables you to perform actions like cloning, pushing, and pulling code. (https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)



## Setup your environment

Create an empty directory with a descriptive name relating to the set of WES and RNA-seq fastq files you wish to process (e.g. `hcc1395_immuno`). This will be your analysis directory. Navigate to that directory.

### Clone git repos
From within your analysis directory, run the following line of code: 

`cloud workflows`- use immuno_local_copy_outputs_jennie branch for now

```bash
git clone https://github.com/wustl-oncology/cloud-workflows.git
cd cloud-workflows
git checkout immuno_local_copy_outputs_jennie # this switches to the correct branch
cd ..
```

`analysis wdls`- use git checkout v1.2.2

```bash
git clone https://github.com/wustl-oncology/analysis-wdls.git
cd analysis-wdls
git checkout v1.2.2
```

This will download the workflow and analysis files from GitHub into folders called `cloud-workflows` and `analysis-wdls` 

### Check strandedness of tumor RNA data

If using RNA data in the immuno pipeline, it is required to know the strandedness of your samples. When you are unsure of the strandedness, follow the steps below to check the strandedness (if already know, skip this step). This information will be used for creating your yaml files in the next step. 

```bash
# Assuming you are now in your analysis directory and have cloned cloud-workflows and analysis-wdls git repos
cd cloud-workflows/manual-workflows

# Open and edit stranded.sh with your desired editor (e.g. vim)
# Change the following paths to your RNA read1 file and read2 files: 
--reads_1 path_to_rna_read_1.fastq \
--reads_2 path_to_rna_read_2.fastq -n 100000 > ../../trimmed_read_strandness_check.txt

# Run this script
bash stranded.sh

# This script will create a file in your working directory: trimmed_read_strandness_check.txt
# The last line of the file will indicate strandedness 
# Put this information later in your yaml file under "immuno.strand"
```

 

### yamls

The yaml files contain exact paths to each samples' raw data. 

Create a folder called `yamls` under the analysis directory. 

Create a yaml file for EACH sample you are processing using templates from the following link: https://github.com/wustl-oncology/immuno_gcp_wdl_compute1/tree/main/example_yamls/human_GRCh38_ens105

Name each yaml as `sample_immuno.yaml` 

### Set up job groups

Setting up job groups is not mandatory, but it is a good practice when you are running many jobs for different projects. It can be useful when you want to check the jobs status for certain project or kill all jobs for that project. 
Note that if you do not supply a job group with the `bsub -g` argument, a default job group named `/${compute_username}/default` will be created with a low default running job limit will be set.
For running the immuno pipeline, it is recommended to set two job groups:
1) `/${compute_username}/2_job` -- This group will have a max job limit of 2 jobs. This ensures that if you submit multiple samples at a time, only two samples will run in parallel simultaneously. This prevents overwhelming the server as the immuno pipeline is resource intensive.
2) `/${compute_username}/${project_name}` -- This group will have a max job limit of >80 (as much as you like). This group is for jobs running WITHIN each sample (e.g. alignment, variant calling, etc.). We want this group to have higher job limits since the pipeline will run several jobs simultaneously for each sample.

To create job groups, follow the code below: 

```bash
# to check existing job groups
bjgroup | grep /${compute_username}

# to add a job group that can run a maximum of 2 jobs:
bgadd -L 2 /${compute_username}/2_job

# to add a job group that can run a maximum of 80 jobs:
bgadd -L 80 /${compute_username}/${project_name}

# to modify job limit to 5 jobs of an existing group:
bgmod -L 5 /${compute_username}/${group_name}
```

### Make user-specific changes

Navigate and edit this file: `cloud-workflows/manual-workflows/cromwell.config.wdl`.

Find the following lines (there are TWO of these in the file) and replace the path to your own LSF job group that can run many jobs at a time (e.g. the `/${compute_username}/${project_name}` you previously created):

```bash
  -g /path/to/your/job_group \
```

### Directory setup
The setup of your directory should look something like this (the pipeline does not assume strict structure for the `raw_data`, as long as all paths are correct in the yaml files): 
```
hcc1395_immuno_analysis_directory/
│
├── cloud-workflows/
│   ├── manual-workflows/
│       └── run_immuno_compute1.sh
│       └── runCromwellWDL.sh
│       └── cromwell.config.wdl
│       └── stranded.sh
├── analysis-wdls/
├── yamls/
│   ├── sample1_immuno.yaml
│   ├── sample2_immuno.yaml
│   ├── sample3_immuno.yaml
│   └── sample4_immuno.yaml
└── raw_data/
    ├── sample1/
    │   └── normal_exome/
    │       └── sample1_normal_exome_1.fastq.gz
    │       └── sample1_normal_exome_2.fastq.gz
    │   └── tumor_exome/
    │       └── sample1_tumor_exome_1.fastq.gz
    │       └── sample1_tumor_exome_2.fastq.gz
    │   └── tumor_rna/
    │       └── sample1_tumor_rna_1.fastq.gz
    │       └── sample1_tumor_rna_2.fastq.gz
    ├── sample2/
    └── sample3/
```

## Submit job to run the immuno workflow

Within your analysis directory, navigate to `cloud-workflows/manual-workflows`.

Double-check that the parameters defined in `run_immuno_compute1.sh` are correct (file paths, cromwell jar, clean scratch directory settings etc.)

Note: for `your_job_group_name` in this command below, it is the job group that will run a maximum of 2 jobs (i.e. `/${compute_username}/2_job`)

```bash
bash run_immuno_compute1.sh "your_sample_ID" "path_to_your_scratch_directory" "your_job_group_name"

# example: bash run_immuno_compute1.sh "Hu_254" "/scratch1/fs1/mgriffit/jyao/miller_immuno/" "/j.x.yao/2_job"
```
