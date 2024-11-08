# Exercises

1. [Setting up the cloud computing](#setting-up-the-cloud-computing)
   1. [Setting up VS Code](#setting-up-vs-code)
   2. [Connecting to the remote machine with VS Code](#connecting-to-the-remote-machine-with-vs-code)
   3. [Cloning the course's GitHub repository](#cloning-the-courses-github-repository)
2. [Getting the raw data](#getting-the-raw-data)
3. [QC and trimming](#qc-and-trimming)
   1. [QC of the raw data](#qc-of-the-raw-data)
   2. [Read trimming](#read-trimming)
   3. [QC of the trimmed data](#qc-of-the-trimmed-data)
4. [Read-based taxonomic profiling](#read-based-taxonomic-profiling)
   1. [singleM](#singlem)
   2. [sourmash](#sourmash)
5. [Metagenome assembly](#metagenome-assembly)
   1. [Assembly QC](#assembly-qc)
6. [Genome-resolved metagenomics with anvi'o](#genome-resolved-metagenomics-with-anvio)
7. [Quality control and taxonomic annotation of metagenome-assembled genomes (MAGs)](#quality-control-and-taxonomic-annotation-of-metagenome-assembled-genomes-mags)
8. [Automatic binning with SemiBin2](#automatic-binning-with-semibin2)
9. [Targeted functional analysis of MAGs](#targeted-functional-analysis-of-mags)

## Setting up the cloud computing

We will use the [Amazon Cloud](https://aws.amazon.com/ec2/) (AWS EC2) services for most of the analyses.  
The IP address of the remote machine will change every day, so a new IP adress will be posted in Slack each morning.  
Your username - that you have received by e-mail/Slack - will be the same for the whole course.  
We will use `ssh` to connect to the remote machine.  
We encourage the use of [VS Code](https://code.visualstudio.com/Download), but you are welcome to use any IDE or terminal emulator that you are comfortable with.  

### Setting up VS Code

1. Download `VS Code` and set it up as shown [here](Lectures/course-outline-and-practical-info.pdf)  
2. Save the `.pem` file that you have received by e-mail somewhere in your computer  
3. **Linux/MacOS users only:**  
   1. Launch the `Terminal` app  
   2. `cd` to the directory where you saved the `.pem` file
   3. run `chmod 600 userXX.pem` (remember to change `userXX.pem` by the name of your own file)
4. Back to `VS Code`, go to `View -> Command Palette`  
5. Search for `ssh config`  
6. Select `Remote-SHH: Open SSH Configuration File...`  
7. In the next dialogue box, hit `Enter/Return`
8. Copy and paste the following text:

```
Host physalia
  HostName 54.245.21.143
  User user1
  IdentityFile ~/Desktop/user1.pem
```

9. In the 2nd, 3rd, and 4th lines:  
   1. HostName: change to the IP adress of the day
   2. User: change to your own username
   3. IdentityFile: change to the location and name of the `.pem` file that you have saved in your computer 
10. Save and close the `config` file

### Connecting to the remote machine with VS Code

11. Go to `View -> Command Palette`
12. Search for `ssh connect`
13. Select `Remote-SHH: Connect to Host...`
14. Select `physalia` (a new window will open)
15. If a dialogue box opens asking the server type, select `Linux`
16. If a dialogue box opens asking if you are sure, select `Continue`
17. If a terminal does not open by default, go to `Terminal -> New Terminal`

That's it, you should now be connected to the remote machine and ready to go!  
**Remember:** every day you should redo steps 4-7 and update `HostName` to match the IP adress of the day.  

### Cloning the course's GitHub repository

Once you have connected to the remote machine, you will be in your home folder (`/users/userXX`, also represented by `~` or `$HOME`).  
**Remember:** You can check where you are with the command `pwd`.  

To have access to the course's content, let's copy the GitHub repository to your `home` folder using `git clone`:

**Do this on the first day:** 

```bash
cd ~
git clone https://github.com/NikolayOskolkov/Physalia_EnvMetagenomics_2024
```

**Do this every once in a while, at least each day before starting the activities:**  

```bash
cd ~/Physalia_EnvMetagenomics_2024
git pull
```

**Note:** All exercises will be executed inside the `Physalia_EnvMetagenomics_2024` folder that you cloned inside your own `home` folder.  
So remember to `cd ~/Physalia_EnvMetagenomics_2024` every time you connect to the remote machine.  

## Getting the raw data

Copy the raw sequencing data to your own `01_DATA` folder.  
Also copy the file `SAMPLES.txt`, which will be useful for running `for loop` and etc.  

```bash

cd ~/Physalia_EnvMetagenomics_2024
mkdir 01_DATA

cp ~/Share/toy_data/*.fastq.gz 01_DATA/
cp ~/Share/toy_data/SAMPLES.txt ./
```

## QC and trimming

Now that you have copied the raw data to your working directory, let's do some quality control.  
The sequencing process is subject to several types of problems that can introduce errors and artifacts in the sequences.  
Because of this, bioinformatics analyses usually start with the quality control of raw sequences.  
He we will use [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc) and [MultiQC](https://multiqc.info/) to obtain quality reports, and [Cutadapt](https://cutadapt.readthedocs.io/en/stable/) for trimming the Illumina data, respectively.  

### QC of the raw data

Go to your `Physalia_EnvMetagenomics_2024` folder, create a folder for the QC files, and activate the `conda` environment:  

```bash
cd ~/Physalia_EnvMetagenomics_2024
mkdir 02_QC_RAW
conda activate envmetagenomics
```

And now you're ready to run the QC on the raw data:

```bash
fastqc 01_DATA/*.fastq.gz -o 02_QC_RAW -t 4
multiqc 02_QC_RAW -o 02_QC_RAW --interactive
```

After the QC is finished, copy the `MultiQC` report (`02_QC_RAW/multiqc_report.html`) to your local machine and open it with your favourite browser.  
We will go through the report together before continuing with the pre-processing.  

**NOTE:** to move files to and from local and remote machines, you can use: 
- The command-line tool [scp](https://kb.iu.edu/d/agye)  
- A file transfer software such as [FileZilla](https://filezilla-project.org)  
- The `VS Code` built-in `Explorer` tool (`View -> Explorer`)  

### Read trimming

Our QC reports tell us that a significant percentage of the raw sequences contain some isses such as the presence of adapters.  
Before proceeding, it is necessary to clean up/trim the raw sequences.  
Before start trimming the data, let's create a folder for the processed data and activate the `conda` environment:  

```bash
cd ~/Physalia_EnvMetagenomics_2024
mkdir 03_TRIMMED
conda activate envmetagenomics
```

For the Illumina data, we will use a `for loop` to process each of the samples one after the other:  

```bash
for sample in $(cat SAMPLES.txt); do
  cutadapt 01_DATA/${sample}_R1.fastq.gz \
           01_DATA/${sample}_R2.fastq.gz \
           -o 03_TRIMMED/${sample}_R1.fastq.gz \
           -p 03_TRIMMED/${sample}_R2.fastq.gz \
           -a CTGTCTCTTATACACATCTCCGAGCCCACGAGAC \
           -A CTGTCTCTTATACACATCTGACGCTGCCGACGA \
           -m 50 \
           -q 20 \
           -j 4 > 03_TRIMMED/${sample}.log
done
```

While `Cutadapt` is running: looking at the [online manual](https://cutadapt.readthedocs.io/en/stable/index.html) or running `cutadapt --help`, answer:  

- What do the `-o`, `-p`, `-a`, `-A`, `m`, `-q`, and `-j` flags mean?  
- How did we choose the values for `-m` and `-q`?  
- What is the purpose of the redirection (`> 03_TRIMMED/${sample}.log`)?  


### QC of the trimmed data

Now the data has been trimmed, it would be a good idea to run `FastQC` and `MultiQC` again.  
Modify the [commands used for the raw data](#qc-of-the-raw-data) to match the trimmed data and run the two QC softwares.  

While you wait, take a look at the `Cutadapt` logs.  
When `Cutadapt` runs, it prints lots of interesting information to the screen, which we lose once we logout of the remote machine.  
Because we used redirection (`>`) to capture the standard output (`stdout`) of `Cutadapt`, this information is now stored in a file (`03_TRIMMED/${sample}.illumina.log`).  
Take a look at the log file for one of the samples using the program `less`:  

**NOTE:** You can scroll up and down using the arrow keys on your keyboard, or move one "page" at a time using the spacebar.  
**NOTE:** To quit `less`, hit the `q` key.  
**NOTE:** If you have set it up, you can also access the files using the `Explorer` tab on `VS Code` (`View -> Explorer`).  

By looking at the `Cutadapt` log, can you answer:  
- How many read pairs we had originally?  
- How many reads contained adapters?  
- How many read pairs were removed because they were too short?  
- How many base calls were quality-trimmed?  
- Overall, what is the percentage of base pairs that were kept?  

When `FastQC` and `MultiQC` have finished, copy the `MultiQC` report to your local machine and open it with a browser.  
Compare this with the report obtained earlier for the raw data.  
Do the data look better now?  

## Read-based taxonomic profiling

There are many different tools and approaches for obtaining taxonomic profiles from metagenomes.  
Here we will use a popular read-based taxonomic profiler [Kraken2](https://github.com/DerrickWood/kraken2) and [sourmash](https://sourmash.readthedocs.io/en/latest/).  
What is the basic approach that each of these tools use and how they can impact the results?  
Well, let's find out!  

First let's create a folder to store the results:  

```bash
cd ~/Physalia_EnvMetagenomics_2024
mkdir 05_TAXONOMIC_PROFILE
```

### Kraken2

And now let's run `Kraken2`:  

```bash
conda activate envmetagenomics

kraken2 --db ~/Share/Databases/minikraken2_v2_8GB_201904_UPDATE \
	--paired 03_TRIMMED/sample1_ILM_R1.fastq.gz 03_TRIMMED/sample1_ILM_R2.fastq.gz \
	--output 05_TAXONOMIC_PROFILE/sequences.kraken --report 05_TAXONOMIC_PROFILE/kraken.output \
	--report-minimizer-data --use-names --threads 10
```

Now that we have got our hands into some tables describing the abundance of the different taxa in our metagenome, it is time to make sense of the data.  
One way to do this is making summaries, plots, statistical tests, etc, as you would normally do for any kind of species distribution data.  
Here you are free to use whichever tool you are most familiar with (but we all know that there is only one co`R`rect tool for this).  

The idea here is to: 
- Learn what are the main (most abundant) taxa in our samples  
- Learn about potential differences in community composition between the samples  
- Learn what fraction of the community we were actually able to identify at, let's say, the genus level  
- Compare the taxonomic profiles obtainted from Illumina and Nanopore data  

Hopefully you will be able to learn a bit about these metagenomic datasets.  
And realise that there is so much that still remains unknown...  

If you don't have R installed or can't install packages yourself, we have prepared a virtual Rstudio for you with example data.  
Just click this: [![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/karkman/binder_rstudio/main?urlpath=rstudio)

### sourmash

There are many different appraoches for taxonomic profiling of metagenomes, each of them with their own up- and downsides.  
Let's now try a `sourmash`:  

```bash
conda activate envmetagenomics

for sample in $(cat SAMPLES.txt); do
  sourmash sketch dna 03_TRIMMED/${sample}.illumina.R?.fastq.gz \
                      -p k=31,scaled=1000,abund \
                      -o 05_TAXONOMIC_PROFILE/${sample}.sig.zip \
                      --merge ${sample}

  sourmash gather 05_TAXONOMIC_PROFILE/${sample}.sig.zip \
                  ~/Share/Databases/gtdb-rs207.genomic-reps.dna.k31.zip \
                  -k 31 \
                  -o 05_TAXONOMIC_PROFILE/${sample}.gather.csv
done

# Gather results
sourmash tax metagenome -g 05_TAXONOMIC_PROFILE/*.gather.csv \
                        -t ~/Share/Databases/gtdb-rs207.taxonomy.with-strain.csv.gz \
                        --output-dir 05_TAXONOMIC_PROFILE \
                        --output-base sourmash.phylum \
                        --output-format lineage_summary \
                        --rank phylum

sourmash tax metagenome -g 05_TAXONOMIC_PROFILE/*.gather.csv \
                        -t ~/Share/Databases/gtdb-rs207.taxonomy.with-strain.csv.gz \
                        --output-dir 05_TAXONOMIC_PROFILE \
                        --output-base sourmash.genus \
                        --output-format lineage_summary \
                        --rank genus
```

Now analyse the results from `sourmash` in `R` or other data analysis tool of your preference.  
Are there differences between the taxonomic profiles obtained by the two different tools?  

## Metagenome assembly

Now it's time to move forward to metagenome assembly.  
First let's create a folder where the assembly will go:  

```bash
cd ~/Physalia_EnvMetagenomics_2024
mkdir 06_ASSEMBLY
```

For the assembly of our Nanopore data we will use [Flye](https://github.com/fenderglass/Flye).  
`Flye` is a long-read de novo assembler that can also handle metagenomic data.  

Before you start the assembly, have a look at the [Flye manual](https://github.com/fenderglass/Flye/blob/flye/docs/USAGE.md), especially the parts about Nanopore data and metagenome assembly.  

__What options do we need?__  
We have only given the output directory in the script below; modify it as necessary and run `Flye`:  

```bash 
conda activate flye

flye \
   --nano-raw 03_TRIMMED/nanopore.fastq.gz \
   --meta \
   -t 4 \  
   --out-dir 06_ASSEMBLY
```

### Assembly QC

For the sake of speed, the metagenomic assembly you have done above was made with heavily downsampled data.  
We have also prepared an assembly made from the original (not downsampled) data, which will be the assembly you will actually use downstream.  
So let's copy this bigger assembly from the `Share` folder to the `Flye` output folder.  
We will run QC for both assemblies and compare the outputs.  

```bash
cp ~/Share/Data/${STUDY}/full_assembly.fasta 06_ASSEMBLY
```

For assembly QC we will use the metagenomic version of Quality Assessment Tool for Genome Assemblies, [Quast](http://quast.sourceforge.net/) for evaluating (and comparing) our assemblies.

```bash
mkdir 07_ASSEMBLY_QC

conda activate quast

metaquast.py 06_ASSEMBLY/*.fasta \
      --output-dir 07_ASSEMBLY_QC \
      --max-ref-number 0 \
      --threads 4
```

## Genome-resolved metagenomics with anvi'o

`Anvi'o` is an analysis and visualization platform for omics data.  
We will use `anvi'o` for binning contigs into metagenome-assembled genomes (MAGs).  
You should definitely take a look at their [website](https://anvio.org/) and maybe even join their [discord channel](https://discord.gg/C6He6mSNY4).  
Let's start by making a new folder for `anvi'o`:  

```bash
cd ~/Physalia_EnvMetagenomics_2023
mkdir 08_ANVIO
```

### Reformat the assembly file

Before creating the contigs database in `anvi'o`, we need to do some reformatting for our assmebly file.  
The program removes contigs shorter than 1 000 bp and simplifyes the sequence names.

```bash
conda activate anvio

anvi-script-reformat-fasta \
    06_ASSEMBLY/full_assembly.fasta \
    -o 08_ANVIO/contigs.fasta \
    --report-file 08_ANVIO/reformat_report.txt \
    --min-len 1000 \
    --simplify-names
```

### Contigs database and annotations

The first step in our genome-resolved metagenomics pipeline is the contruction of contigs database from our metagenomic contigs.  
During this step `anvi'o` calls genes, calculates the nucleotide composition for each contigs and annotates the identified genes in three different steps.

**Generate contigs database**

```bash
anvi-gen-contigs-database \
    -f 08_ANVIO/contigs.fasta \
    -o 08_ANVIO/CONTIGS.db \
    -n FullAssembly \
    -T 4
```

**Annotate marker genes**  

These are used to estimate the completeness and redundancy of bins in `anvi'o`. 

```bash
anvi-run-hmms \
    -c 08_ANVIO/CONTIGS.db \
    -T 4
```

**Annotate COGs**  

This adds COG annotations to gene calls. 

```bash
anvi-run-ncbi-cogs \
    -c 08_ANVIO/CONTIGS.db \
    -T 4
```

**Annotate single-copy core genes**  

These are used for taxonomic annotations of bins. 

```bash
anvi-run-scg-taxonomy \
    -c 08_ANVIO/CONTIGS.db \
    -T 4
```

### Mapping Illumina reads back to assembly

The differential coverage for each contig is calculated by mapping sequencing reads to the assembly.  
We will use Bowtie2 to map the short-read Illumina data to our assembly (the anvio-reformatted version of it).

```bash
bowtie2-build --threads 4 08_ANVIO/contigs.fasta 08_ANVIO/contigs

for sample in $(cat SAMPLES.txt); do
   bowtie2 \
        -1 03_TRIMMED/${sample}.illumina.R1.fastq.gz \
        -2 03_TRIMMED/${sample}.illumina.R2.fastq.gz \
        -x 08_ANVIO/contigs \
        -S 08_ANVIO/${sample}.sam \
        --threads 4 \
        --no-unal

   samtools view -@ 4 -F 4 -bS 08_ANVIO/${sample}.sam |\
      samtools sort -@ 4 > 08_ANVIO/${sample}.bam
    
   samtools index -@ 4 08_ANVIO/${sample}.bam
    
   rm -f 08_ANVIO/${sample}.sam
done 
```

### Profiling

When the contigs database and all three mappings are ready, we can make the profile databases for each sample.

```bash
for sample in $(cat SAMPLES.txt); do
   anvi-profile \
        -i 08_ANVIO/${sample}.bam \
        -c 08_ANVIO/CONTIGS.db \
        -S ${sample} \
        --min-contig-length 5000 \
        -o 08_ANVIO/${sample}_PROFILE \
        -T 4
done
```

And finally merge the individual profiles

```bash
anvi-merge \
   -o 08_ANVIO/SAMPLES-MERGED \
   -c 08_ANVIO/CONTIGS.db \
   --enforce-hierarchical-clustering \
   08_ANVIO/*_PROFILE/PROFILE.db 
```

### Visualization

Now we have all the files ready for anvi'o and we can visualize the results in anvi'o interactive view.
We will go thru the first steps together.  

Your port number will be 8080 + your user number (user1 == 1 == 8081).

```bash
export ANVIOPORT=
```

Then you can launch the interactive interface with the following command.

```bash
anvi-interactive \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --server-only \
    --port-number $ANVIOPORT
```

### Metagenomic binninng

After making the pre-clustering, start refining the clusters.

```bash
anvi-refine \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --server-only \
    --port-number $ANVIOPORT \
    --collection-name PreCluster \
    --bin-id Bin_1
```

When all cluster have been refined a bit more, we can make a new collection from these.

```bash
anvi-rename-bins \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --collection-to-read PreCluster \
    --collection-to-write PreBins \
    --prefix Preliminary \
    --report-file 08_ANVIO/PreBin_report.txt
```

And then we can make a summary of the cluster or bins we have so far.

```bash
anvi-summarize \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --collection-name PreBins \
    --output-dir 08_ANVIO/PreBins_SUMMARY \
    --quick-summary
```

The last step in anvi'o is to make a final collection and summarize that final collection.  
To make the rename part universal, store the name of your most recent collection in env variable `$COLLECTION`.

```bash
export COLLECTION=YOUR_MOST_RECENT_COLLECTION
```

Then run the renaming.  

```bash
anvi-rename-bins \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --collection-to-read $COLLECTION \
    --collection-to-write Final \
    --prefix $USER \
    --report-file 08_ANVIO/Final_report.txt
```

And then we can make a summary of the final bins.

```bash
anvi-summarize \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --collection-name Final \
    --output-dir 08_ANVIO/SUMMARY_Final
```

## Quality control and taxonomic annotation of metagenome-assembled genomes (MAGs)

Now we have obtained some bins that we think could represent genomes present in our samples. Next steps are QC and taxonomic annotation of our genomes.  

We will use a program called [checkM2](https://github.com/chklovski/CheckM2) to get more precise estimates of the completeness and redundancy of these genomes.  
For taxonomic annotation we will use Genome Taxonomy Database ([GTDB](https://gtdb.ecogenomic.org/)) and a tool called [GTDB-Tk](https://ecogenomics.github.io/GTDBTk/installing/index.html#installing-gtdbtk-reference-data).

But before we can do these steps, we should copy the most interesting genomes to a separate folder (to save time we only do this for few genomes)  .  

First make text file called: `Final_genomes.txt` in the `08_ANVIO` folder that has the names of the bins that you want to work with.  

Our file would look like this:

```bash
ubuntu_Bin_00001
ubuntu_Bin_00002
ubuntu_Bin_00003
ubuntu_Bin_00004
ubuntu_Bin_00005
```

Then we'll make a new folder for these genomes and copy the fasta files for each of these genomes there from the anvi'o summary folder (`SUMMARY_Final`).  

```bash 
mkdir 09_GENOMES

for bin in $(cat 08_ANVIO/Final_genomes.txt); do
    cp 08_ANVIO/SUMMARY_Final/bin_by_bin/${bin}/${bin}-contigs.fa 09_GENOMES/
done
```

Now you should have one fasta file per genome you selected in folder `09_GENOMES`. 

### checkM2

```bash
conda activate checkm2

checkm2 predict \
      --input 09_GENOMES \
      --output-directory 09_GENOMES/checkM2 \
      -x fa \
      --threads 4 
```

### GTDB-tk

```bash 
conda activate gtdbtk

gtdbtk classify_wf \
      --genome_dir 09_GENOMES \
      --out_dir 09_GENOMES/GTDB \
      -x fa \
      --cpus 4 \
      --skip_ani_screen
```

## Automatic binning with SemiBin2

SemiBin2 is one of the several automatic binning algorithms published. Whether is good, better than others or the worst one available, you have to judge yourself.  
If you want learn more, there is a [pre-print available](https://www.biorxiv.org/content/10.1101/2023.01.09.523201v1.full). More practical reading can be found from the documentation: [https://semibin.readthedocs.io/en/latest/](https://semibin.readthedocs.io/en/latest/).  

SemiBin2 uses self-supervised learning and has some pre-trained models, which makes to computation faster. It requires as input the results from mapping reads back to assembly (sorted and indexed bam files, which we already have) and the assembly (which we also have).

```bash
cd ~/Physalia_EnvMetagenomics_2023
mkdir 10_SEMIBIN
```

Depending on the data you're analysing run the right code block below.  

__WWTP:__

```bash
export environment=wastewater
export sample=Sample3
```

__Tundra:__

```bash
export environment=soil
export sample=m12208
```

Run SemiBin

```bash
conda activate SemiBin

SemiBin single_easy_bin \
        --input-fasta 08_ANVIO/contigs.fasta \
        --input-bam 08_ANVIO/${sample}.bam \
        --sequencing-type=short_read \
        -o 10_SEMIBIN \
        --environment $environment \
        --threads 4
```

Prepare a file for anvi'o

```bash
cd 10_SEMIBIN

for file in output_bins/*.fa; do 
   for line in $(grep ">" $file); do 
      file=$(basename ${file%.fa})
      file=${file/./_}
      echo -e $line"\t"${file}
   done
done |sed 's/>//g' > semibin_results.txt
```

And import the binning results to anvi'o as a new collection called `SemiBin`.

```bash
cd ~/Physalia_EnvMetagenomics_2023 

anvi-import-collection \
   -c 08_ANVIO/CONTIGS.db \
   -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
   -C SemiBin \
   --contigs-mode \
   10_SEMIBIN/semibin_results.txt
```

Visualize the binning results in anvi'o. Remember to define the right port.

```bash 
conda activate anvio
export ANVIOPORT=

anvi-interactive \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --server-only \
    --port-number $ANVIOPORT
```

And when anvi'o is running, you can load the SemiBin collection under `Bins` and `Load bins collection`.

## Targeted functional analysis of MAGs

There are several approaches we can take to annotate our MAGs and find, for example, which kind of metabolic pathways they encode.  
At this we point, we stop doing metagenomics and start doing genomics, so in one way we have reached the end of our workshop.  
Everything you do next, depends a lot on what are you trying to use metagenomics for.  
Below we give some examples of how you can annotate in MAGs in `anvi'o`.  

### Broad-scale annotation

Here we can use three databases/approaches that `anvi'o` offers us to annotate our contigs/MAGs.  

* [COG](https://www.ncbi.nlm.nih.gov/research/cog)  
* [KEGG](https://www.genome.jp/kegg/kegg2.html)
* [Pfam](http://pfam.xfam.org)

We have annotated our contigs using the COG database when we are preparing our files for `anvi'o`.  
Unfortunately annotating against KEGG and Pfam would take a lot of time, so we won't do them now.  
But below are examples on how you could do this:   

```bash
conda activate anvio

# Annotate against COG
anvi-run-ncbi-cogs -c 08_ANVIO/CONTIGS.db -T 4

# Annotate against KEGG
anvi-run-kegg-kofams -c 08_ANVIO/CONTIGS.db -T 4

# Annotate against Pfam
anvi-run-pfams -c 08_ANVIO/CONTIGS.db -T 4
```

And to export the annotations we could do:  

```bash
anvi-export-functions -c 08_ANVIO/CONTIGS.db -o 08_ANVIO/annotation.txt
```

Or even better, we can summarise our collection again:  

```bash
anvi-summarize \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --collection-name Final \
    --output-dir 08_ANVIO/SUMMARY_Final_with_annotation
```

And look at the annotation for each of our individual MAGs/bins.  

### Looking for specific genes with HMMs

Another approach we can use in `anvi'o` is to run HMMs specifically for gene(s) of interest.  
In this example we will search for the [mcrA](https://www.genome.jp/dbget-bin/www_bget?K00399+K00400+K00401+K00402+K03421+K03422+2.8.4.1+R04541) gene, which is the key gene for methanogenesis.  

```bash
conda activate anvio

# Run HMMs
anvi-run-hmms -c 08_ANVIO/CONTIGS.db --hmm-profile-dir ~/Share/Databases/mcrA_HMM -T 4
```

And to export the annotation we could do:  

```bash
anvi-script-get-hmm-hits-per-gene-call -c 08_ANVIO/CONTIGS.db --hmm-source mcrA_HMM -o 08_ANVIO/mcrA_HMM.txt
```

Or even better, we can summarise our collection again:  

```bash
anvi-summarize \
    -c 08_ANVIO/CONTIGS.db \
    -p 08_ANVIO/SAMPLES-MERGED/PROFILE.db \
    --collection-name Final \
    --output-dir 08_ANVIO/SUMMARY_Final_with_mcrA_hmm
```

Can you find the mcrA gene in your MAGs?  
If you look at the `GTDB-Tk` results, can we say to which lineage the `mcrA` MAG(s) belong(s)?  
Is this metabolic capability already known for this lineage?  
Time to start writing the manuscript!! :)
