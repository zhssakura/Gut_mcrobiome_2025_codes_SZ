```diff
 🔴 The metagenomic dataset is somehow missing from the manuscript.
 🔴 It is available under the NCBI BioProject Accession PRJNA1288367; 
 🔴 The script is available at: https://github.com/zhssakura/Gut_mcrobiome_2025_code_SZ.git
```
# MetaG + MetaT Analysis Workflow v1.0.0

**Publication:** Tang, Y., Kuang, J., Xia, X., Yao, C., Zhou, Z., Liu, J., Ren, Z., Ding, K., Li, M., Li, Y., Jiao, F., Zheng, D., Chen, T., Zhao, A., Wan, X., Ji, G., **Zhang, S.**, Zheng, X.*, Jia, W.* (2026). Targeting microbiota-generated acetaldehyde to prevent progression of metabolic dysfunction-associated steatotic liver disease. Cell Metabolism, 38, 1–15. (IF: 30.9) https://doi-org.eproxy.lib.hku.hk/10.1016/j.cmet.2026.01.021.

# Download Raw Data with BioSAK (v1.123.7) on HKU HPC2021
```
# Downloead fastq.gz files from ENA database.
# MetaG data
# Script: ena-file-download-read_run-PRJEB55534-fastq_ftp-20250501-1123.sh
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR101/000/ERR10114000/ERR10114000_1.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR101/000/ERR10114000/ERR10114000_2.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR101/000/ERR10114100/ERR10114100_1.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR101/000/ERR10114100/ERR10114100_2.fastq.gz
# ... add more as needed

# Sownload SRA file using BioSAK (v1.123.7).
# MetaG data
cd ./PRJNA1246224_MetaG/1_Raw_data
BioSAK sra -i ./PRJNA1246224_MetaG/0_Metadata/sra_id_AKK_Project_98_metaG.txt -o AKK_Projuect_98_20251115_metaG -t 16 -maxsize 100G
# ... add more as needed

conda activate BioSAK
# MetaT data
cd ./PRJNA1246224_MetaT/1_Raw_data
BioSAK sra -i ./PRJNA1246224_MetaT/0_Metadata/BioSAK/sra_id_AKK_Project_98_3.txt -o AKK_Projuect_98_20251109_metaT_3 -t 16 -maxsize 100G
# ... add more as needed
```

# Convert SRA files to FASTQ Format.
```
conda activate BioSAK
cd ./PRJNA1246224_MetaT/1_Raw_data
fasterq-dump ./PRJNA1246224_MetaT/1_Raw_data/AKK_Projuect_98_20251109_metaT_3/SRS24675716/SRR33081990/SRR33081990.sra --split-3 -O AKK_Projuect_98_20251109_metaT_3 -t AKK_Projuect_98_20251109_metaT_3/fasterq_dump_tmp
```

# Quality Control with FastQC (v0.11.8)
```
fastqc ERR10114000_1.fastq.gz -o ./ERR10114000_1_fastqc -t 16
fastqc ERR10114000_2.fastq.gz -o ./ERR10114000_2_fastqc -t 16
# ... add more as needed
```
# Trimming Reads with Trimmomatic (v0.39)
```
trimmomatic PE ERR10114000_1.fastq.gz ERR10114000_2.fastq.gz \
ERR10114000_1_P.fq.gz ERR10114000_1_UP.fq.gz \
ERR10114000_2_P.fq.gz ERR10114000_2_UP.fq.gz \
ILLUMINACLIP:/software/trimmomatic-0.39/adapters/TruSeq3-PE-2.fa:2:30:10:6:true \
LEADING:20 TRAILING:20 HEADCROP:5 SLIDINGWINDOW:4:20 MINLEN:50 -threads 16
```

# Assembly with SPAdes (v4.2.0)
```
spades.py --meta --only-assembler -k 21,41,61,81,101 \
-1 ERR10114000_1_P.fastq -2 ERR10114000_2_P.fastq \
-o $out_dir -t 16
```

# Binning with MetaWRAP (v1.3.2)
```
metawrap binning -t 36 --metabat2 --maxbin2 --concoct \
-a ./3_Assembly/SPAdes/ERR10114000/scaffolds.fasta \
-o ./4_Binning/ERR10114000_metaWRAP_wd \
./2_Trimmomatic/ERR10114000_1.fastq ./2_Trimmomatic/ERR10114000_2.fastq
# Noted: ERR10114000_1.fastq and _2.fastq are renamed paired sequencing reads.
```

