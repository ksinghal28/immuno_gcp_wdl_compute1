# Running the WASHU Immunogenomics Workflow on Google Cloud - compute1 version

## Preamble
This tutorial demonstrates how to run the WASHU immunogenomics pipeline (immuno.wdl) on Google Cloud.
This will be done by first setting up the workflow definitions, input data and reference files and a 
YAML config file on a user's local system. The user will also set up a Google Cloud environment
and all the inputs will be staged to this cloud environment.  Next Cromwell will be used to execute the pipeline
using the specified input and reference files, and finally the results will be pulled back to the local system
and cloud resources will be cleaned up. 

This version assumes that your are staging your input data files from the WASHU RIS storage1/compute1 cluster. 
Some steps are run within docker containers on that cluster. It therefore requires that you have access to 
storage1 disk and ability to launch jobs on compute1 using LSF (bsub).

### Source of instructions
This tutorial is a specific example of how to run a specific pipeline (immuno) on a specific example dataset (HCC1395 Tumor/normal cell line pair). The steps below are taken from the following link where you will find a  more generic set of documentation that explains in detail how to run any WDL pipeline on the Google Cloud using tools created to assist this process. 
https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows

### Prerequisites
- google-cloud-sdk
- git

### The example data set and analysis to be performed
To demonstrate an analysis on the Google cloud we will run the WASHU immunogenomics pipeline on a publicly available set of exome and bulk RNA-seq data generated for a tumor/normal cell line pair (HCC1395 and HCC1395/BL). The HCC1395 cell line is a well known breast cancer cell line that can be purchased and is commonly used for benchmarking cancer genomics analysis and methods development. The datasets we will use here are realistic deeply sequenced exome and RNA-seq data. The immunogenomics pipeline is a very elaborate end-to-end pipeline that starts with raw data and performs data QC, germline variant calling, somatic variant calling (multiple variant callers and variant types), HLA typing, RNA expression analysis and neoantigen identification.  

### Interacting with Google buckets from your local system
Note that you can use this docker image to access `gsutil` for exploration of your google storage: `docker(google/cloud-sdk)`

## Step-by-step instructions

### Set some Google Cloud and other environment variables
The following environment variables are used merely for convenience and should be customized to produce intuitive labeling for your own analysis:
```bash
export GROUP=compute-oncology
export GCS_PROJECT=griffith-lab
export GCS_SERVICE_ACCOUNT=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com
export GCS_BUCKET_NAME=griffith-lab-test-kartik
export GCS_BUCKET_PATH=gs://$GCS_BUCKET_NAME
export GCS_INSTANCE_NAME=kartik-immuno-test
export BASE=/storage1/fs1/mgriffit/Active/griffithlab/pipeline_test
export WORKING_BASE=$BASE/kartik
export RAW_DATA_DIR=$BASE/raw_data
export WORKFLOW=immuno.wdl
export WORKFLOW_PATH=$WORKING_BASE/git/analysis-wdls/definitions/$WORKFLOW
export LOCAL_YAML=mcdb024_immuno_local-WDL.yaml
export CLOUD_YAML=mcdb024_immuno_cloud-WDL.yaml
```

## Local setup

### First create a working directory on your local system
The following directory on the local system will contain: (a) a git repository for this tutorial including an example YAML file set up to work with the test HCC1395 data, (b) git repository for the WDL workflows, including the immuno worflow, (c) git repository for tools that help provision and manage our workflow runs on the cloud, (d) raw data that we will download for this tutorial, (e) a YAML file describing the input data and paramters for the analysis, and (f) final results file from the workflow that we will pull down from the cloud after a successful run.
```bash
mkdir $WORKING_BASE
cd $WORKING_BASE
```

### Clone git repositories that have the workflows (pipelines) and scripts to help run them
The following repositories contain: this tutorial (immuno_gcp_wdl), the WDL workflows (analysis-wdls), and tools for running these on the cloud (cloud-workflows). Note that the command below will clone the main branch of each repo. But as a comment at the end of each command in brackets is a specific tag of that repo. The combination of these tags for each repo were tested together and verified to work for this tutorial.
```bash
mkdir git
cd git
git clone git@github.com:ksinghal28/immuno_gcp_wdl_compute1.git # (v1.0.1)
git clone git@github.com:griffithlab/analysis-wdls.git # (v1.0.0)
git clone git@github.com:griffithlab/cloud-workflows.git # (v1.3.0)
```

