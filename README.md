# SVCall

SVCall is a scalable, parallel and efficient implementation of next generation sequencing data pre-processing and variant calling workflows. Our design tightly integrates most pre-processing workflow stages, using Spark built-in functions to sort reads by coordinates, and mark duplicates efficiently. A cluster scaled DeepVariant for both CPU-only and CPU+GPU clusters is also integrated in this workflow. 

## Apache Arrow Dependencies
 - [C++ libraries](https://github.com/apache/arrow/tree/master/cpp)
 - [C bindings using GLib](https://github.com/apache/arrow/tree/master/c_glib)
 - [Plasma Object Store](https://github.com/apache/arrow/tree/master/cpp/src/plasma): a
   shared-memory blob store, part of the C++ codebase
 - [Python libraries](https://github.com/apache/arrow/tree/master/python)

## Installation Requirements
 - Ubuntu 16.04/18.04
 - Apache Spark 3.0.1
 - [Singularity](https://sylabs.io/docs/) container
 
## Configuration & Build

1. Install [Singularity](https://sylabs.io/docs/) container
2. Download our Singularity [script](https://github.com/abs-tudelft/arrow-gen/tree/master/Singularity) and generate singularity image (this image contains all Arrow related packges necessary for building/compiling BWA-MEM)
3. Now enter into generated image using command:
         
        sudo singularity shell <image_name>.simg

## Custom Image creation on GCP DataProc Cluster:
- Create a bucket inside GCP [storage](https://console.cloud.google.com/storage) to store a custom image like `gs://{user}/images`
- Open https://console.cloud.google.com/ 
- Use “gcloud config set project [PROJECT_ID]” to change to a different project.
- Inside Cloud Shell run:

      git clone https://github.com/tahashmi/custom-images
      cd custom-images
      python3 generate_custom_image.py --image-name "bwa-custom" --dataproc-version "2.0.1-ubuntu18" --customization-script bwa_standalone.sh --zone "asia-east1-a" --gcs-bucket "gs://{user}/images" --shutdown-instance-timer-sec 50 --no-smoke-test
This will create a custom image which can be used on Google GCP DataProc cluster instances.

## Setting up GCP DataProc cluster:

    #On all master and worker nodes
    #***********************************************#
    sudo apt-get -y update && sudo apt-get install nfs-common
    sudo mkdir -p /mnt/fs_shared
    sudo mount 10.35.205.242:/fs_shared /mnt/fs_shared/
    sudo chmod go+rw /mnt/fs_shared/
    df -h --type=nfs

    #On any node 
    #***********************************************#
    mkdir -p /mnt/fs_shared/reference
    cd /mnt/fs_shared/reference
    # Download reference genome
    wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/001/405/GCA_000001405.15_GRCh38/seqs_for_alignment_pipelines.ucsc_ids/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz 
    gunzip GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz 
    mv GCA_000001405.15_GRCh38_no_alt_analysis_set.fna GRCh38.fa
    wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/001/405/GCA_000001405.15_GRCh38/seqs_for_alignment_pipelines.ucsc_ids/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.fai
    mv GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.fai GRCh38.fa.fai
    
    # Download query data
    mkdir -p /mnt/fs_shared/query/ERR001268
    cd /mnt/fs_shared/query/ERR001268
    wget ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/phase3/data/NA12878/sequence_read/ERR001268_1.filt.fastq.gz
    gunzip ERR001268_1.filt.fastq.gz
    mv ERR001268_1.filt.fastq ERR001268_1.fastq
    wget ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/phase3/data/NA12878/sequence_read/ERR001268_2.filt.fastq.gz
    gunzip ERR001268_2.filt.fastq.gz
    mv ERR001268_2.filt.fastq ERR001268_2.fastq

    #Create output dirs
    mkdir -p  /mnt/fs_shared/query/ERR001268/arrow
    sudo chmod go+rw  /mnt/fs_shared/query/ERR001268/arrow

    cd /mnt/fs_shared/query/ERR001268
    mkdir bams
    sudo chmod go+rw  bams/

    cd  /mnt/fs_shared/query/ERR001268/bams
    mkdir outputs
    sudo chmod go+rw  outputs/
    
    cd /mnt/fs_shared
    #Download vcfmerge script https://gist.github.com/tahashmi/84927410a47fd0b76a66228c1b37a744
    wget https://gist.github.com/tahashmi/84927410a47fd0b76a66228c1b37a744 
    sudo chmod +x /mnt/fs_shared/query/ERR001268/vcfmerge.sh

## Standalone pre-processing on clusters:
FASTQ data is streamed to BWA on every cluster node, BWA output is piped into Sambamba to perform sorting, duplicates removal option is also available, if enabled sorted data is piped to this stage as well. For final output, Samtools (merge) is used to produces a single BAM output, ready for further down stream analysis.
##### BWA (alignment) `->` Sambamba (Sorting) `->` Sambamba (Duplicates removal (optional)) `->` Samtools (merge BAMs)

1. Custom image is needed to be used on DataProc cluster, follow these [steps]() to create one.
2. Perfrom the following [steps](https://github.com/abs-tudelft/SVCall/blob/main/README.md#setting-up-gcp-dataproc-cluster) to make GCP DataProc cluster ready to run this workflow.