# Bin Refinement
```
metawrap bin_refinement -o ./4_Binning/ERR10114000_metaWRAP_wd/refine_wd \
-t 36 -c 50 -x 5 \
-A ./4_Binning/ERR10114000_metaWRAP_wd/metabat2_bins \
-B ./4_Binning/ERR10114000_metaWRAP_wd/maxbin2_bins \
-C ./4_Binning/ERR10114000_metaWRAP_wd/concoct_bins

# Output location:
/4_Binning/C149_metaWRAP_wd/refine_wd/metawrap_50_5_bins/
```

# GTDB-Tk (v2.4.1) — Taxonomic Classification (requires ~200GB RAM)
```
export GTDBTK_DATA_PATH=/scr/u/shanbio/Database/GTDB/release226/release226
gtdbtk classify_wf --cpus 36 --pplacer_cpus 1 \
--genome_dir refined_bin_renamed_20250612 \
--extension fna --skip_ani_screen \
--out_dir refined_bin_renamed_GTDB_r226_20250612 \
--prefix refined_bin_renamed_GTDB_r226
```

# Coverage Estimation with CoverM (v0.7.0)
```
coverm genome --bam-files ./4_Binning/ERR10114000_metaWRAP_wd/work_files/ERR10114000.bam \
--genome-fasta-directory ./4_Binning/ERR10114000_metaWRAP_wd/refine_wd/metawrap_50_5_bins/ \
-o metawrap_50_5_bins_coverm.tsv -t 36
```

# Quality Assessment with CheckM2 (v1.1.0)
```
conda activate checkm2
checkm2 predict --force --threads 36 --input refined_bin_renamed \
-x fna --output-directory refined_bin_renamed_checkm2
```

# De-replication with dRep (v3.6.2)
```
dRep dereplicate refined_bin_renamed_dRep_ANI97 \
-pa 0.9 -sa 0.97 -comp 50 -p 36 \
-g refined_bin_renamed_20250528/*.fna \
--genomeInfo refined_bin_renamed_checkm2_quality_for_drep.tsv \
--multiround_primary_clustering --primary_chunksize 5000 \
--greedy_secondary_clustering --run_tertiary_clustering
```

# Pathway & Metabolic Analysis with gapseq (v1.4.0)
```
declare -a keywords=("PWY0-1314" "GLYCOLYSIS" "PWY-5484" "fructokinase" "2.7.1.7" "PYRUVDEHYD-PWY" "PWY-8275" "PWY-5938" "PWY-5939" "PWY-8274" "PWY-6389" "P41-PWY" "PWY-5482" "PWY-5485" "PWY-5537" "PWY-5768" "PWY3O-440" "PWY-6588" "CENTFERM-PWY" "PWY-6583" "PWY-6883" "PWY-5480" "PWY-5486" "PWY-6587" "PWY-6863" "PWY-7111" "P108-PWY" "PWY-7545" "PWY-7544" "PWY-8450" "PWY-7351" "P125-PWY" "PWY-6396" "PWY-5464" "PWY4LZ-257" "GLYCOLYSIS-TCA-GLYOX-BYPASS" "PWY-5742" "PWY-6970" "PWY-5481" "PWY-5096" "PWY-5100" "P142-PWY" "PWY-5483" "PWY-5538" "PWY-5600")

for i in "${keywords[@]}"
do
  gapseq find -p "$i" -t Bacteria -b 100 -c 70 -l MetaCyc -n -y ERR10114000.fna > ERR10114000-"$i"-report.sh
done
```

# Merging Pathway Tables for Data Analysis
## On MacOS:
```
python3 ./PycharmProjects/Gut_microbiomes/gapseq/Add_one_column_with_file_name_argv.py -dir ./8_gapseq/ -format Pathways.tbl
cd ./8_gapseq
cat *-Pathways_MAG_added.tsv > 00_merged-Pathways_MAG_added.tsv
python3 ./PycharmProjects/Gut_microbiomes/gapseq/Reformat_matrix_using_dod_dict_of_dict_and_dict_of_list_argv.py -in ./8_gapseq/find/00_merged-Pathways_MAG_added.tsv
```

## On HPC2021:
### Add additional column to files
```
python3 ./PycharmProjects/Gut_microbiomes/gapseq/Add_one_column_with_file_name_argv.py -dir ./8_gapseq/ -format Pathways.tbl
python3 ./PycharmProjects/Gut_microbiomes/gapseq/Add_one_column_with_file_name_argv.py -dir ./8_gapseq_Arc/ -format Pathways.tbl

cd ./8_gapseq
rm -rf 00_merged-Pathways_MAG_added.tsv
cat *-Pathways_MAG_added.tsv > 00_merged-Pathways_MAG_added.tsv
scp -r shanbio@hpc2021-io1:./8_gapseq/00_merged-Pathways_MAG_added.tsv ./8_gapseq/find/0609/
```

