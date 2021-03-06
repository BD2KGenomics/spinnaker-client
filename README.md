# spinnaker-client (core-client)
Analysis Core Submission Client

## Introduction
The submission system client that uploads data/metadata to Redwood and indicates to the Spinnaker web service that a submission bundle is ready for validation.

## For Users
You'll need [Docker](https://docs.docker.com/engine/installation/) and a Redwood access token.

### Configuration
All users need to do this in order to use the client.

See the underlying [dcc-redwood-client docs](https://github.com/BD2KGenomics/dcc-redwood-client/tree/1.2.1#getting-started) for details on how to configure the client. In short:
```
wget https://raw.githubusercontent.com/BD2KGenomics/dcc-redwood-client/develop/src/main/conf/application-redwood.properties
```

Then edit the file to specify your Redwood access token.

### Upload Data
Create a `manifest.tsv` (or `manifest.xlsx`) like this example [spreadsheet](https://docs.google.com/spreadsheets/d/13fqil92C-Evi-4cy_GTnzNMmrD0ssuSCx3-cveZ4k70/edit?usp=sharing) or [tsv](manifests/three_manifest.tsv). See below for more details. Then run:

```
docker run -it -e REDWOOD_ENDPOINT=storage.ucsc-cgl.org -v $(pwd)/application-redwood.properties:/dcc/dcc-redwood-client/conf/application-redwood.properties -v $(pwd):/data -v $(pwd)/manifest.tsv:/manifests/manifest.tsv -v $(pwd)/outputs:/outputs quay.io/ucsc_cgl/core-client:1.1.1 spinnaker-upload /manifests/manifest.tsv
```

Make sure 'File Path' correctly locates the data where it's mounted into the docker container.

If you're re-uploading newer versions of files already tracked in the storage system, use command-line flag `--force-upload`.

Once upload completes, you will find a receipt file (`/outputs/receipt.tsv`) which you should save. It provides various IDs assigned to your donor, specimen, sample and file that make it much easier to find/audit later.

NOTE: Uploads can take a long time and our feedback on the command line needs to be improved. I suggest using a tool like `dstat` to monitor network usage to ensure uploads are in progress.

### Download Data (via Manifest)
This assumes the current working directory (`pwd`) has a manifest, like the ones you can download from http://ucsc-cgl.org/file_browser/.  The command below will then download the files to the current working directory.

NOTE: make sure you have enough space in `pwd`
```
docker run -e REDWOOD_ENDPOINT=storage.ucsc-cgl.org -v $(pwd)/application-redwood.properties:/dcc/dcc-redwood-client/conf/application-redwood.properties -v $(pwd):/data -v $(pwd)/outputs quay.io/ucsc_cgl/core-client:1.1.1 redwood-download /data/manifest.tsv /data
```

### Download Data (by id)

You can also download a single file by its unique id (not bundle id):
```
docker run -e REDWOOD_ENDPOINT=storage.ucsc-cgl.org -v $(pwd)/application-redwood.properties:/dcc/dcc-redwood-client/conf/application-redwood.properties -v $(pwd):/data quay.io/ucsc_cgl/core-client:1.1.1 download <object-id> /data
```

### Metadata Spreadsheet Column Details
The contents of your metadata spreadsheet will dictate how the analysis core handles your data (jobs run, etc.).  The full list of columns is:
- Program
- Project
- Center Name
- Submitter Donor ID
- Donor UUID
- Submitter Specimen ID
- Specimen UUID
- Submitter Specimen Type
- Submitter Experimental Design
- Submitter Sample ID
- Submitter Donor Primary Site
- Sample UUID
- Analysis Type
- Workflow Name
- Workflow Version 
- File Type
- File Path
- Upload File ID
- Data Bundle ID
- Metadata.json

To standardize our terminology to allow for automated processing we use the codelists (controlled vocabularies) from the ICGC.  
See http://docs.icgc.org/dictionary/viewer/#?viewMode=codelist

#### Program
e.g. TEST

#### Project
e.g. _DEV_

### Center Name
e.g. Stanford

#### Submitter Donor ID
Unique donor identifier as tracked by submitter.

#### Donor UUID
Should be left blank for first-time uploads. Will be populated by `spinnaker-upload`

#### Submitter Specimen ID
Unique specimen identifier as tracked by submitter.

#### Specimen UUID
Should be left blank for first-time uploads.

#### Submitter Specimen Type
* dna normal
* dna tumor
* rna tumor
* rna normal (rare)

And there are others as well but these are the major ones we'll encounter for now.

The actual values should come from the ICGC Codelist above.  Specifically the
`specimen.0.specimen_type.v3` codelist.

#### Submitter Experimental Design
* WXS
* WGS
* Gene Panel
* RNAseq

The actual values should come from the ICGC Codelist above.  Specifically the
`GLOBAL.0.sequencing_strategy.v1` codelist.

#### Submitter Sample ID
Unique sample identifier. Must be unique from Donor IDs and Specimen IDs.

#### Sample UUID
Should be left blank for first-time uploads.

#### Analysis Type
e.g.
* sequence_upload
* alignment
* germline_variant_calling
* somatic_variant_calling

#### Workflow Name
Name of the workflow to be run

#### Workflow Version
Version of the workflow to be run

#### File Type
* sequence/fastq
* sequence/unaligned BAM
* alignment/BAM & BAI pair
* expression/RSEM(?)
* variants/VCF

#### File Path
A path that must point to the data where it's mounted in to the docker container.

#### Upload File ID
Should be left blank for first-time uploads.

#### Data Bundle ID
Should be left blank for first-time uploads.

#### Metadata.json
Should be left blank for first-time uploads.

These will all come from the [EDAM Ontology](http://edamontology.org).  They have
a mechanism to add terms as needed.

#### Analysis Types
* germline_variant_calling -> normal specimen level
* rna_quantification (and various other RNASeq-based analysis) -> tumor specimen level
* somatic_variant_calling -> tumor specimen level (or donor if called simultaneously for multiple tumors)
* immuno_target_pipelines -> tumor specimen level

Unfortunately, the CVs from ICGC don't cover the above, see [here](http://docs.icgc.org/dictionary/viewer/#?viewMode=table).
Look for items like `variation_calling_algorithm` and you'll see they are actually just
TEXT with a regular expression to validate them.

Take home, I think we use our own CV for these terms and expand it over time here.

I think we also need to support tagging with multiple EDAM terms as well which can,
together, describe what I'm trying to capture above.  For example:

germline_variant_calling could be:

* [Variant calling](http://edamontology.org/operation_3227): http://edamontology.org/operation_3227

Which isn't very specific and the description sounds closer to somatic calling.

So this argues that we should actually just come up with our own specific terms
used for the institute since we aren't attempting to capture the whole world's
possible use cases here.

Over time I think this will expand.  Each are targeted at a distinct biospecimen "level".
This will need to be incorporated into changes to the index builder.

## For Developers
For developing the spinnaker client

### Overview
This repo contains several items relate to metadata JSONs used to describe biospecimen and analysis events for the core.

First, there are JSON schema, see `analysis_flattened.json` and `biospecimen_flattened.json`.

Second, this repo contains a `spinnaker.py` script that takes a TSV format and converts it into metadata JSON documents (and also has an option for uploading, we use this for bulk uploads to our system).

### Quick Tips
Build docker image with:
```
docker build -t quay.io/ucsc_cgl/core-client:dev
```

Run Redwood in dev mode via [dcc-ops](https://github.com/BD2KGenomics/dcc-ops) to test upload.

### Git Process
We use [HubFlow](https://datasift.github.io/gitflow/GitFlowForGitHub.html) for our feature branch/release process.

* `master` is the stable release branch
* `develop` is the unstable branch
* make features on feature branches
* candidate release branches are created right before a release

### Install

#### Ubuntu 14.04
You need to make sure you have system level dependencies installed in the appropriate way for your OS.  For Ubuntu 14.04 you do:

    sudo apt-get install python-dev libxml2-dev libxslt-dev lib32z1-dev python-setuptools build-essential

#### Python
Use python 2.7.x.

See [here](https://www.dabapps.com/blog/introduction-to-pip-and-virtualenv-python/) for information on setting up a virtual environment for Python.

If you haven't already installed pip and virtualenv, depending on your system you may
(or may not) need to use `sudo` for these:

    sudo easy_install pip
    sudo pip install virtualenv

Now to setup:

    virtualenv env
    source env/bin/activate
    pip install jsonschema jsonmerge openpyxl sets json-spec elasticsearch semver luigi

Alternatively, you may want to use Conda, see [here](http://conda.pydata.org/docs/_downloads/conda-pip-virtualenv-translator.html)
 [here](http://conda.pydata.org/docs/test-drive.html), and [here](http://kylepurdon.com/blog/using-continuum-analytics-conda-as-a-replacement-for-virtualenv-pyenv-and-more.html)
 for more information.

    conda create -n schemas-project python=2.7.11
    source activate schemas-project
    pip install jsonschema jsonmerge openpyxl sets json-spec elasticsearch semver luigi

### Generate Test Metadata (and Optionally Upload Data to Storage Service)
First, create a file for your access token e.g. `accessToken`.

We need to create a bunch of JSON documents for multiple donors and multiple
experimental designs and file upload types.  To do that we (Chris) developed a very simple
TSV to JSON tool and this will ultimately form the basis of our helper applications
that clients will use in the field to prepare their samples.

    python spinnaker.py \
		--input-metadata-schema schemas/input_metadata.json \
		--metadata-schema schemas/metadata_schema.json \
		--output-dir output_metadata \
		--receipt-file receipt.tsv \
		--storage-access-token `cat accessToken` \
		--skip-upload \
		sample_tsv/sample.tsv

  - `input_metadata.json` is a json schema used to do a very basic validation on input data.
  - `metadata_schema.json` is a json schema used to validate output metadata.json files.
  - `output_metadata` is the directory where the metadata files will be written.
  - `receipt.tsv` is the upload confirmation file where assigned UUIDs are recorded. Find it in `output_metadata` after a successful upload.

Take out `--skip-upload` if you want to perform upload, see below for more details.

In case there are already existing bundle ID's that cause a collision on the S3 storage, you can specify the `--force-upload` switch to replace colliding bundle ID's with the current uploading version.

Now look in the `output_metadata` directory for per-bundle directories that contain metadata files for each analysis workflow.

#### Enabling Upload
By default the upload won't take place if the directory `ucsc-storage-client` is not present in the `dcc-storage-schema`
directory.  In order to get the client, you need to be given an access key and download our client tarball.  See our public [S3 bucket](https://s3-us-west-2.amazonaws.com/beni-dcc-storage-dev/20161216_ucsc-storage-client.tar.gz)
for the tarball.

If you have the directory setup and don't pass in `--skip-upload` the upload will take place.  Keep this in
mind if you're just testing the metadata components and don't want to create a ton of uploads.  If you upload
the fact data linked to from the `sample.tsv` the program and project will both be TEST which should make
it easy to avoid in the future. The file is based on [this](https://docs.google.com/spreadsheets/d/13fqil92C-Evi-4cy_GTnzNMmrD0ssuSCx3-cveZ4k70/edit?usp=sharing) google doc.

**NOTE:** Here's a new template I'm working on that now includes tissue: [20170104 - Sample Upload Metadata Doc](https://docs.google.com/spreadsheets/d/1xuPS8ogphvvNrdJ8gRDlqFf-9fLQh9Pj1Zgx1mnUyeA/edit#gid=0)

And I think we should use [BRENDA](http://www.brenda-enzymes.org/ontology.php?ontology_id=3) for the tissue ontology.  E.g. prostate would be officially "prostate gland" which corresponds to BTO:0001129 

ICGC is using a limited list of hard-coded terms for primary tissues based on [this](http://docs.icgc.org/submission/projects/). So not super generic for what we’re doing.

**NOTE:** you may need to modify the storage and metadata service URLs used via the `--metadata-server-url` and `--storage-server-url` parameters if you are using a non-production storage system.

## Upload Simulation
Perform multiple uploads in order to generate data for testing.

### simulate_upload.py
This script runs an unlimited number of BAM file uploads at random intervals.  The script will run until killed.

    # FIXME
    cd luigi_task_executor
    python simulate_upload.py --bam-url https://s3.amazonaws.com/oconnor-test-bucket/sample-data/NA12878.chrom20.ILLUMINA.bwa.CEU.low_coverage.20121211.bam \
    --input-metadata-schema ../input_metadata.json --metadata-schema ../metadata_schema.json --output-dir output_metadata --receipt-file receipt.tsv \
    --storage-access-token `cat ../ucsc-storage2-client/accessToken` --metadata-server-url https://storage2.ucsc-cgl.org:8444 \
    --storage-server-url https://storage2.ucsc-cgl.org:5431  --ucsc-storage-client-path ../ucsc-storage2-client

Another script, this time it simulates the upload of fastq files:

    cd simulated_uploaders
    python simulate_upload_rnaseq_fastq.py --fastq-r1-path \
    https://s3.amazonaws.com/oconnor-test-bucket/sample-data/ERR030886_1.fastq.gz \
    --fastq-r2-path https://s3.amazonaws.com/oconnor-test-bucket/sample-data/ERR030886_2.fastq.gz \
    --input-metadata-schema ../schemas/input_metadata.json --metadata-schema ../schemas/metadata_schema.json \
    --output-dir output_metadata --receipt-file receipt.tsv \
    --storage-access-token `cat ../accessToken` --metadata-server-url https://storage2.ucsc-cgl.org:8444 \
    --storage-server-url https://storage2.ucsc-cgl.org:5431  --ucsc-storage-client-path ../ucsc-storage-client

## Data Types
We support the following types.  First and foremost, the types below are just intended
to be an overview. We need to standardize on actual acceptable terms. To do this
we use the Codelists (controlled vocabularies) from the ICGC.  See http://docs.icgc.org/dictionary/viewer/#?viewMode=codelist

In the future we will validate metadata JSON against these codelists via the Spinnaker service.

### Sample Types:
* dna normal
* dna tumor
* rna tumor
* rna normal (rare)

And there are others as well but these are the major ones we'll encounter for now.

The actual values should come from the ICGC Codelist above.  Specifically the
`specimen.0.specimen_type.v3` codelist.

### Experimental Design Types
* WXS
* WGS
* Gene Panel
* RNAseq

The actual values should come from the ICGC Codelist above.  Specifically the
`GLOBAL.0.sequencing_strategy.v1` codelist.

### File Types/Formats
* sequence/fastq
* sequence/unaligned BAM
* alignment/BAM & BAI pair
* expression/RSEM(?)
* variants/VCF

These will all come from the [EDAM Ontology](http://edamontology.org).  They have
a mechanism to add terms as needed.

### Analysis Types
* germline_variant_calling -> normal specimen level
* rna_quantification (and various other RNASeq-based analysis) -> tumor specimen level
* somatic_variant_calling -> tumor specimen level (or donor if called simultaneously for multiple tumors)
* immuno_target_pipelines -> tumor specimen level

Unfortunately, the CVs from ICGC don't cover the above, see [here](http://docs.icgc.org/dictionary/viewer/#?viewMode=table).
Look for items like `variation_calling_algorithm` and you'll see they are actually just
TEXT with a regular expression to validate them.

Take home, I think we use our own CV for these terms and expand it over time here.

I think we also need to support tagging with multiple EDAM terms as well which can,
together, describe what I'm trying to capture above.  For example:

germline_variant_calling could be:

* [Variant calling](http://edamontology.org/operation_3227): http://edamontology.org/operation_3227

Which isn't very specific and the description sounds closer to somatic calling.

So this argues that we should actually just come up with our own specific terms
used for the institute since we aren't attempting to capture the whole world's
possible use cases here.

Over time I think this will expand.  Each are targeted at a distinct biospecimen "level".
This will need to be incorporated into changes to the index builder.
