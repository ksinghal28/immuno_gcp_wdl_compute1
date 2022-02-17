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
export PROJECT=griffith-lab
export GCS_BUCKET_NAME=griffith-lab-test-immuno-pipeline
export GCS_BUCKET_PATH=gs://griffith-lab-test-immuno-pipeline
export WORKING_BASE=/storage1/fs1/mgriffit/Active/griffithlab/gcp_wdl_test
```

## Local setup

### First create a working directory on your local system
The following directory on the local system will contain: (a) a git repository for this tutorial including an example YAML file set up to work with the test HCC139 data, (b) git repository for the WDL workflows, including the immuno worflow, (c) git repository for tools that help provision and manage our workflow runs on the cloud, (d) raw data that we will download for this tutorial, (e) a YAML file describing the input data and paramters for the analysis, and (f) final results file from the workflow that we will pull down from the cloud after a successful run.
```bash
mkdir $WORKING_BASE
cd $WORKING_BASE
```

### Clone git repositories that have the workflows (pipelines) and scripts to help run them
The following repositories contain: this tutorial (immuno_gcp_wdl), the WDL workflows (analysis-wdls), and tools for running these on the cloud (cloud-workflows).
```bash
mkdir git
cd git
git clone git@github.com:griffithlab/immuno_gcp_wdl_compute1.git
git clone git@github.com:griffithlab/analysis-wdls.git
git clone git@github.com:griffithlab/cloud-workflows.git
```

### Login to GCP and set the desired project
From the command line, you will need to authenticate your cloud access (using your google cloud account credentials). This generally only needs to be done once, though there is no harm to re-authenticating. The login command below will generate a custom URL to enter in your browser. Once you do this, you will be prompted to log into your Google account. If you have multiple Google accounts (e.g. one for your institution/company and a personal one) be sure to use the correct one.  Once you have logged in you will be presented with a long code. Enter this at the prompt generated by the login command below. Finally, set your desired Google Project. This Project should correspond to a Google Billing account in the Google console. If you are using Google Cloud for the first time, both billing and a project should be set up before proceeding. Configuring billing alerts is also probably wise at this point.
```bash
gcloud auth login
gcloud config set project $PROJECT
```

### Set up cloud account and bucket
Run the following command and make note of the "Service Account" returned (e.g. "cromwell-server@griffith-lab.iam.gserviceaccount.com").

```bash
cd $WORKING_BASE/git/cloud-workflows/manual-workflows/
bash resources.sh init-project --project griffith-lab --bucket griffith-lab-test-immuno-pipeline
```

This step should have created two new configuration files in your current directory: `cromwell.conf` and `workflow_options.json`.

### Gather input data and reference files to your local system
Create a directory for YAML files and create one for the desired pipeline that points to the location of input files on your local system

```bash
cd $WORKING_BASE
mkdir yamls
cd yamls
```

### Setup input data and reference files

Download RAW data for hcc1395
```bash
mkdir -p $WORKING_BASE/raw_data/hcc1395
cd $WORKING_BASE/raw_data/hcc1395
wget http://genomedata.org/pmbio-workshop/fastqs/all/Exome_Norm.tar
wget http://genomedata.org/pmbio-workshop/fastqs/all/Exome_Tumor.tar
wget http://genomedata.org/pmbio-workshop/fastqs/all/RNAseq_Tumor.tar
tar -xvf Exome_Norm.tar Exome_Tumor.tar RNAseq_Tumor.tar
rm -f Exome_Norm.tar Exome_Tumor.tar RNAseq_Tumor.tar
```


Setup yaml files for an example run.
```bash
cp $WORKING_BASE/git/immuno_gcp_wdl_compute1/example_yamls//hcc1395_immuno_local.yaml $WORKING_BASE/yamls/
```

Note that this YAML file has been set up to work with the HCC1395 raw data files downloaded above. If you are modifying this tutorial to work with your own data, you will need to modify the beginning of the YAML that relates to input sequence files.  For both DNA and RNA files, both FASTQ and Unaligned BAM files are supported as input.  Similarly, you have have your data in one file (or one file pair) or you may have multiple data files that will be merged together. Depending on how your input data is organized the YAML entries will look slightly different.

### Stage input files to cloud bucket

Start an interactive docker session capable of running the "cloudize" scripts
```bash
bsub -Is -q oncology-interactive -G $GROUP -a "docker(jackmaruska/cloudize-workflow:latest)" /bin/bash
```

Attempt to cloudize your workflow and inputs
```bash
export WORKFLOW_DEFINITION=$WORKING_BASE/git/analysis-wdls/definitions/immuno.wdl
export LOCAL_YAML=hcc1395_immuno_local.yaml
export CLOUD_YAML=hcc1395_immuno_cloud.yaml
python3 /opt/scripts/cloudize-workflow.py $GCS_BUCKET_NAME $WORKFLOW_DEFINITION $WORKING_BASE/yamls/$LOCAL_YAML --output=$WORKING_BASE/yamls/$CLOUD_YAML
```

### Start a Google VM that will run cromwell and orchestrate completion of the workflow

```bash
export INSTANCE_NAME=mg-immuno-test
export SERVER_ACCOUNT=cromwell-server@griffith-lab.iam.gserviceaccount.com

