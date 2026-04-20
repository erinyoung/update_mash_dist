# RefSeq Prokaryotic Mash Reference

[![Zenodo](https://zenodo.org/badge/DOI/10.5281/zenodo.19211013.svg)](https://zenodo.org/records/19211013)

This repository provides an automated, scalable pipeline to generate and distribute updated Mash sketch files for representative prokaryotic (bacterial and archaeal) genomes.

## Status: Automated Updates
This project was transitioned from a manual process to a high-concurrency GitHub Actions pipeline to bypass local storage limitations and keep pace with RefSeq's quarterly release cycle. The reference is currently tracking the most recent RefSeq release (v220+).

## Download
The final consolidated Mash sketches are available for download via Zenodo:
**[https://zenodo.org/records/19211013](https://zenodo.org/records/19211013)**

## Usage

The sketches are generated using **Mash v2.3**. To use the reference for genomic distance estimation:

### 1. Download and Decompress
```bash
wget https://zenodo.org/records/19211013/files/RefSeqSketches_latest.msh.gz
gunzip RefSeqSketches_latest.msh.gz
```

### 2. Estimate Distances

The `-p` flag enables multithreading, which is recommended when querying against a reference of this size. The output consists of five columns: `Query-ID, Reference-ID, Mash-distance, P-value, Matching-hashes`.

```bash
# Compare a query fasta against the full RefSeq reference
mash dist -p 8 RefSeqSketches_latest.msh query_genome.fasta > distances.tab
```
### 3. Parse Results

The results should be sorted by the third column (Mash distance). A distance of 0 indicates a perfect k-mer match, while a distance of 1 indicates no shared k-mers.

```bash
# Sort by distance (column 3) to find the closest taxonomic matches
sort -gk3 distances.tab | head -n 10
```
---

## Replication Logic

The automated pipeline consists of three distinct stages within GitHub Actions to maintain data integrity while operating within infrastructure limits.

### Stage 1: ID Generation
The NCBI Datasets CLI is used to query all representative prokaryotic accessions. This list is cleaned and split into 100 chunks to facilitate parallel processing across a job matrix.

```bash
# download datasets and dataformat
wget -q https://ftp.ncbi.nlm.nih.gov/pub/datasets/command-line/v2/linux-amd64/datasets
wget -q https://ftp.ncbi.nlm.nih.gov/pub/datasets/command-line/v2/linux-amd64/dataformat
chmod +x datasets dataformat

# get a list of ids
./datasets summary genome taxon 2 --reference --as-json-lines | \
  ./dataformat tsv genome --fields accession,organism-name --elide-header | \
  sed 's/\[//g; s/\]//g; s/["'\'']//g; s/endosymbiont of /endosymbiont_of_/g' | \
  grep ^"G" > ids.txt       

# used by github actions, but not needed if running locally
# this splits ids.txt into 100 files so that download can run concurrently.
split -n l/100 ids.txt -d -a 3 --additional-suffix=.txt x
```

### Stage 2. Execution Loop
The following loop ensures that storage usage remains low by cleaning the workspace after every iteration.

```bash
# getting the RefSeq version
version=$(curl -s https://ftp.ncbi.nlm.nih.gov/refseq/release/RELEASE_NUMBER)

# cycling through ids individually for download
while read -r line <&3; do
  id=$(echo "$line" | awk '{print $1}')
  ge=$(echo "$line" | awk '{print $2}')
  sp=$(echo "$line" | awk '{print $3}')
  
  [ -z "$ge" ] && ge="unknown"
  [ -z "$sp" ] && sp="unknown"
  [[ ! "$id" =~ ^G ]] && continue

  # iteration cleanup
  rm -f ncbi_dataset.zip md5sum.txt README.md
  rm -rf ncbi_dataset/

  if ./datasets download genome accession "$id" --no-progressbar --filename "ncbi_dataset.zip" < /dev/null; then
    unzip -qo ncbi_dataset.zip < /dev/null
    GENOME_PATH=$(find ncbi_dataset/data -name "*_genomic.fna" | head -n 1)
    
    if [ -n "$GENOME_PATH" ]; then
      if [ ! -f "RefSeqSketches_${version}.msh" ]; then
        ./mash sketch "$GENOME_PATH" -o "RefSeqSketches_${version}" -I "${ge}_${sp}_${id}"
      else
        ./mash sketch "$GENOME_PATH" -o single_sample -I "${ge}_${sp}_${id}"
        ./mash paste new_combined "RefSeqSketches_${version}.msh" single_sample.msh
        mv new_combined.msh "RefSeqSketches_${version}.msh"
        rm -f single_sample.msh
      fi
    fi
  fi
done 3< ids.txt
```

### 3. Final Compression

```bash
gzip RefSeqSketches_${version}.msh
```
