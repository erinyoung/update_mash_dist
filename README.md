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
This repository uses GitHub accessions to split ids.txt into several files, which are processed in parallel.
```bash
cut -f 1 ids.txt > accessions.txt
datasets download genome accession --no-progressbar --inputfile accessions.txt --dehydrated --filename genomes.zip
unzip -n genomes.zip -d genomes
# dehydrate and rehydrate is more stable
datasets rehydrate --no-progressbar --directory genomes
```

### Stage 3: Add mash sketches
```bash
NEW_MASH=new
TEMP_MASH=temp
FINAL_MASH=final

while read line
do
  id=$(echo "$line" | awk '{print $1}')
  ge=$(echo "$line" | awk '{print $2}')
  sp=$(echo "$line" | awk '{print $3}')
  GENOME_PATH=$(find genomes/ncbi_dataset/data/${id} -name "*_genomic.fna" | head -n 1)
  if [ -n "$GENOME_PATH" ]
  then
    if [ ! -f "$FINAL_MASH.msh" ]
    then
      mash sketch "$GENOME_PATH" -o $FINAL_MASH -I "${ge}_${sp}_${id}"
    fi
  else
    mash sketch "$GENOME_PATH" -o $NEW_MASH -I "${ge}_${sp}_${id}"
    mash paste $TEMP_MASH $FINAL_MASH.msh $NEW_MASH.msh
    mv $TEMP_MASH.msh $FINAL_MASH.msh
    rm $NEW_MASH.msh
  fi
done < ids.txt
```
Since the sketches are split and done in parallel, this repository will then paste the split msh into one final file.

---

## Maintenance
* Trigger: The workflow is currently set to workflow_dispatch but can be automated to track the RefSeq release cycle.
* Infrastructure: Runs on ubuntu-latest GitHub runners.
* Storage: Large artifacts are stored in Zenodo; only the master ID list and compressed metadata are kept in this repository.
