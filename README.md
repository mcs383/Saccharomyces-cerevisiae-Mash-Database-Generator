# ***Saccharomyces cerevisiae*** **Mash Database Generator**

## **An automated GitHub Actions workflow for creating up-to-date *Saccharomyces cerevisiae* mash sketch databases for strain analysis**

This repository contains a GitHub Actions workflow designed to automatically build a comprehensive and up-to-date Mash sketch database from a manually curated list of genomes combined with all publically available *S. cerevisiae* and outgroup reference genomes in NCBI. This project was created to provide a fully automated, cloud-based method to generate a fresh Mash sketch database for *S. cerevisiae* strain analysis on demand, without requiring massive local computation or storage resources.  
The workflow was adapted from the [Fungal-RefSeq-Mash-Database-Generator](https://github.com/mcs383/Fungal-RefSeq-Mash-Database-Generator) and the original work by [erinyoung/update_mash_dist](https://github.com/erinyoung/update_mash_dist) to support a yeast genomics project as a reusable resource for the yeast community.

## **How it Works**

This repository uses a single workflow (`.github/workflows/building_sc_mash_sketch.yaml`) that automates the entire process using GitHub's own servers. The workflow consists of three main jobs that run in sequence:

1. **Job 1: `list_genomes`**
   * Reads a **manually** provided `sc_ids.tsv` file from the repository containing a list of curated genome accession numbers.  
   * Connects to the NCBI database and downloads a complete list of accession numbers for all currently available *S. cerevisiae* genomes (**NCBI `taxon 4932`**) and *S. paradoxus* genomes (**NCBI `taxon 29763`**) as an outgroup.  
   * **Combines these lists and removes any duplicates** to create a final master list.  
   * Splits this master list into 10 smaller "chunk" files to enable parallel processing and uploads them as an artifact for the next job.  
2. **Job 2: `create_sketches`**
   * This is a parallel matrix job that runs 10 jobs simultaneously.  
   * Each job is assigned one of the "chunk" files.  
   * It processes its list one genome at a time in a disk-efficient loop: it downloads a single genome, creates a Mash sketch, merges it into a combined sketch for the chunk, and then deletes the downloaded files before starting the next one. This is crucial for running within the disk space limits of the GitHub servers.  
   * Each job uploads its completed partial sketch as an artifact.  
3. **Job 3: `combine_sketches`**
   * This final job runs only after all 10 sketching jobs are complete.  
   * It downloads all 10 of the partial sketch artifacts and the final `ids.tsv` list.  
   * It uses `mash paste` to merge them into one final, comprehensive database sketch file.  
   * It then compresses the final sketch and creates a formal **GitHub Release**, attaching both the sketch file and the complete `ids.tsv` list for download.

## **How to Use This Workflow**

1. **Fork this Repository:** Create your own copy of this repository by clicking the "Fork" button.  
2. **Prepare Your Manual List (Optional):** Create or edit the `sc_ids.tsv` file in this repository. Add the NCBI accession numbers and tab-separated names for any specific public strains you want to include (one per line). Commit this change.  
3. **Enable Actions:** Go to the "Actions" tab of your newly forked repository. You may need to click a button to enable workflows.  
4. **Run the Workflow:**  
   * In the Actions tab, select **"Build a mash sketch database..."** from the list of workflows on the left.  
   * Click the **"Run workflow"** button on the right.  
   * You will be prompted to enter a **version tag** (e.g., `v1.0`). This tag must not contain spaces.  
   * Click the final green **"Run workflow"** button.  
5. **Wait:** The process will take several hours to complete. You can monitor its progress on the "Actions" tab at any time.  
6. **Download Your Database:** When the workflow is complete (it will have a green checkmark ✅), go to the main page of your repository. On the right-hand side, click on **"Releases"**. Your new database will be there, attached as a `.msh.gz` file and an `ids.tsv` file containing the list of genome sketches in the database, ready to download.

## **How to Include Proprietary (Private) Genomes**

This workflow is designed to download public genomes from NCBI. To include your own in-house or proprietary genomes without making them public, use the following secure, two-stage "hybrid" method.

#### **Stage 1: Build the Public Database Using GitHub**

Run the GitHub Actions workflow as described above. Download the final `.msh.gz` sketch file from the "Releases" page to your local computer and unzip it.

#### **Stage 2: Add Your Private Genomes Locally**

On your own computer, you will create a sketch of your private genomes and "paste" it into the public database you just downloaded.

1. **Organize your files:** Place all your private `.fasta` genome files into a single folder (e.g., `proprietary_genomes/`).  
2. **Run this local script:** This script sketches your private genomes and merges them with the public database.

```
#!/bin/bash  
# --- CONFIGURATION ---  
PUBLIC_DB="SCerevisiae_RefSeq_v1.0.msh" # Change this to your downloaded file name  
PROPRIETARY_DIR="proprietary_genomes"  
FINAL_DB="Final_Combined_Database.msh"

# --- SCRIPT START ---  
echo "1. Sketching proprietary genomes..."  
mash sketch -o proprietary.msh "${PROPRIETARY_DIR}"/*.fasta

echo "2. Pasting proprietary sketch into the public database..."  
mash paste "$FINAL_DB" "$PUBLIC_DB" proprietary.msh

rm proprietary.msh  
echo "✅ Done! Your final, combined database is ready: $FINAL_DB"
```

You will now have a file named `Final_Combined_Database.msh` that contains all public genomes plus your private ones. This file exists only on your computer and can be used for your local analyses.

## **Verifying the Database Contents**

To ensure your sketch file is complete, compare the number of genomes it contains with the number expected from NCBI.

#### **Step 1: Get the Current Genome Count from NCBI**

This command queries NCBI for all available *S. cerevisiae* genomes (Note: you must have the `ncbi-datasets-cli` tool installed).  
```
datasets summary genome taxon 4932 | grep "total_count"
```
This will print a line like `"total_count": 1675`. This number, plus any unique outgroups and manual strains, is your target.

#### **Step 2: Count the Genomes in Your Sketch File**

Use the `mash info` command to count the sequences inside your final `.msh` file.  
```
# Replace Your_Sketch_File.msh with your actual file name  
mash info Your_Sketch_File.msh | wc -l
```

#### **Step 3: Compare the Numbers**

The number from Step 2 should match your expected total from Step 1. If they match, your database is verified!

## **How to Cite**

If you use this workflow or the database it generates in your research, please cite this GitHub repository.  
[Smardz, M. C.], & Young, E. (2025). *Saccharomyces cerevisiae Mash Database Generator: An automated GitHub Actions workflow for creating up-to-date Saccharomyces cerevisiae mash sketch databases for strain analysis* (Version 1.0). GitHub. https://github.com/mcs383/Saccharomyces-cerevisiae-Mash-Database-Generator

## **Acknowledgments**

This workflow is a modified version adapted specifically for *S. cerevisiae*. The original concept and robust parallel architecture were derived from the excellent work by Erin Young and the Fungal database generator.

* **Original Repository:** [erinyoung/update_mash_dist](https://github.com/erinyoung/update_mash_dist)  
* **Fungi Repository:** [mcs383/Fungal-RefSeq-Mash-Database-Generator](https://github.com/mcs383/Fungal-RefSeq-Mash-Database-Generator)
