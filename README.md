# User Guide (immuno_lsf_wdl_compute1)
A tutorial for running immuno.wdl using LSF on WASHU compute1

## 1. Before you start
### Get access to the WUSTL Oncology Enterprise

Before starting, make sure you are able to pull the [`ghcr.io`](http://ghcr.io) docker image using an interactive session: 

```bash
LSF_DOCKER_PRESERVE_ENVIRONMENT=false bsub -q oncology-interactive -G compute-oncology -n 1 -M 60G -R 'select[mem>60G] span[hosts=1] rusage[mem=60G]' -Is -a 'docker(ghcr.io/genome/genome_perl_environment:compute1-58)' /bin/bash

# if the above worked, exit the interactive session
exit
```

If not, you need to ask to be added into the WUSTL Oncology enterprise (ask in channel general (Tom Mooney, Chris Miller, and our PIs have the ability to add people to the enterprise). 

More details on that here:

https://github.com/genome/genome/wiki/Pulling-ghcr.io-docker-images

Follow instructions on creating the personal access token and logging into it on the server using that token. 

### Set up SSH key between GitHub and your RIS account.

Setting up SSH for GitHub allows you to securely connect and authenticate to GitHub without needing to repeatedly enter your username and personal access token. It simplifies access to repositories and enables you to perform actions like cloning, pushing, and pulling code. (https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)



## 2. Setup your environment

Create an empty directory with a descriptive name relating to the set of WES and RNA-seq fastq files you wish to process (e.g. `hcc1395_immuno`). This will be your analysis/working directory. Navigate to that directory.

### Clone git repos
From within your analysis directory, clone two repositories: 

1. `cloud workflows`- use immuno_local_copy_outputs_jennie branch for now

```bash
git clone https://github.com/wustl-oncology/cloud-workflows.git
cd cloud-workflows
git checkout immuno_local_copy_outputs_jennie # this switches to the correct branch
cd ..
```

2. `analysis wdls`

```bash
git clone https://github.com/wustl-oncology/analysis-wdls.git
cd analysis-wdls
git checkout v1.3.0 # will use pvactools v5.3.0
cd ..
```

This will download the workflow and analysis files from GitHub into folders called `cloud-workflows` and `analysis-wdls`. (Note: the git tags/version for these repos may be upadated from time to time, to make sure you have the most up to dated versions, check if these align with those [here](https://github.com/wustl-oncology/immuno_gcp_wdl_compute1?tab=readme-ov-file#clone-git-repositories-that-have-the-workflows-pipelines-and-scripts-to-help-run-them).


### Check strandedness of tumor RNA data

If using RNA data in the immuno pipeline, it is required to know the strandedness of your samples. When you are unsure of the strandedness, follow the steps below to check the strandedness (if already know, skip this step). This information will be used for creating your yaml files in the next step. 

```bash
# Navigate to your analysis directory if you are not already in there
cd $ANALYSIS_DIRECTORY

# Enter an interactive session
bsub -q general-interactive -G compute-mgriffit -n 1 -M 32G -R 'select[mem>32G] span[hosts=1] rusage[mem=32G]' -Is -a 'docker(mgibio/checkstrandedness:v1)' /bin/bash

# Run the following command to check the starndedness of your RNA sample:
check_strandedness --print_commands \
	--gtf /storage1/fs1/mgriffit/Active/griffithlab/common/reference_files/human_GRCh38_ens105/rna_seq_annotation/Homo_sapiens.GRCh38.105.gtf \
	--kallisto_index /storage1/fs1/mgriffit/Active/griffithlab/common/reference_files/human_GRCh38_ens105/rna_seq_annotation/Homo_sapiens.GRCh38.cdna.all.fa.kallisto.idx \
	--reads_1 path_to_your_Read1_RNA_DATA/R1.fastq.gz \
	--reads_2 path_to_your_Read2_RNA_DATA/R2.fastq.gz -n 100000 > ./read_strandness_check.txt
```
This script will create a file in your analysis directory: read_strandness_check.txt
The last line of the file will indicate strandedness 
Put this information later in the RNA section of your yaml file under "immuno.strand"
 

### yamls

The yaml files contain exact paths to each samples' raw data. 

Create a folder called `yamls` under the analysis directory. 

Create a yaml file for EACH sample you are processing using templates [here](https://github.com/wustl-oncology/analysis-wdls/blob/main/example_data/immuno_storage1.yml)

Name each yaml as `sample_immuno.yaml` 

#### Check YAML for common errors
Use validate_immuno_yaml.py to check for common errors that come up during creation of the immuno YAML such as syntax errors, mismatched sample names, missing values, etc.
```bash
cd $WORKING_BASE/yamls
bsub -Is -q oncology-interactive -G $GROUP -a "docker(mgibio/cloudize-workflow:latest)" /bin/bash

python3 /opt/scripts/validate_immuno_yaml.py $LOCAL_YAML

exit
```


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

1. Change job group in the config file  

   Navigate and edit this file: `cloud-workflows/manual-workflows/cromwell.config.wdl`.

   Find the following lines (there are TWO of these in the file) and replace the path to your own LSF job group that can run many jobs at a time (e.g. the `/${compute_username}/${project_name}` you previously created):

   ```bash
   -g /path/to/your/job_group \ # <- change this to /${compute_username}/${project_name} job group you made in the previous step
   ```

3. Clean scratch dirctory after pipeline run

   The immuno pipeline write numerous temperary files while running, and these files take up a lot of space in the scratch directory. When testing the pipeline for the first time we recommend keeping all temperary files in the scratch directory. After testing, it is recommended to change the following in the `run_immuno_compute1.sh` file to erase temperary files from the scratch directory:
   ```
   # change all instances of
   --clean NO
   # to
   --clean YES
   ```

### Directory setup
The script to launch the pipeline `run_immuno_compute1.sh` depends strongly on the structure of the following directory setup (except for `raw data`, paths to your raw data is set in the `yamls`). The setup of your directory should look something like this: 
```
hcc1395_immuno_analysis_directory/
│
├── cloud-workflows/
│   ├── manual-workflows/
│       └── run_immuno_compute1.sh
│       └── runCromwellWDL.sh
│       └── cromwell.config.wdl
│   └── ...
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

## 3. Submit job to run the immuno workflow

Within your analysis directory, navigate to `cloud-workflows/manual-workflows`.

Double-check that the parameters defined in `run_immuno_compute1.sh` are correct (file paths, cromwell jar, clean scratch directory settings etc.)

You will need to input 4 arguments to run the command: 
1) `--sample` - sample name in the format of "Hu_254" for a single sample or "Hu_344 Hu_048" for multiple samples.
2) `--work_dir` - the path of your analysis/working directory (e.g. `hcc1395_immuno_analysis_directory` in the example above)
3) `--scratch_dir` - the path of your scratch directory
4) `--job_group` - the job group that will run a maximum of 2 jobs (i.e. `/${compute_username}/2_job`)

Example command: 
```bash
bash run_immuno_compute1.sh --sample "Hu_254" --work_dir "/storage1/fs1/mgriffit/Active/immune/j.yao/Miller/immuno" --scratch_dir "/scratch1/fs1/mgriffit/jyao/miller_immuno" --job_group "/j.x.yao/2_job"
# Example usage: bash run_immuno_compute1.sh --sample "Hu_254" --scratch_dir "/scratch1/fs1/mgriffit/jyao/miller_immuno/" --job_group "/j.x.yao/2_job"
# Example usage to submit multiple samples: bash run_immuno_compute1.sh --sample "Hu_344 Hu_048" --scratch_dir "/scratch1/fs1/mgriffit/jyao/miller_immuno/" --job_group "/j.x.yao/2_job"

# for more information on this run cammand, use the following:
bash run_immuno_compute1.sh --help
```
