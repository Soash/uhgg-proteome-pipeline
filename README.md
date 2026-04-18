
---

# UHGG Proteome Pipeline

A bash-based pipeline to parse the [Unified Human Gastrointestinal Genome (UHGG) database v2.0.2](https://www.nature.com/articles/s41587-020-0603-3), download reference proteomes for gut microbiome species using the NCBI Datasets CLI, and compile them into a unified FASTA database for downstream analysis (e.g., BLAST, DIAMOND).

## 📋 Prerequisites

This pipeline requires `conda` to manage the environment and install dependencies.

```bash
# Create and activate environment
conda create -n UHGG_pp_env -c conda-forge ncbi-datasets-cli unzip
conda activate UHGG_pp_env

# (Optional but recommended) Ensure you have the latest NCBI Datasets CLI
conda update -c conda-forge ncbi-datasets-cli
```

## 🚀 Workflow

### 1. Download UHGG Metadata
Retrieve the v2.0.2 UHGG metadata containing **289,231** human gut genomes.

```bash
wget https://ftp.ebi.ac.uk/pub/databases/metagenomics/mgnify_genomes/human-gut/v2.0.2/genomes-all_metadata.tsv
```
```bash
tail -n +2 genomes-all_metadata.tsv | wc -l # 289,231
```

### 2. Parse Species Names
Extract and clean the species names from the metadata file to create a targeted, deduplicated list for NCBI downloads.

```bash
mkdir -p logs

# Extract the species column (15th column) and remove header (Yields 3,910 entries)
tail -n +2 genomes-all_metadata.tsv | cut -f15 | sort | uniq > logs/clean_species_names_1.txt

# Remove the GTDB taxonomic prefix 's__' (Yields 3,443 entries)
awk -F';s__' '$2 != "" {print $2}' logs/clean_species_names_1.txt | sort | uniq > logs/clean_species_names_2.txt

# Remove unnamed/sp. placeholders like " sp1", " sp2" (Yields 1,057 entries)
grep -v ' sp[0-9]' logs/clean_species_names_2.txt > logs/clean_species_names_3.txt

# Remove uppercase suffix modifiers like _A, _B (Yields 941 species)
sed -E 's/_[A-Z]+//g' logs/clean_species_names_3.txt | sort | uniq > species.txt
```
```bash
wc -l logs/clean_species_names_1.txt # 3910
wc -l logs/clean_species_names_2.txt # 3443
wc -l logs/clean_species_names_3.txt # 1057
wc -l species.txt # 941
```

### 3. Download Proteomes
Execute the download script. This script reads `species.txt`, queries the NCBI Datasets API for the reference genome of each species, and downloads the associated protein `.zip` packages into the `downloads/` directory.

```bash
chmod +x download_proteomes.sh
./download_proteomes.sh
```
*Note: Progress and errors are tracked in the `logs/` directory.*

```bash
wc -l logs/download_failed.txt # 79
ls downloads | wc -l # 862
```

### 4. Extract and Combine Proteins
Extract all `.faa` (FASTA amino acid) files from the downloaded archives and merge them into a single `UHGG.fasta` database. To save space, we compress the final file.

```bash
chmod +x extract_proteins.sh
./extract_proteins.sh

# 2401130
grep -c "^>" UHGG.fasta

# Compress the final 2.4M+ sequence database
gzip UHGG.fasta
```

## 📊 Pipeline Stats (18 April, 2026)
* **Total Genomes in UHGG:** 289,231
* **Unique Raw Species (GTDB):** 3,910
* **Cleaned Target Species:** 941
* **Successful Downloads:** 862
* **Failed Downloads:** 79 *(Species lacking reference proteomes on NCBI)*
* **Total Protein Sequences Extracted:** 2,401,130

## 📂 Project Structure

```text
.
├── download_proteomes.sh
├── downloads/                  # Zipped genome packages from NCBI
├── extract_proteins.sh
├── genomes-all_metadata.tsv    # Raw UHGG metadata
├── logs/                       # Tracking success/fails & parsing stages
├── README.md               
├── species.txt                 # Cleaned target list of species
├── UHGG.fasta                  # Extracted FASTA database (if unzipped)
└── UHGG.fasta.gz               # Compressed final FASTA database
```

## 🛠️ Downstream Usage

The resulting `UHGG.fasta.gz` can be used for local alignment searches. Depending on your tool of choice, you can use standard BLAST or a faster alternative like DIAMOND (which natively supports `.gz` inputs).

### Option A: Using DIAMOND (Recommended for large databases)
DIAMOND is highly recommended for a database of this size (~2.4M sequences). It is significantly faster than standard BLAST and allows you to build the database directly from the compressed file.

```bash
# Build DIAMOND database directly from the .gz file
mkdir -p blast

diamond makedb --in UHGG.fasta.gz -d blast/uhgg_db

# Run alignment query
diamond blastp -q blast/queries.fasta -d blast/uhgg_db -o blast/results.tsv -f 6
```

### Option B: Using standard BLAST+
If you prefer standard NCBI BLAST, you must uncompress the database first.

```bash
mkdir -p blast

# Unzip for standard BLAST
gunzip -c UHGG.fasta.gz > blast/UHGG.fasta

# Build BLAST database
makeblastdb -in blast/UHGG.fasta -dbtype prot -out blast/uhgg_db

# Run alignment query
blastp -query blast/queries.fasta -db blast/uhgg_db -outfmt 6 -out blast/results.tsv
```