cd $WORKING_BASE/git/cloud-workflows/manual-workflows/
bash start.sh $INSTANCE_NAME --server-account $SERVER_ACCOUNT
```

### Log into the VM and check status 

After logging in, use journalctl to see if the instance start up has completed, and cromwell launch has completed.

For details on how to recognize whether these processes have completed refer: [here](https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows#ssh-in-to-vm).

```bash
gcloud compute ssh $INSTANCE_NAME
journalctl -u google-startup-scripts -f
journalctl -u cromwell -f
exit
```

### Localize your inputs file

First **on your local system**, copy your cloudized YAML file to a google bucket

```bash
cd $WORKING_BASE/yamls/
gsutil cp $CLOUD_YAML $GCS_BUCKET_PATH/yamls/$CLOUD_YAML
```

Now log into Google instance again and copy the YAML file to its local file system
```bash
gcloud compute ssh $INSTANCE_NAME

export GCS_BUCKET_PATH=gs://griffith-lab-test-immuno-pipeline
export CLOUD_YAML=hcc1395_immuno_cloud.yaml

gsutil cp $GCS_BUCKET_PATH/yamls/$CLOUD_YAML .

``` 

### Run the immuno workflow using everything setup thus far

While logged into the google instance:
```bash
source /shared/helpers.sh
submit_workflow /shared/analysis-wdls/definitions/immuno.wdl $CLOUD_YAML

```

### Monitor progress of the workflow run:

While the job is running you can see Cromwell logs live as they occur by doing this
```bash
journalctl -f -u cromwell

```

### Save information about the workflow run itself - Timing Diagram and Outputs List

After a workflow is run, before exiting and deleting your VM, make sure that the timing diagram and the list of outputs are available so you can make use of the data outside of the cloud.

First determine you WORKFLOW_ID. This can be done several ways. If the run was successful it should be reported at the bottom of the cromwell log as "$WORKFLOW_ID  completed with status Succeeded". Or you find it by the name of the directory where your run was stored in the Google bucket. Both of these approaches are illustrated here:

```bash
export GCS_BUCKET_PATH=gs://griffith-lab-test-immuno-pipeline
gsutil ls $GCS_BUCKET_PATH/cromwell-executions/immuno/

journalctl -u cromwell | tail | grep "Workflow actor"
```

Now save the workflow information in your google bucket
```bash
export WORKFLOW_ID=<id from above>
source /shared/helpers.sh
save_artifacts $WORKFLOW_ID $GCS_BUCKET_PATH/workflow_artifacts
```

This command will upload the workflow's artifacts to your google bucketS so they can be used after the VM is deleted. They can be found at paths:

```bash
gs://$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/timing.html
gs://$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json
```

Confirm that they were successfully transferred:
```bash
gsutil ls $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
```

The file `outputs.json` will simply be a map of output names to their GCS locations. The `pull_outputs.py` script can be used to retrieve the actual files.


### Pulling the Outputs from Google Cloud Bucket back to your local system or cluster
After the work in your compute instance is all done, including `save_artifacts`, and you want to bring your results back to the cluster, leverage the `pull_outputs.py` script with the generated `outputs.json` to retrieve the files.

On compute1 cluster, jump into a docker container with the script available

```bash
bsub -Is -q general-interactive -G $GROUP -a "docker(jackmaruska/cloudize-workflow:latest)" /bin/bash
```

Execute the script

```bash
export WORKFLOW_ID=<id from above>
cd $WORKING_BASE
mkdir final_results
cd final_results

python3 /opt/scripts/pull_outputs.py --outputs-file=$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json --outputs-dir=$WORKING_BASE/final_results/
```

### Once the workflow is done and results retrieved, destroy the Cromwell VM on GCP to avoid wasting resources

```bash
gcloud compute instances delete $INSTANCE_NAME
```