### Additional Data Processing & Visualization
```
Follow the sequential commands provided in your script for further analyses, merging, filtering, and plotting.*
Plotting (MacOS & HPC2021)
# On MacOS:
# NMDS plot
./Scripts/Rscript_NMDS_gut_microbiome.R
```


# Gene Prediction of Assembled Metagenomic Scaffold Reads with Prokka (v1.14.6)
```
conda activate blast
module load prokka/1.14.6 
cd ./PRJNA1246224_MetaG/11_Prokka
prokka --force --metagenome --cpus 36 --kingdom Bacteria --prefix Spades_assembly --locustag SRS24590218 --strain SRS24590218 --outdir SRS24590218_Bac ./PRJNA1246224_MetaG/3_Assembly/SPAdes_assemblies_renamed/scaffold_file_name_renamed_ID_renamed/fasta_renamed_bins/SRS24590218.fasta --compliant
```


# Conduct KEGG or COG Annotation on Assembled Metagenomic Scaffold Reads using BioSAK (v1.123.7).
```
conda activate BioSAK
cd ./11_Prokka_29/SRS24590646_Bac
BioSAK COG2020 -m P -t 36 -db_dir ~/my_DB/COG2020 -i Spades_assembly.faa
# The output file for next step is ./11_Prokka_29/SRS24590646_Bac/Spades_assembly_COG2020_wd/Spades_assembly_query_to_cog.txt
```


# Metatranscriptomics Analysis
## 1. Remove Low Quality Reads and Adaptors by Running KneadData (v0.12.3) to Get High-Quality Non-human Metatranscriptomic Reads.
## 2. Complete FastQC by running internally using the KneadData
### (CPU/RAM consuming, 150-200GB RAM recommend)
```
# The human transcriptome (hg38) reference database is also available (https://hgdownload.cse.ucsc.edu/downloads.html#human) for download (approx. size = 254 MB).
conda activate kneaddata
kneaddata_database --download human_transcriptome bowtie2 ./my_DB/kneaddata_DB/human_transcriptome_bowtie2/
# This will generate output files:
# human_hg38_refMrna.1.bt2
# human_hg38_refMrna.2.bt2
# human_hg38_refMrna.3.bt2
# human_hg38_refMrna.4.bt2
# human_hg38_refMrna.rev.1.bt2
# human_hg38_refMrna.rev.2.bt2


db_dir="./my_DB/kneaddata_DB/human_transcriptome_bowtie2"
cd ./PRJNA1246224_MetaT/1_Raw_data/AKK_Projuect_98_20251109_metaT_fastq
SRS_id="SRS24675982"
mkdir ./PRJNA1246224_MetaT/2_kneaddate_Bowtie2_fqc_single_node_PE2_headcrop15
out_dir="./PRJNA1246224_MetaT/2_kneaddate_Bowtie2_fqc_single_node_PE2_headcrop15/"$SRS_id
mkdir $out_dir
kneaddata -i1 $SRS_id"_1.fastq" -i2 $SRS_id"_2.fastq" --reference-db $db_dir -o $out_dir --trimmomatic-options "-Xmx64g ILLUMINACLIP:./PRJNA1246224_MetaT/TruSeq3-PE-2.fa:2:30:10 SLIDINGWINDOW:4:20 LEADING:20 TRAILING:20 HEADCROP:15 MINLEN:50" --sequencer-source TruSeq3 --run-fastqc-start --run-fastqc-end -t 16
# This will generate output files:
# adapters.fa
# fastqc
# SRS24675614_1_kneaddata_human_hg38_refMrna_0_bowtie2_paired_contam_1.fastq
# SRS24675614_1_kneaddata_human_hg38_refMrna_0_bowtie2_paired_contam_2.fastq
# SRS24675614_1_kneaddata_human_hg38_refMrna_0_bowtie2_unmatched_1_contam.fastq
# SRS24675614_1_kneaddata_human_hg38_refMrna_0_bowtie2_unmatched_2_contam.fastq
# SRS24675614_1_kneaddata_human_hg38_refMrna_1_bowtie2_paired_contam_1.fastq
# SRS24675614_1_kneaddata_human_hg38_refMrna_1_bowtie2_paired_contam_2.fastq
# SRS24675614_1_kneaddata_human_hg38_refMrna_1_bowtie2_unmatched_1_contam.fastq
# SRS24675614_1_kneaddata_human_hg38_refMrna_1_bowtie2_unmatched_2_contam.fastq
# SRS24675614_1_kneaddata.log
# SRS24675614_1_kneaddata_paired_1.fastq
# SRS24675614_1_kneaddata_paired_2.fastq
# SRS24675614_1_kneaddata.repeats.removed.1.fastq
# SRS24675614_1_kneaddata.repeats.removed.2.fastq
# SRS24675614_1_kneaddata.repeats.removed.unmatched.1.fastq
# SRS24675614_1_kneaddata.repeats.removed.unmatched.2.fastq
# SRS24675614_1_kneaddata.trimmed.1.fastq
# SRS24675614_1_kneaddata.trimmed.2.fastq
# SRS24675614_1_kneaddata.trimmed.single.1.fastq
# SRS24675614_1_kneaddata.trimmed.single.2.fastq
# SRS24675614_1_kneaddata_unmatched_1.fastq
# SRS24675614_1_kneaddata_unmatched_2.fastq


# Keep following filtered non-human metatranscriptomi reads:
SRS24675614_1_kneaddata_paired_1.fastq
SRS24675614_1_kneaddata_paired_2.fastq
SRS24675614_1_kneaddata_unmatched_1.fastq
SRS24675614_1_kneaddata_unmatched_2.fastq
```


