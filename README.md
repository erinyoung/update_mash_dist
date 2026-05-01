# RefSeq Prokaryotic Mash Reference

[![Zenodo](https://zenodo.org/badge/DOI/10.5281/zenodo.19211013.svg)](https://zenodo.org/records/19211013)

This repository provides an automated, high-concurrency pipeline for generating and distributing [Mash](https://mash.readthedocs.io/en/latest/tutorials.html) sketch files for all representative prokaryotic genomes in RefSeq. 

## The Automated Pipeline
To bypass GitHub Action storage limits and execution timeouts, the process is split into three sequential stages:

1. ID Generation : Queries NCBI for the latest representative accessions and splits them into 100 manageable chunks.
2. Distributed Sketching : Uses a job matrix to process all 100 chunks in parallel. Each job downloads genomes, generates sketches via Mash, and verifies integrity.
3. Consolidation & Release : Merges all chunked sketches into a single master reference, verifies the final accession count, updates the repository via Pull Request, and publishes the new version to Zenodo.

---

## Download
The consolidated reference is updated following RefSeq releases and is available on Zenodo:

[Download Latest RefSeq Mash Sketches](https://zenodo.org/records/19211013)

---

## Usage
Sketches are generated using Mash v2.3. Mash must be installed in the environment to use these files.

### 1. Setup
```bash
wget https://zenodo.org/records/19211013/files/RefSeqSketches_latest.msh.gz
gunzip RefSeqSketches_latest.msh.gz
```
### 2. Fast Sequence Screening
Use mash screen to identify which RefSeq genomes are present in a raw sequencing read set (FastQ) or assembly (Fasta).
```bash
# -p 8 enables multithreading
mash screen -p 8 RefSeqSketches_latest.msh query_genome.fasta | sort -gr | head -n 10
```
### 3. Genomic Distance Estimation
Use mash dist to calculate Mash distances (an approximation of ANI) between your query and the entire reference set.
```bash
mash dist -p 8 RefSeqSketches_latest.msh query_genome.fasta > distances.tab

# Sort by distance (Column 3). 0.0 = identical; 1.0 = no similarity.
sort -gk3 distances.tab | head -n 10
```
---

## Pipeline Logic (for Replication)

The logic follows these core bash routines.

### Stage 1: Taxonomy Filtering
[NCBI Datasets CLI](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/command-line-tools/download-and-install/) extracts representative genomes, specifically cleaning names to ensure compatibility with Mash headers.

```bash
./datasets summary genome taxon 2 --reference --as-json-lines | \
  ./dataformat tsv genome --fields accession,organism-name --elide-header | \
  sed 's/\[//g; s/\]//g; s/["'\'']//g; s/endosymbiont of /endosymbiont_of_/g' | \
  grep ^"G" > ids.txt
```

### Stage 2: Download Genomes

This repository uses GitHub accessions to split ids.txt into several files, which are processed in parallel. This is what it would behave like if not split.

```bash
cut -f 1 ids.txt > accessions.txt
datasets download genome accession --no-progressbar --inputfile accessions.txt --dehydrated --filename genomes.zip
unzip -n genomes.zip -d genomes
# dehydrate and rehydrate is more stable
datasets rehydrate --no-progressbar --directory genomes
```

### Stage 3: Add mash sketches

This repository uses GitHub accessions to split ids.txt into several files, which are processed in parallel. This is what it would behave like if not split.

```bash
ORIG_WORKSPACE=$(pwd)
DATASET_WORKSPACE=$(pwd)/tmp_datasets_workspace
CHUNK_FILE="ids.txt"

cd $ORIG_WORKSPACE
mkdir -p tmp_mash_workspace tmp_datasets_workspace

# getting absolute paths for mash files
NEW_MASH=$ORIG_WORKSPACE/new
TEMP_MASH=$ORIG_WORKSPACE/temp
FINAL_MASH=$ORIG_WORKSPACE/fin
         
while read line
do
  cd $ORIG_WORKSPACE
  rm -rf $DATASET_WORKSPACE/*
            
  id=$(echo "$line" | awk '{print $1}')
  ge=$(echo "$line" | awk '{print $2}')
  sp=$(echo "$line" | awk '{print $3}')

  # getting values for line
  echo "Looking for reference for"
  echo "Genome accession: $id"
  echo "Genus: $ge"
  echo "Species: $sp"

  # getting approximate line number for sanity
  wc -l $CHUNK_FILE
  grep -n $id $CHUNK_FILE

  GENOME_PATH=""
  # first looking to see if it downloaded in the batch (common)
  if [ -d "genomes/ncbi_dataset/data/${id}" ]
  then
    GENOME_PATH=$(find genomes/ncbi_dataset/data/${id} -name "*_genomic.fna" | head -n 1)
  fi

  # if didn't download as batch, try again as individual
  if [ ! -n "$GENOME_PATH" ]
  then
    cd $DATASET_WORKSPACE
    rm -rf $DATASET_WORKSPACE/*
             
    MAX_RETRIES=10
    attempt=0
    success=false
    echo "Accession $id was not found. Re-attempting download."
              
    until [ $attempt -ge $MAX_RETRIES ]
    do
      echo "Downloading attempt number $attempt"
      datasets download genome accession "${id}" --no-progressbar --no-progressbar --filename ncbi_dataset.zip < /dev/null
      if [ -f ncbi_dataset.zip ]
      then
        unzip ncbi_dataset.zip
        GENOME_PATH=$(find ncbi_dataset/data -name "*_genomic.fna" | head -n 1)
      fi
                
      if [ -n "$GENOME_PATH" ]
      then
        success=true
        break
      else
        rm -rf $DATASET_WORKSPACE/*
        attempt=$((attempt+1))
        sleep $((attempt * 10))s
      fi
    done
  fi

  if [ -n "$GENOME_PATH" ]
  then
    if [ ! -f "$FINAL_MASH.msh" ]
    then
      mash sketch "$GENOME_PATH" -o $FINAL_MASH -I "${ge}_${sp}_${id}"
    else
      mash sketch "$GENOME_PATH" -o $NEW_MASH -I "${ge}_${sp}_${id}"
      mash paste $TEMP_MASH $FINAL_MASH.msh $NEW_MASH.msh
      mv $TEMP_MASH.msh $FINAL_MASH.msh
      rm $NEW_MASH.msh
    fi
  else
    echo "Could not download the genome for $id"
    echo "FAILURE"
    # comment out the exit to press on
    exit 1
  fi
done < $CHUNK_FILE
```

### Step 4. Verification
```bash
mash info -t fin.msh | cut -f 3 | grep G | rev | cut -f 1-2 -d _ | rev | sort | uniq > final_accessions.txt
cat ids.txt | grep -vf final_accessions.txt > missing_accessions.txt
          
ORIGINAL_COUNT=$(wc -l < ids.txt)
MASH_COUNT=$(wc -l < final_accessions.txt)
MISSING_COUNT=$(wc -l < missing_accessions.txt)
          
echo "- **Total Accessions Expected:** $ORIGINAL_COUNT"
echo "- **Total Accessions Sketched:** $MASH_COUNT"
echo "- **Missing Accessions:** $MISSING_COUNT"
```


## Testing

```bash
wget -q --no-check-certificate https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/019/048/245/GCF_019048245.1_ASM1904824v1/GCF_019048245.1_ASM1904824v1_genomic.fna.gz
gunzip GCF_019048245.1_ASM1904824v1_genomic.fna.gz
mash screen -p 4 ${{ matrix.chunk }}.msh GCF_019048245.1_ASM1904824v1_genomic.fna | sort -gr > test.txt
head test.txt
```



---

## Maintenance
* Trigger: The workflow is currently set to workflow_dispatch but can be automated to track the RefSeq release cycle.
* Infrastructure: Runs on ubuntu-latest GitHub runners.
* Storage: Large artifacts are stored in Zenodo; only the master ID list and compressed metadata are kept in this repository.
