# Welcome

In the following we will detail the usage of some commonly used tools associated with Illumina data processing and denovo genome assembly.

The commands below assume that you have Docker installed.

***Note***
> Generally, I have designed the commands so that the first three lines are instructions to docker and all following lines refer to the actual piece of software in the container, so if you happend to have the particular software installed globally on your machine, it should also work if you simply ommit the first three lines of the commands. Just make sure you adjust the paths to your input files.


## [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) - a tool for Illumina data quality trimming

The command assumes that your read data (`reads.1.fastq.gz` and `reads.2.fastq.gz`) are present in your present working directory.
```bash
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/trimmomatic-docker:0.38 \
trimmomatic PE -phred33 -threads 2 reads.1.fastq.gz reads.2.fastq.gz -baseout trimmed.fastq LEADING:30 TRAILING:30 SLIDINGWINDOW:5:15 MINLEN:120 \
ILLUMINACLIP:/usr/src/Trimmomatic/0.38/Trimmomatic-0.38/adapters/TruSeq3-PE.fa:2:30:10
```

## [Flash](https://ccb.jhu.edu/software/FLASH/)- a tool for merging paired end Illumina data

The command assumes that your read data (`reads.1.fastq.gz` and `reads.2.fastq.gz`) are present in your present working directory.
```bash
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/flash:1.2.11 \
flash -z -o testdata --threads 2 reads.1.fastq.gz reads.2.fastq.gz
```

## [Minia](http://minia.genouest.org) - a super fast de Bruijn graph denovo genome assembler

The commands assume that your read data (e.g. `reads.1.fastq.gz` and `reads.2.fastq.gz`) are in your current working directory.
Run Minia with standard paired end data.
```bash
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/minia:3.2.4 \
minia -in reads.1.fastq.gz -in reads.2.fastq.gz -kmer-size 31 -abundance-min 3 -out minia_k31
```

Run Minia with trimmed reads
```bash
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/minia:3.2.4 \
minia -in trimmed_1P.fastq -in trimmed_1U.fastq -in trimmed_2P.fastq -in trimmed_2U.fastq \
-kmer-size 31 -abundance-min 3 -out minia_trimmed_k31
```

Run Minia with merged and paired end reads
```bash
docker run \
--rm -v $(pwd):/in -w /in \ 
chrishah/minia:3.2.4 \
minia -in testdata.notcombined_1.fastq.gz -in testdata.notcombined_2.fastq.gz -in testdata.extendedfrags.fastq.gz \
-kmer-size 31 -abundance-min 3 -out minia_merged_k31
```

## [Spades](http://bioinf.spbau.ru/en/spades) - a popular de Bruijn graph denovo assembler

Run Spades using paired end data (the command assumes that your data are in your current working directory):
```bash
mkdir spades-assemble
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/spades:v3.14.0 \
spades.py -o ./spades-assemble \
-1 reads.1.fastq.gz -2 reads.2.fastq.gz \
--only-assembler -t 2 -m 4
```

Run Spades using paired end data, but first error correct the data with the built in correction module.
```bash
mkdir spades-ec-assemble
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/spades:v3.14.0 \
spades.py -o ./spades-ec-assemble \
-1 reads.1.fastq.gz -2 reads.2.fastq.gz \
-t 2 -m 4
```

Run Flash (see above) on spades errorcorrected reads
```bash
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/flash:1.2.11 \
flash -z -o spades_ec --threads 2 spades-ec-assemble/corrected/reads.1.fastq.00.0_0.cor.fastq.gz spades-ec-assemble/corrected/reads.2.fastq.00.0_0.cor.fastq.gz
```

Run Spades using a simple combination of paired end and single end data (assumes data are in your present working directory).
```bash
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/spades:v3.14.0 \
spades.py -o ./spades-ecd-merged \
-1 spades_ec.notCombined_1.fastq.gz -2 spades_ec.notCombined_2.fastq.gz \
-s spades-ec-assemble/corrected/reads._unpaired.00.0_0.cor.fastq.gz --merged spades_ec.extendedFrags.fastq.gz \
--only-assembler -t 2 -m 4
```

## [Platanus](http://platanus.bio.titech.ac.jp/) - a popular de Bruijn graph assembler, tuned for highly heterozygous samples

Note that Platanus does not accept gzipped data and we'd need to decompress our data first - in the below example I use the reads that came out of the trimmomatic run above.
First create a directory to which all Platanus files will go.
```bash
mkdir Platanus
```
Now do the assembly - Note that Platanus has three modules that need to be run consecutively.
```bash
#assembly module
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/platanus:v1.2.4 \
platanus assemble -o Platanus/platanus_raw -f trimmed_1P.fastq trimmed_2P.fastq -t 2 -m 4

#scaffolding module
docker run \
--rm -v $(pwd):/in -w /in \
-v $(pwd)/../../Illumina_data:/data -v $(pwd):/working -w /working \
chrishah/platanus:v1.2.4 \
platanus scaffold -o Platanus/platanus_raw -c Platanus/platanus_raw_contig.fa -b Platanus/platanus_raw_contigBubble.fa -IP1 trimmed_1P.fastq trimmed_2P.fastq -t 2

#gap closing module
docker run \
--rm -v $(pwd):/in -w /in \
chrishah/platanus:v1.2.4 \
platanus gap_close -o Platanus/platanus_raw \
-c Platanus/platanus_raw_scaffold.fa \
-IP1 trimmed_1P.fastq trimmed_2P.fastq \
-f trimmed_1U.fastq -f trimmed_2U.fastq
```

## [Quast](http://bioinf.spbau.ru/quast) - a popular tool for assessing basic assembly metrics

Run Quast on an assembly in your present working directory.
```bash
docker run \ 
--rm -v $(pwd):/in -w /in \
reslp/quast:5.0.2 \
quast.py --threads 2 spades-assemble/scaffolds.fasta
```

Run Quast on three assemblies all present in your present working directory.
```bash
docker run --rm \
-v $(pwd):/in -w /in \
reslp/quast:5.0.2 \
quast.py assembly_minia.fasta assembly_spades.fasta assembly_platanus.fasta
```
