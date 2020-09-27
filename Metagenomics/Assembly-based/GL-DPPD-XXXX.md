# Bioinformatics pipeline for Illumina metagenomics data (assembly-based analysis)  

> **This page holds an overview and some example code of how GeneLab processes Illumina metagenomics datasets using an assembly-based analysis approach.**

---

**Date:** September 16, 2020  
**Revision:** -  
**Document Number:** GL-DPPD-XXXX  

**Submitted by:**  
Michael D. Lee  

**In review**

---

# Table of contents  

---

# Software used

|Program|Version*|
|:------|:-----:|
|[FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)|`fastqc -v`|
|[MultiQC](https://multiqc.info/)|`multiqc -v`|
|[bbduk](https://jgi.doe.gov/data-and-tools/bbtools/bb-tools-user-guide/bbduk-guide/)|`bbduk.sh --version`|
|[megahit](https://github.com/voutcn/megahit#megahit)|`megahit -v`|
|[bit](https://github.com/AstrobioMike/bioinf_tools#bioinformatics-tools-bit)|`bit-version`|
|[bowtie2](https://github.com/BenLangmead/bowtie2#overview)|`bowtie2 --version`|
|[samtools](https://github.com/samtools/samtools#samtools)|`samtools --version`|
|[prodigal](https://github.com/hyattpd/Prodigal#prodigal)|`prodigal -v`|
|[KOFamScan](https://github.com/takaram/kofam_scan#kofamscan)|`exec_annotation -v`|
|[CAT](https://github.com/dutilh/CAT#cat-and-bat)|`CAT -v`|
|[Snakemake](https://snakemake.readthedocs.io/en/stable/)|`snakemake -v`|

>**\*** Exact versions are available along with the processing code for each specific dataset (this is due to how the system may need to be updated regularly).

---

# General processing overview with example code  

> Exact processing code for the review example dataset is in the [Snakefile here](Assembly-based/example-output/processing_info/Snakefile).  

---

## 1. Raw Data QC  

```
fastqc -o raw_fastqc_output *.fastq.gz
```

**Parameter Definitions:**

* `-o` – the output directory to store results
* `*.fastq.gz` – the input reads are specified as a positional argument, and can be given all at once with wildcards like this, or as individual arguments with spaces in between them

**Input files and/or types:**

* fastq, compressed or uncompressed

**Output files and/or types:**

* fastqc.html (FastQC output html summary)
* fastqc.zip (FastQC output data)


### Compile Raw Data QC  

```
multiqc -o raw_multiqc_output raw_fastqc_output
```

**Parameter Definitions:**

*	`-o` – the output directory to store results
*	`raw_fastqc_output/` – the directory holding the output data from the fastqc run, provided as a positional argument

**Input files and/or types:**

* fastqc.zip (FastQC output data)

**Output files and/or types:**

* multiqc_report.html (multiqc output html summary)
* multiqc_data (directory containing multiqc output data)

<br>  

---

## 2. Quality filtering/trimming

```      
bbduk.sh in=sample-1-R1.fastq.gz in2=sample-1-R2.fastq.gz out1=sample-1-R1-trimmed.fastq.gz \
         out2=sample-1-R2-trimmed.fastq.gz ref=ref-adapters.fa ktrim=l k=17 ftm=5 qtrim=rl \
         trimq=10 mlf=0.5 maxns=0 > bbduk.log 2>&1
```

**Parameter Definitions:**

*	`in` and `in2` – specifies the forward and reverse input reads, respectively

*	`out1` and `out2` – specifies the forward and reverse output reads, respectively

*	`ref` – specifies a fasta file holding potential adapter sequences (comes with bbduk installation)

*	`ktrim` – specifies to trim adapters from the 5’ end (left) if found

*	`k` – sets minimum length of kmer match to identify adapter sequences (provided by the “ref” file above)

*	`ftm` – sets a multiple of expected length the sequence should be (handles poor additional bases that are sometimes present, see “Force-Trim Modulo” section on this page)

*	`qtrim` – sets quality-score-based trimming to be applied to left and right sides

*	`trimq` – sets the score to use for PHRED-algorithm trimming

*	`mlf` – sets the minimum length of reads retained based on their initial length

*	`maxns` – sets the maximum number of Ns allowed in a read before it will be filtered out

*	`> bbduk.log 2>&1` – redirects the stderr and stdout to a log file for saving

**Input files and/or types:**

* fastq, compressed or uncompressed (original reads)

**Output files and/or types:**

* fastq, compressed or uncompressed (filtered reads)
* tsv (per sample read counts before and after filtering)
* txt (log file of standard output and error from bbduk run)

<br>

---

## 3. Filtered/Trimmed Data QC
```
fastqc -o trimmed_fastqc_output/ *trimmed.fastq.gz
```

**Parameter Definitions:**

*	`-o` – the output directory to store results  
*	`*trimmed.fastq.gz` – the input reads are specified as a positional argument, and can be given all at once with wildcards like this, or as individual arguments with spaces in between them  

**Input files and/or types:**

* fastq, compressed or uncompressed (trimmed/filtered reads)

**Output files and/or types:**

* fastqc.html (FastQC output html summary)
* fastqc.zip (FastQC output data)


### Compile Filtered Data QC
```
multiqc -o trimmed_multiqc_output  trimmed_fastqc_output
```

**Parameter Definitions:**

*	`-o` – the output directory to store results
*	`trimmed_fastqc_output` – the directory holding the output data from the fastqc run, provided as a positional argument

**Input files and/or types:**

* fastqc.zip (FastQC output data)

**Output files and/or types:**

* multiqc_report.html (multiqc output html summary)
* multiqc_data (directory containing multiqc output data)

<br>

---

## 4. Sample assembly
```
megahit -1 sample-1-R1-trimmed.fastq.gz -2 sample-1-R2-trimmed.fastq.gz \
        -o sample-1-assembly -t 10 -m 500 > sample-1-assembly.log 2>&1
```

**Parameter Definitions:**  

*	`-1 and -2` – specifies the input forward and reverse reads

*	`-o` – specifies output directory

*	`-t` – specifies the number of threads to use

*	`-m` – specifies the minimum contig length to write out

*	`> sample-1-assembly.log 2>&1` – sends stdout/stderr to log file


**Input files and/or types:**

* fastq, compressed or uncompressed (filtered reads)

**Output files and/or types:**

* fasta, assembly file
* txt, log file

<br>

---

## 5. Renaming contigs and summarizing assemblies

**Renaming contig headers:**
```
bit-rename-fasta-headers -i sample-1-assembly-orig.fasta -w c_sample-1 -o sample-1-assembly.fasta
```

**Parameter Definitions:**  

*	`-i` – input fasta file

*	`-w` – wanted header prefix (a number will be appended for each contig), starts with a “c_” to ensure they won’t start with a number which can be problematic

*	`-o` – output fasta file

*	`--remove-temp-output` – delete the temp files after finishing

**Summarizing assemblies:**

```
bit-summarize-assembly -o assembly-summaries.tsv *assembly.fasta
```

**Parameter Definitions:**  

*	`-o` – output summary table

*	– multiple input assemblies can be provided as positional arguments


**Input files and/or types:**

* fasta, assembly files

**Output files:**

* fasta, contig-renamed assembly file
* tsv, table of assembly summary statistics

<br>

---

## 6. Gene prediction
```
prodigal -a sample-1-genes.faa -d sample-1-genes.fasta -f gff -p meta -c -q \
         -o sample-1-gene-calls.gff -i sample-1-assembly.fasta
```
**Parameter Definitions:**

*	`-a` – specifies the output amino acid sequences file

*	`-d` – specifies the output nucleotide sequences file

*	`-f` – specifies the output format gene-calls file

*	`-p` – specifies which mode run the gene-caller in 

*	`-c` – no incomplete genes reported 

*	`-q` – run in quiet mode (don’t output process on each contig) 

*	`-o` – specifies the name of the output gene-calls file 

*	`-i` – specifies the input assembly

**Input files and/or types:**

* fasta, assembly file

**Output files and/or types:**

* fasta, sample-1-genes.faa, amino-acid fasta file
* fasta, sample-1-genes.fasta, nucleotide fasta file
* gff, sample-1-genes.gff, gene-calls in general feature format

<br>

---

## 7. Functional annotation
> **Notes**  
> The annotation process overwrites the same temporary directory by default. So if running multiple at a time, it is necessary to specify a specific temporary directory with the `--tmp-dir` argument as shown below.


Downloading reference database of HMM models (only needs to be done once):

```
curl -LO ftp://ftp.genome.jp/pub/db/kofam/profiles.tar.gz
curl -LO ftp://ftp.genome.jp/pub/db/kofam/ko_list.gz
tar -xzvf profiles.tar.gz
gunzip ko_list.gz 
```

Running KEGG annotation:
```
exec_annotation -p profiles/ -k ko_list --cpu 15 -f detail-tsv -o sample-1-KO-tab.tmp \
                --tmp-dir sample-1-tmp-KO --report-unannotated sample-1-genes.faa 
```

**Parameter Definitions:**
*	`-p` – specifies the directory holding the downloaded reference HMMs

*	`-k` – specifies the downloaded reference KO terms 

*	`--cpu` – specifies the number of searches to run in parallel

*	`-f` – specifies the output format

*	`-o` – specifies the output file name

*	`--tmp-dir` – specifies the temporary directory to write to (needed if running more than one concurrently, see Notes above)

*	`--report-unannotated` – specifies to generate an output for each entry

*	the input file is specified as a positional argument


Filtering output to retain only those passing the KO-specific score and top hits:
```
bit-filter-KOFamScan-results -i sample-1-KO-tab.tmp -o sample-1-annotations.tsv

  # removing temporary files
rm -rf sample-1-tmp-KO/ sample-1-KO-annots.tmp
```

**Parameter Definitions:**  

*	`-i` – specifies the input table

*	`-o` – specifies the output table


**Input files and/or types:**

* fasta, gene-calls amino acid fasta file

**Output files and/or types:**

* tsv, table of KO annotations assigned to gene IDs

<br>

---

## 8. Taxonomic classification

Pulling and un-packing pre-built reference db (only needs to be done once):
```
wget tbb.bio.uu.nl/bastiaan/CAT_prepare/CAT_prepare_20200618.tar.gz
tar -xvzf CAT_prepare_20200618.tar.gz
```

Running taxonomic classification:
```
CAT contigs -c sample-1-assembly.fasta -d CAT_prepare_20200618/2020-06-18_database/ \
            -t CAT_prepare_20200618/2020-06-18_taxonomy/ -p sample-1-genes.faa \
            -o sample-1-tax-out.tmp -n 15 -r 3 --top 3 --I_know_what_Im_doing
```

**Parameter Definitions:**  

*	`-c` – specifies the input assembly fasta file

*	`-d` – specifies the CAT reference sequence database

*	`-t` – specifies the CAT reference taxonomy database

*	`-p` – specifies the input protein fasta file

*	`-o` – specifies the output prefix

*	`-n` – specifies the number of cores to use

*	`-r` – specifies the number of top protein hits to consider in assigning tax

*	`--top` – specifies the number of protein alignments to store

*	`--I_know_what_Im_doing` – allows us to alter the `--top` parameter


Adding taxonomy info from taxids to genes:
```
CAT add_names -i sample-1-tax-out.tmp.ORF2LCA.txt -o sample-1-gene-tax-out.tmp \
              -t CAT_prepare_20200618/2020-06-18_taxonomy/ --only_official
```

Adding taxonomy info from taxids to contigs:
```
CAT add_names -i sample-1-tax-out.tmp.contig2classification.txt -o sample-1-contig-tax-out.tmp \
              -t CAT-ref/2020-06-18_taxonomy/ --only_official
```

**Parameter Definitions:**  

*	`-i` – specifies the input taxonomy file

*	`-o` – specifies the output file 

*	`-t` – specifies the CAT reference taxonomy database

*	`--only_official` – specifies to add only standard taxonomic ranks



Formatting gene-level output with awk and sed:
```
awk -F $'\t' ' BEGIN { OFS=FS } { if ( $2 == "lineage" ) { print $1,$2,$4,$5,$6,$7,$8,$9,$10 } \
    else if ( $2 == "ORF has no hit to database" || $2 ~ /^no taxid found/ ) \
    { print $1,"NA","NA","NA","NA","NA","NA","NA","NA" } else { n=split($2,lineage,";"); \
    print $1,lineage[n],$4,$5,$6,$7,$8,$9,$10 } } ' sample-1-gene-tax-out.tmp | \
    sed 's/not classified/NA/g' | sed 's/superkingdom/domain/' | sed 's/^# ORF/gene_ID/' | \
    sed 's/lineage/taxid/' | sed 's/\*//g' > sample-1-gene-tax-out.tsv
```

Formatting contig-level output with awk and sed:
```
awk -F $'\t' ' BEGIN { OFS=FS } { if ( $2 == "classification" ) { print $1,$4,$6,$7,$8,$9,$10,$11,$12 } \
    else if ( $2 == "unclassified" ) { print $1,"NA","NA","NA","NA","NA","NA","NA","NA" } \
    else { n=split($4,lineage,";"); print $1,lineage[n],$6,$7,$8,$9,$10,$11,$12 } } ' sample-1-contig-tax-out.tmp | \
    sed 's/not classified/NA/g' | sed 's/superkingdom/domain/' | sed 's/: [0-9\.]*//g' | sed 's/^# contig/contig_ID/' | \
    sed 's/lineage/taxid/' | sed 's/\*//g' > sample-1-contig-tax-out.tsv

  # clearing intermediate files
rm sample-1*.tmp*
```

**Input files and/or types:**

* fasta, assembly fasta file
* fasta, gene-calls amino acid fasta file

**Output files and/or types:**

* tsv, gene-level taxonomic classifications
* tsv, contig-level taxonomic classifications

<br>

---

## 9. Read-mapping

Building reference index:
```
bowtie2-build sample-1-assembly.fasta sample-1-assembly-bt-index
```

**Parameter Definitions:**  

*	first positional argument specifies the input assembly

*	second positional argument specifies the prefix of the output index files


Performing mapping, conversion to bam, and sorting:
```
bowtie2 --threads 15 -x sample-1-assembly-bt-index -1 sample-1-R1-trimmed.fastq.gz \
        -2 sample-1-R2-trimmed.fastq.gz 2> sample-1-mapping.log | samtools view -b | samtools sort -@ 15 > sample-1.bam
```

**Parameter Definitions:**  

*	`--threads` – specifies the number of threads to run in parallel

*	`-x` – specifies the prefix of the reference index files to map to

*	`-1 and -2` – specifies the forward and reverse reads to map

* `2> sample-1-mapping.log` – capture the printed summary results in a log file

*	`samtools view -b` – convert the output directly to bam format (compressed)

*	`samtools sort -@` – sort the bam file using the specified number of threads threads

*	`>` – redirect the output to a file

Indexing:
```
samtools index -@ 15 sample-1.bam 
```

**Parameter Definitions:**  
*	`-@` – set number of threads to use 

*	input bam file is provided as a positional argument

**Input files and/or types:**

* fasta, assembly file
* fastq, trimmed read files

**Output files and/or types:**

* bam, mapping file
* bai, bam index file
* log, read-mapping log file

<br>

---

## 10. Getting coverage and detection information from bam files, and filtering coverages based on detection
> **Notes**  
> “Detection” is a metric of what proportion of a reference sequence recruited reads (see [here](http://merenlab.org/2017/05/08/anvio-views/#detection)). 
Filtering coverage levels based on detection is one way of helping to mitigate non-specific read-recruitment.  

```
pileup.sh -in sample-1.bam fastaorf=sample-1-genes.fasta outorf=sample-1-gene-cov-and-det.tmp \
          out=sample-1-contig-cov-and-det.tmp
```

**Parameter Definitions:**  

*	`-in` – the input bam file

*	`fastaorf=` – input genes nucleotide fasta

*	`outorf=` – the output tsv file


Filtering gene coverage based on requiring 50% detection and parsing down to just gene ID and coverage:
```
grep -v "#" sample-1-gene-cov-and-det.tmp | awk -F $'\t' ' BEGIN { OFS=FS } { if ( $10 <= 0.5 ) $4 = 0 } \
     { print $1,$4 } ' > sample-1-gene-cov.tmp

cat <( printf "gene_ID\tcoverage\n" ) sample-1-gene-cov.tmp > sample-1-gene-coverages.tsv
```

Filtering contig coverage based on requiring 50% detection and parsing down to just contig ID and coverage:
```
grep -v "#" sample-1-contig-cov-and-det.tmp | awk -F $'\t' ' BEGIN { OFS=FS } { if ( $5 <= 50 ) $2 = 0 } \
     { print $1,$2 } ' > sample-1-contig-cov.tmp

cat <( printf "contig_ID\tcoverage\n" ) sample-1-contig-cov.tmp > sample-1-contig-coverages.tsv

  # removing intermediate files

rm sample-1-*.tmp
```

**Input files and/or types:**

* bam, bam file
* fasta, gene-calls nucleotide file

**Output files and/or types:**

* tsv, table with gene-level coverages
* tsv, table with contig-level coverages

<br>

---

## 11. Combining gene coverages, taxonomy, and functional annotations into one table for each sample
> **Notes**  
> Just uses `paste`, `sed`, and `awk`, all are standard in any Unix-like environment.  

```
paste <( tail -n +2 sample-1-gene-coverages.tsv | sort -V -k 1 ) <( tail -n +2 sample-1-annotations.tsv | sort -V -k 1 | cut -f 2- ) \
      <( tail -n +2 sample-1-tax-out.tsv | sort -V -k 1 | cut -f 2- ) > sample-1-gene-tab.tmp

paste <( head -n 1 sample-1-gene-coverages.tsv ) <( head -n 1 sample-1-annotations.tsv | cut -f 2- ) \
      <( head -n 1 sample-1-tax-out.tsv | cut -f 2- ) > sample-1-header.tmp

cat sample-1-header.tmp sample-1-gene-tab.tmp > sample-1-gene-coverage-annotation-and-tax.tsv

  # removing intermediate files
rm sample-1*tmp sample-1-gene-coverages.tsv sample-1-annotations.tsv sample-1-gene-tax-out.tsv
```

**Input files and/or types:**

* tsv, gene coverage, annotation, and taxonomy files


**Output files and/or types:**

* tsv, table with combined gene coverage, annotation, and taxonomy info

<br>

---

## 12. Combining contig coverages and taxonomy info into one table for each sample
> **Notes**  
> Just uses `paste`, `sed`, and `awk`, all are standard in any Unix-like environment.  

```
paste <( tail -n +2 sample-1-contig-coverages.tsv | sort -V -k 1 ) \
      <( tail -n +2 sample-1-contig-tax-out.tsv | sort -V -k 1 | cut -f 2- ) > sample-1-contig.tmp

paste <( head -n 1 sample-1-contig-coverages.tsv ) <( head -n 1 sample-1-contig-tax-out.tsv | cut -f 2- ) \
      > 5492-contig-header.tmp
      
cat sample-1-contig-header.tmp sample-1-contig.tmp > sample-1-contig-coverage-annotation-and-tax.tsv

  # removing intermediate files
rm sample-1*tmp sample-1-contig-coverages.tsv sample-1-contig-tax-out.tsv
```

**Input files and/or types:**

* tsv, contig coverage and taxonomy files

**Output files and/or types:**

* tsv, table with combined contig coverage and taxonomy info

<br>

---

## 13. Generating normalized, gene-level coverage summary tables of KO-annotations and taxonomy across samples
> **Notes**  
> * To combine across samples to generate these summary tables, we need the same "units". This is done for annotations based on the assigned KO terms, and all non-annotated functions are included together as "Not annotated". It is done for taxonomic classifications based on taxids (full lineages included in the table), and any not classified are included together as "Not classified". 
> * The values we are working with are coverage per gene (so they are number of bps recruited to the gene normalized by the length of the gene). These have been normalized by making the total coverage of a sample 1,000,000 and setting each individual gene-level coverage its proportion of that 1,000,000 total. So basically percent, but out of 1,000,000 instead of 100 to make the numbers more friendly. 

```
bit-GL-combine-KO-and-tax-tables *-gene-coverage-annotation-and-tax.tsv -o GLDS-286
```

**Parameter Definitions:**  

*	takes positional arguments specifying the input tsv files, can be provided as a space-delimited list of files, or with wildcards like above

-	`-o` – specifies the output prefix (e.g. as above, will generate “GLDS-286-KO-function-coverages.tsv” and “GLDS-286-taxonomy-coverages.tsv”


**Input files and/or types:**

* tsv, tables with gene-level coverage, functional annotations, and taxonomic classifications to combine

**Output files and/or types:**

* tsv, table with all samples combined based on KO annotations (normalized to coverage per million genes covered)
* tsv, table with all samples combined based on gene-level taxonomic classifications (normalized to coverage per million genes covered)

<br>

---
