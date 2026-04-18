
---

# UHGG Proteome Pipeline

A bash-based pipeline to parse the [Unified Human Gastrointestinal Genome (UHGG) database](https://www.nature.com/articles/s41587-020-0603-3), download reference proteomes for gut microbiome species using the NCBI Datasets CLI, and compile them into a unified FASTA database for downstream analysis (e.g., BLAST).

## 📋 Prerequisites

This pipeline requires `conda` to manage the environment and install dependencies.

```bash
# Create and activate environment
conda create -n pangenome_env -c conda-forge ncbi-datasets-cli unzip
conda activate pangenome_env
```

## 🚀 Workflow

### 1. Download UHGG Metadata
Retrieve the latest version of the UHGG metadata.
```bash
wget https://ftp.ebi.ac.uk/pub/databases/metagenomics/mgnify_genomes/human-gut/v2.0.2/genomes-all_metadata.tsv
```

### 2. Parse Species Names
Extract and clean the species names from the metadata file to create a target list for NCBI downloads.

```bash
mkdir -p logs

# Extract the species column (15th column) and remove header
tail -n +2 genomes-all_metadata.tsv | cut -f15 | sort | uniq > logs/clean_species_names_1.txt

# Remove the GTDB taxonomic prefix (s__)
awk -F';s__' '$2 != "" {print $2}' logs/clean_species_names_1.txt | sort | uniq > logs/clean_species_names_2.txt

# Remove unnamed species (e.g., " sp1", " sp2")
grep -v ' sp[0-9]' logs/clean_species_names_2.txt > logs/clean_species_names_3.txt

# Remove uppercase suffix modifiers (e.g., _A, _B)
sed -E 's/_[A-Z]+//g' logs/clean_species_names_3.txt | sort | uniq > species.txt
```

### 3. Download Proteomes
Execute the download script. This script reads `species.txt`, queries the NCBI Datasets API for the reference genome of each species, and downloads the associated protein datasets into a `downloads/` directory.

```bash
chmod +x download_proteomes.sh
./download_proteomes.sh
```
*Note: Progress and errors are tracked in the `logs/` directory.*

### 4. Extract and Combine Proteins
Once the `.zip` files are downloaded, extract all `.faa` (FASTA amino acid) files and combine them into a single `UHGG.fasta` database.

```bash
chmod +x extract_proteins.sh
./extract_proteins.sh
```

## 📊 Pipeline Stats (Example Run)
* **Total species queried:** 941
* **Successful downloads:** 862
* **Failed downloads:** 79 (Species lacking reference proteomes on NCBI)
* **Total protein sequences extracted:** ~2.4 million

## 📂 Project Structure



## 🛠️ Downstream Usage
The resulting `UHGG.fasta` can be used to build a local BLAST database:
```bash
makeblastdb -in blast/UHGG.fasta -dbtype prot -out blast/uhgg_db
blastp -query blast/queries.fasta -db blast/uhgg_db -outfmt 6 -out blast/results.tsv
```