### Login to GCP and set the desired project
From the command line, you will need to authenticate your cloud access (using your google cloud account credentials). This generally only needs to be done once, though there is no harm to re-authenticating. The login command below will generate a custom URL to enter in your browser. Once you do this, you will be prompted to log into your Google account. If you have multiple Google accounts (e.g. one for your institution/company and a personal one) be sure to use the correct one.  Once you have logged in you will be presented with a long code. Enter this at the prompt generated by the login command below. Finally, set your desired Google Project. This Project should correspond to a Google Billing account in the Google console. If you are using Google Cloud for the first time, both billing and a project should be set up before proceeding. Configuring billing alerts is also probably wise at this point.
```bash
bsub -Is -q oncology-interactive -G $GROUP -a "docker(google/cloud-sdk:latest)" /bin/bash
gcloud auth login
gcloud config set project $GCS_PROJECT
gcloud config list
exit
```

### Set up cloud account and bucket
Run the following command and make note of the "Service Account" returned (e.g. "cromwell-server@griffith-lab.iam.gserviceaccount.com").

This inititialization step does several things: creates a few service accounts (`cromwell-server` and `cromwell-compute`), updates some user IAM permissions for these accounts, creates a network (`cloud-workflows`) and sub-network (`cloud-workflows-default`), and creates a Google bucket.

Note that the IP ranges or "CIDRs" specified below specify all the IP addresses that a user from WASHU might be coming from. A firewall rule is created based on these two ranges to limit access to only those users on the WASHU network. Details of this firewall rule will appear in the Google Cloud console under: `VPC network` -> `Firewall` -> `cloud-workflows-allow-ssh`.

Note that if any of these things are already initialized already (due to a previous run), the script will report errors, but if the message describes that the resource already exists, this is fine.

```bash
cd $WORKING_BASE/git/cloud-workflows/manual-workflows/
bash resources.sh init-project --project $GCS_PROJECT --bucket $GCS_BUCKET_NAME --ip-range "128.252.0.0/16,65.254.96.0/19"
```

This step should have created two new configuration files in your current directory: `cromwell.conf` and `workflow_options.json`.

### Setup input data and reference files

Download RAW data for hcc1395
```bash
mkdir -p $RAW_DATA_DIR/hcc1395
cd $RAW_DATA_DIR/hcc1395
wget http://genomedata.org/pmbio-workshop/fastqs/all/Exome_Norm.tar
wget http://genomedata.org/pmbio-workshop/fastqs/all/Exome_Tumor.tar
wget http://genomedata.org/pmbio-workshop/fastqs/all/RNAseq_Tumor.tar
tar -xvf Exome_Norm.tar Exome_Tumor.tar RNAseq_Tumor.tar
rm -f Exome_Norm.tar Exome_Tumor.tar RNAseq_Tumor.tar
```

### Gather input data and reference files to your local system
Create a directory for YAML files and create one for the desired pipeline that points to the location of input files on your local system

```bash
cd $WORKING_BASE
mkdir yamls
cd yamls
```

Setup yaml files for an example run.
```bash
cp $WORKING_BASE/git/immuno_gcp_wdl_compute1/example_yamls/human_GRCh38_ens105/$LOCAL_YAML $WORKING_BASE/yamls/
```

Note that this YAML file has been set up to work with the HCC1395 raw data files downloaded above. If you are modifying this tutorial to work with your own data, you will need to modify the beginning of the YAML that relates to input sequence files.  For both DNA and RNA files, both FASTQ and Unaligned BAM files are supported as input.  Similarly, you have have your data in one file (or one file pair) or you may have multiple data files that will be merged together. Depending on how your input data is organized the YAML entries will look slightly different.

### Stage input files to cloud bucket

