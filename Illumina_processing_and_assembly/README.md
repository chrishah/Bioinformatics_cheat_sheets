# Welcome

In the following we will detail the usage of some commonly used tools associated with Illumina data processing and denovo genome assembly.

The commands below assume that you have Docker installed.

***Note***
> Generally, I have designed the commands so that the first three lines are instructions to docker and all following lines refer to the actual piece of software in the container, so if you happend to have the particular software installed globally on your machine, it should also work if you simply ommit the first three lines of the commands. Just make sure you adjust the paths to your input files.


## [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) - a tool for Illumina data quality trimming

The command assumes that your read data (`reads.1.fastq.gz` and `reads.2.fastq.gz`) are present in your present working directory.
```bash
docker run --rm \
-v $(pwd):/in -w /in \
chrishah/trimmomatic-docker:0.38 \
trimmomatic PE -phred33 -threads 2 \
reads.1.fastq.gz reads.2.fastq.gz -baseout trimmed.fastq LEADING:30 TRAILING:30 SLIDINGWINDOW:5:15 MINLEN:90
```

## [Flash](https://ccb.jhu.edu/software/FLASH/)- a tool for merging paired end Illumina data

The command assumes that your read data (`reads.1.fastq.gz` and `reads.2.fastq.gz`) are present in a directory called `/home/classdata/Day2`.
```bash
docker run --rm \
-v /home/classdata/Day2:/data -v $(pwd):/in -w /in \
chrishah/flash:1.2.11 \
flash -z -o testdata /data/reads.1.fastq.gz /data/reads.2.fastq.gz
```

## [Minia](http://minia.genouest.org) - a super fast de Bruijn graph denovo genome assembler

Run Minia with standard paired end data (the command assumes that your data are in a directory called `/home/classdata/Day2`).
```bash
docker run --rm \
-v /home/classdata/Day2:/data -v $(pwd):/in -w /in \
chrishah/minia:3.2.4 \
minia -in /data/reads.1.fastq.gz -in reads.2.fastq.gz -kmer-size 31 -abundance-min 3 -out minia_k31
```

Run Minia with a combination of paired end and single end data (specify pe data first, then se data).
```bash
docker run --rm \
-v /home/classdata/Day2:/data -v $(pwd):/in -w /in \
chrishah/minia:3.2.4 \
minia -in /data/testdata.notCombined_1.fastq.gz -in /data/testdata.notCombined_2.fastq.gz -in /data/testdata.extendedFrags.fastq.gz \
-kmer-size 31 -abundance-min 3 -out minia_merged_k31
```

## [Spades](http://bioinf.spbau.ru/en/spades) - a popular de Bruijn graph denovo assembler

Run Spades using paired end data (the command assumes that your data are in a directory called `/home/classdata/Day2`).
```bash
docker run --rm \
-v /home/classdata/Day2:/data -v $(pwd):/in -w /in \
chrishah/spades:v3.14.0 \
spades.py -o ./spades-assemble-only \
-1 /data/reads.1.fastq.gz -2 /data/reads.2.fastq.gz \
--only-assembler --checkpoints last -t 2 -m 4
```

Run Spades using paired end data, but first error correct the data with the built in correction module.
```bash
docker run --rm \
-v /home/classdata/Day2:/data -v $(pwd):/in -w /in \
chrishah/spades:v3.14.0 \
spades.py -o ./spades-ec-assemble \
-1 /data/reads.1.fastq.gz -2 /data/reads.2.fastq.gz \
--checkpoints last -t 2 -m 4
```

Run Spades using a simple combination of paired end and single end data (assumes data are in your present working directory).
```bash
docker run --rm \
-v $(pwd):/in -w /in \
chrishah/spades:v3.14.0 \
spades.py -o ./spades-pe-se \
-1 reads.1.fastq.gz -2 reads.2.fastq.gz \
-s reads.se.fastq.gz \ 
--only-assembler --checkpoints last -t 2 -m 4
```

## [Platanus](http://platanus.bio.titech.ac.jp/) - a popular de Bruijn graph assembler, tuned for highly heterozygous samples

Note that Platanus does not accept gzipped data (`reads.1.fastq.gz` and `reads.2.fastq.gz` in directory `/home/classdata/Day2`) and we'll need to decompress our data first.
```bash
#copy and gunzip data
cp /home/classdata/Day2/reads.* .
gunzip reads.*

#specify a prefix variable that will be used throughout the assembly (for convenience)
prefix=platanus-pe

#Platanus needs to be run in three stages (assemble, scaffold, gap close)

#initial assembly module
docker run --rm \
-v $(pwd):/in -w /in \
chrishah/platanus:v1.2.4 \
platanus assemble -o $prefix \
-f reads.1.fastq reads.2.fastq -t 2 -m 4

#scaffolding module
docker run --rm \
-v $(pwd):/in -w /in \
chrishah/platanus:v1.2.4 \
platanus scaffold -o $prefix \
-c $prefix\_contig.fa -b $prefix\_contigBubble.fa -IP1 reads.1.fastq reads.2.fastq -t 2

#gap closing module
docker run --rm \
-v $(pwd):/in -w /in \
chrishah/platanus:v1.2.4 \
platanus gap_close -o $prefix \
-c $prefix\_scaffold.fa -IP1 reads.1.fastq reads.2.fastq
```

## [Quast](http://bioinf.spbau.ru/quast) - a popular tool for assessing basic assembly metrics

Run Quast on three assemblies all present in your present working directory.
```bash
docker run --rm \
-v $(pwd):/in -w /in \
reslp/quast:5.0.2 \
quast.py assembly_minia.fasta assembly_spades.fasta assembly_platanus.fasta
```

Run Quast on three assemblies which are in three separate directories that are above your present working directory in the filesystem.
```bash
docker run --rm \
-v /home/ubuntu/Day2/Minia:/minia_data -v /home/ubuntu/Day2/spades:/spades_data -v /home/ubuntu/Day2/platanus:/platanus_data -v $(pwd):/in -w /in \
reslp/quast:5.0.2 \
quast.py /minia_data/assembly_minia.fasta /spades_data/assembly_spades.fasta /platanus_data/assembly_platanus.fasta
```

In the above case all assemblies are in subdirectories of one common directory (`/home/ubuntu/Day2`), so you could solve the bind-mounting also with a single mountpoint. Note how we adjust the path within the container.
```bash
docker run --rm \
-v /home/ubuntu/Day2:/data -v $(pwd):/in -w /in \
reslp/quast:5.0.2 \
quast.py /data/minia/assembly_minia.fasta /data/spades/assembly_spades.fasta /data/platanus/assembly_platanus.fasta
```