# Run SortMeRNA (v4.3.7) to get High-Quality Non-human & Non-rRNA Metatranscriptomic Reads.
```
conda activate sortmerna
cd ./PRJNA1246224_MetaT/2_kneaddate_Bowtie2_fqc_single_node_PE2_headcrop15_PE2/SRS24675614_trim_rep/

sortmerna -ref ./my_DB/sortmeRNA_4.3.4/rRNA_databases_v4/smr_v4.3_default_db.fasta \
-reads SRS24675614_1_kneaddata_paired_1.fastq \
-reads SRS24675614_1_kneaddata_paired_2.fastq \
-workdir ./PRJNA1246224_MetaT/4_sortmerna/ \
--aligned SRS24675614_aligned \
--other SRS24675614_non_aligned \
-fastx -blast 1 -num_alignments 1 \
--sam --threads 16
# These are High-Quality Non-human & Non-rRNA Metatranscriptomic Reads to be used in mapping step, so keep them.
# SRS24675614/SRS24675614_non_aligned.fq
# SRS24675616/SRS24675616_non_aligned.fq
# SRS24675620/SRS24675620_non_aligned.fq
# SRS24675621/SRS24675621_non_aligned.fq
```

# Mapping use Salmon (v1.10.3)
```
# Split to R1/R2 (needed for tools like Salmon, or if you prefer separate files) Using seqtk (v1.5-r133)
conda activate seqtk
cd ./PRJNA1246224_MetaT/4_sortmerna
seqtk seq -1 ./20251125_old_format_non_alignment/SRS24675614_non_aligned.fq > ./20251125_old_format_non_alignment_seqtk/SRS24675614_non_rRNA_R1.fq 
seqtk seq -2 ./20251125_old_format_non_alignment/SRS24675614_non_aligned.fq > ./20251125_old_format_non_alignment_seqtk/SRS24675614_non_rRNA_R2.fq

conda activate salmon
cd ./PRJNA1246224_MetaT/4_sortmerna/20251125_old_format_non_alignment_seqtk/
salmon index -t ./PRJNA1246224_MetaG/11_Prokka_29/SRS24590646_Bac/Spades_assembly.ffn -i ./PRJNA1246224_MetaG/11_Prokka_29/SRS24590646_Bac/prokka_cds_index -p 12
salmon quant -i ./PRJNA1246224_MetaG/11_Prokka_29/SRS24590646_Bac/prokka_cds_index -l A -1 SRS24675621_non_rRNA_R1.fq -2 SRS24675621_non_rRNA_R2.fq -p 12 -o ./PRJNA1246224_MetaG/11_Prokka_29/SRS24590646_Bac/sample_salmon

# (Optional) Use below lines to find if prokka results have the annotations:
grep AdhE ./PRJNA1246224_MetaG/11_Prokka_29/SRS24590167_Bac/Spades_assembly.ffn
grep dehydrogenase ./PRJNA1246224_MetaG/11_Prokka_29/SRS24590167_Bac/Spades_assembly.ffn
# This gives rough annotation results. More details for example, KO or COG ID is needed.
```


# Get Transcripts of Specific Genes based on COG2020 Annotation Results.
## Genes I am looking for are: tdcE, grcA and AdhE
```
grep -e 'Query' -e 'COG1882' -e 'COG3445' -e 'COG1012' -e 'COG1454' ./11_Prokka_29/SRS24590646_Bac/Spades_assembly_COG2020_wd/Spades_assembly_query_to_cog.txt > ./12_Annotation/SRS24590646_Spades_assembly_query_to_cog_grep_COG1882_COG3445_COG1012_COG1454.txt
```