Start an interactive docker session capable of running the "cloudize" scripts
```bash
bsub -Is -q oncology-interactive -G $GROUP -a "docker(mgibio/cloudize-workflow:latest)" /bin/bash
```

Attempt to cloudize your workflow and inputs
```bash
python3 /opt/scripts/cloudize-workflow.py $GCS_BUCKET_NAME $WORKFLOW_PATH $WORKING_BASE/yamls/$LOCAL_YAML --output=$WORKING_BASE/yamls/$CLOUD_YAML
```

Note that this "cloudize" step is primarily about staging input files that you may have locally on your system that need to be copied to a google cloud bucket so that they can be accessed during your workflow run on the cloud. It therefore takes the desired Google Cloud Bucket Name as input. It also takes your Workflow Definition (WDL file) for your pipeline.  It will perform some basic checks of the input expectations of this workflow against the input YAML configuration file you provide. The final input to this script is your input YAML configuration file (the LOCAL YAML). This should contain all the input parameters needed by your workflow, including paths to reference, annotation and data files. If any of the file paths in your input YAML are local paths, this script will attempt to copy them into your bucket. If a file path is already specified as a Google Cloud Bucket path (e.g. gs://PATH ) it will be skipped. The output of the script (the CLOUD YAML), is a version of your YAML that has been updated to point to new paths for any files that were copied into the cloud bucket. This is the YAML you will use to launch your workflow run in the following steps.

If you get an error during this step, a common cause is that there is some disconnect between what inputs you have defined in your YAML file and what the WDL workflow definition expects. Often the error message will hint at where to look in these two files.

### Start a Google VM that will run Cromwell and orchestrate completion of the workflow

By default, the following command will launch a `e2-standard-2` Google VM (2 CPUs and 8 GB memory).  Note that Cromwell produces a large quantity of database logging. To ensure we have enough space for a least a few runs and to localize intermediate and final results files from the workflow (which include numerous large BAMs) you should add more disk. When not testing, this can probably be safely reduced to 30-40GB. For performance reasons you might also also increase memory. Similarly, you might chose to use faster disk (instead of standard) for the boot disk. To customize these settings add the following parameters to your `start.sh` command below: 

- `--machine-type=e2-highmem-2`. Use an instance with 2 CPUs and 16GB memory (default would be 8 GB for an `e2-standard-2` instance).
- `--boot-disk-size=150GB`. Increase boot disk to 150 GB (default would be 10 GB).
- `--boot-disk-type=pd-ssd`. Use SSD disk (default would HDD).

For more options on configuration of the VM refer to: `gcloud compute instances create --help`. For more information on instance types and costs refer to the [vm-instance-pricing guide](https://cloud.google.com/compute/vm-instance-pricing).

```bash
cd $WORKING_BASE/git/cloud-workflows/manual-workflows/
bash start.sh $GCS_INSTANCE_NAME --server-account $GCS_SERVICE_ACCOUNT --project $GCS_PROJECT --boot-disk-size=50GB --boot-disk-type=pd-standard --machine-type=e2-standard-2
exit #leave the docker session
```

### Log into the VM and check status 

In this step we will confirm that we can log into our Cromwell VM with `gcloud compute ssh` and make sure it is ready for use.

After logging in, use journalctl to see if the instance start up has completed, and cromwell launch has completed.

For details on how to recognize whether these processes have completed refer: [here](https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows#ssh-in-to-vm).

```bash
gcloud compute ssh $GCS_INSTANCE_NAME
journalctl -u google-startup-scripts -f
journalctl -u cromwell -f
exit #leave the google VM
```

### Localize your inputs file

First **on your local system**, copy your cloudized YAML file to a google bucket

```bash
cd $WORKING_BASE/yamls/
gsutil cp $CLOUD_YAML $GCS_BUCKET_PATH/yamls/$CLOUD_YAML
```

Now log into Google Cromwell VM instance again and copy the YAML file to its local file system
```bash
gcloud compute ssh $GCS_INSTANCE_NAME

export GCS_BUCKET_PATH=gs://griffith-lab-test-kartik
export CLOUD_YAML=mcdb024_immuno_cloud-WDL.yaml
export WORKFLOW=immuno.wdl

gsutil cp $GCS_BUCKET_PATH/yamls/$CLOUD_YAML .

``` 

### Run the immuno workflow using everything setup thus far

While logged into the Google Cromwell VM instance:
```bash
source /shared/helpers.sh
submit_workflow /shared/analysis-wdls/definitions/$WORKFLOW $CLOUD_YAML

```

### Monitor progress of the workflow run:

While the job is running you can see Cromwell logs live as they occur by doing this
```bash
journalctl -f -u cromwell

```

You can also query the cromwell server using your $WORKFLOW_ID. This ID will have been printed out when you launched the worflow. It will also be the top level UUID in the output files being generated in your Google Bucket.

Example commands that query the Cromwell server and get the current status, current timing diagram, and then save that to the Google Bucket for easy access:
```bash
export WORKFLOW_ID=<id from above>
curl http://localhost:8000/api/workflows/v1/$WORKFLOW_ID/status
curl http://localhost:8000/api/workflows/v1/$WORKFLOW_ID/timing > $WORKFLOW_ID.timing.html
gsutil cp $WORKFLOW_ID.timing.html $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/$WORKFLOW_ID.timing.html

```

Note that if you have been testing and have several workflows that have been run on the same server it can get confusing to tell which workflow is which and in what order they were started.  The following command will parse the cromwell log for records of workflows that were started, in order and at what time:
```bash
journalctl -u cromwell | grep "Starting workflow UUID" | perl -ne 'if ($_ =~ /(\w+\s+\d+\s+\d+\:\d+\:\d+).*UUID\((\S+)\)/){print "$2\tstarted $1\n"}'
```

### Save information about the workflow run itself - Timing Diagram and Outputs List

After a workflow is run, before exiting and deleting your VM, make sure that the timing diagram and the list of outputs are available so you can make use of the data outside of the cloud.

First determine you WORKFLOW_ID. This can be done several ways. If the run was successful it should be reported at the bottom of the cromwell log as "$WORKFLOW_ID  completed with status Succeeded". Or you find it by the name of the directory where your run was stored in the Google bucket. Both of these approaches are illustrated here:

```bash
export GCS_BUCKET_PATH=gs://griffith-lab-test-kartik
gsutil ls $GCS_BUCKET_PATH/cromwell-executions/immuno/

journalctl -u cromwell | tail | grep "Workflow actor"
```

Now save the workflow information in your google bucket. Include the YAML and WDLs used for this run for analysis provenance.
```bash
export WORKFLOW_ID=<id from above>
source /shared/helpers.sh
save_artifacts $WORKFLOW_ID $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
gsutil cp $GCS_BUCKET_PATH/yamls/$CLOUD_YAML $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/$CLOUD_YAML
gsutil cp /shared/analysis-wdls/workflows.zip $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/workflows.zip
gsutil cp /shared/analysis-wdls/definitions/$WORKFLOW $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW

```

This command will upload the workflow's artifacts to your google bucket so they can be used after the VM is deleted. They can be found at paths:

```bash
$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/timing.html
$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json
```

Confirm that they were successfully transferred and logout of the Cromwell VM on GCP:
```bash
gsutil ls $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
exit #leave the google VM
```

The file `outputs.json` will simply be a map of output names to their GCS locations. The `pull_outputs.py` script can be used to retrieve the actual files.


### Pulling the Outputs from Google Cloud Bucket back to your local system or cluster
After the work in your compute instance is all done, including `save_artifacts`, and you want to bring your results back to the cluster, leverage the `pull_outputs.py` script with the generated `outputs.json` to retrieve the files.

On compute1 cluster, jump into a docker container with the script available

```bash
export WORKFLOW_ID=<id from above>
bsub -Is -q general-interactive -G $GROUP -a "docker(mgibio/cloudize-workflow:latest)" /bin/bash
```

Execute the script:

```bash
cd $WORKING_BASE
mkdir final_results
cd final_results

python3 /opt/scripts/pull_outputs.py --outputs-file=$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/artifacts/outputs.json --outputs-dir=$WORKING_BASE/final_results/
exit #leave the docker session
```

Examine the outputs briefly:
```bash 
cd $WORKING_BASE/final_results
ls -l
du -h

```

### Store a local copy of workflow artifacts
```bash
export WORKFLOW_ID=<id from above>
bsub -Is -q general-interactive -G $GROUP -a "docker(mgibio/cloudize-workflow:latest)" /bin/bash

cd $WORKING_BASE/final_results/
mkdir workflow_artifacts
cd workflow_artifacts

gsutil cp -r  $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/* .

exit
```

### Estimate the cost of executing your workflow

On compute1 cluster, start an interactive docker session as describe below and then use the following python script to generate a cost estimate:

```bash
export WORKFLOW_ID=<id from above>
bsub -Is -q general-interactive -G $GROUP -a "docker(mgibio/cloudize-workflow:latest)" /bin/bash

cd $WORKING_BASE/final_results/workflow_artifacts
mkdir costs
cd costs

python3 $WORKING_BASE/git/cloud-workflows/scripts/estimate_billing.py $WORKFLOW_ID $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/artifacts/metadata/ > costs.json
python3 $WORKING_BASE/git/cloud-workflows/scripts/costs_json_to_csv.py costs.json > costs.csv
cat costs.csv | sed 's/,/\t/g' > costs.tsv

exit #leave the docker session

#### THE FOLLOWING IS NOT WORKING PROPERLY !!! ####
cd $WORKING_BASE/final_results/workflow_artifacts/costs/
cut -f 1 costs.tsv | perl -pe 's/_shard-\d+//g' | sort | uniq | while read i;do echo "$i     $(grep $i costs.tsv | cut -f 13 | awk '{ SUM += $1} END { print SUM}')";done | sed 's/ \+ /\t/g' > costs_total_condensed.tsv
cut -f 1 costs.tsv | perl -pe 's/_shard-\d+//g' | sort | uniq | while read i;do echo "$i     $(grep $i costs.tsv | cut -f 9 | awk '{ SUM += $1} END { print SUM}')";done | sed 's/ \+ /\t/g' > costs_memory_condensed.tsv
cut -f 1 costs.tsv | perl -pe 's/_shard-\d+//g' | sort | uniq | while read i;do echo "$i     $(grep $i costs.tsv | cut -f 10 | awk '{ SUM += $1} END { print SUM}')";done | sed 's/ \+ /\t/g' > costs_cpu_condensed.tsv
cut -f 1 costs.tsv | perl -pe 's/_shard-\d+//g' | sort | uniq | while read i;do echo "$i     $(grep $i costs.tsv | cut -f 11 | awk '{ SUM += $1} END { print SUM}')";done | sed 's/ \+ /\t/g' > costs_disk_condensed.tsv

paste costs_total_condensed.tsv costs_cpu_condensed.tsv costs_memory_condensed.tsv costs_disk_condensed.tsv | cut -f 1,2,4,6,8 > costs_report.tsv
echo -e "task\ttotal\tcpu\tmemory\tdisk" > header.tsv
sort -n -k 2 -r costs_report.tsv | cat header.tsv - > costs_report_final.tsv
rm -f costs_report.tsv header.tsv costs.csv

exit
```

### View high level cost details
```bash
cd $WORKING_BASE/final_results/workflow_artifacts/costs/

#Summarize overall costs
tail costs.json | head -n 9 | tr -d \" | tr -d "," | grep -v durationSeconds | grep -v Time

#List top 20 tasks by cost
head -n 21 costs_report_final.tsv | column -t

#Summarize tasks that were preempted
cat costs.tsv | grep -P -i "preempted|startTime" | cut -f 1,3,8,13 | column -t 

```

### Once the workflow is done and results retrieved, destroy the Cromwell VM on GCP to avoid wasting resources

```bash
gcloud compute instances delete $GCS_INSTANCE_NAME
```

Note that this does NOT empty your bucket. Once you are satisfied that your results are acceptable and you will no longer need to access the cromwell-executions files in your bucket you can clean it up to prevent billing for cloud bucket storage.

