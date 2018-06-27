# README

This repository outlines how Illumina MiSeq COI metabarcodes were processed for the GRDI: EcoBiomics - Soil Theme project by Teresita M. Porter.

## Outline

[Part I - File name cleanup](#part-i---file-name-cleanup)  

[Part II - Forward and reverse read number check](#part-ii---forward-and-verse-read-number-check)  

[Part III - Read pairing](#part-iii---read-pairing)   

[Part IV - Primer trimming](#part-iv---primer-trimming)  

[Part V - Dereplication](#part-v---dereplication)  

Part VI - Denoising

Part VII - Taxonomic assignment

## Part I - File name cleanup

To save space in my directory, I create symbolic links to the raw .gz files.  The command linkfiles calls the script link_files.sh

```linux
linkfiles
```

If necessary, I concatenate results from the same samples from 2 runs.

```linux
perl concatenate_gz.plx
```

## Part II - Forward and reverse read number check

I make sure that the number of reads in the forward R1 files are the same as those in the reverwe R2 files.  The command gz_stats calls the script run_fastq_gz_stats.sh.  Therein the stats2 command links to the fastq_g_stats.plx script.  The filename suffix that targets R1 and R2 files needs to be supplied as an argument.

```linux
gz_stats R1.fq.gz > R1.stats
gz_stats R2.fz.gz > R2.stats
```

## Part III - Read pairing

I pair forward and reverse reads using the SEQPREP program available at https://github.com/jstjohn/SeqPrep .  I use the default settings and specify a minimum quality of Phred 20 at the ends of the reads and an overlap of at least 25 bp.  The command pair runs the script xxx.  I check the read stats by running the gz_stats command described in Part II.

```linux
pair _R1.fq.gz _R2.fq.gz
gz_stats gz > paired.stats
```

## Part IV - Primer trimming

I remove primers using the program CUTADAPT (Martin, 2011) available at http://cutadapt.readthedocs.io/en/stable/index.html .  This is a two-step process that first removes forward primers, takes the output, then removes the reverse primers from paired reads.  I run this command with GNU parallel using as many cores as possible.  GNU parallel is available at https://www.gnu.org/software/parallel/ .  The forward primer is trimmed with the -g flag.  I use default settings but require a minimum length after trimming of at least 150 bp, minimum read quality of Phred 20 at the ends of the sequences, and I allow a maximum of 3 N's.  I get read stats by running the gz_stats command described in Part II.  The reverse primer is inidicated with the -a flag and the primer sequence should be reverse-complemented when analyzing paired-reads.  CUTADAPT will automatically detect compressed fastq.gz files for reading and will convert these to .fasta.gz files based on the file extensions provided.  I get read stats by running the fasta_gz_stats command that calls the run_bash_fasta_gz_stats.sh script.  Therein the stats3 command links to the fasta_gz_stats.plx script.

```linux
ls | grep gz | parallel -j 20 "cutadapt -g <INSERT FOWARD PRIMER SEQ>  -m 150 -q 20,20 --max-n=3 --discard-untrimmed -o {}.Ftrimmed.fastq.gz {}"
gz_stats gz > Ftrimmed.stats
ls | grep .Ftrimmed.fastq.gz | parallel -j 20 "cutadapt -a <INSERT REVCOMP REVERSE PRIMER SEQ> -m 150 -q 20,20 --max-n=3  --discard-untrimmed -o {}.Rtrimmed.fasta.gz {}"
fasta_gz_stats gz > Rtrimmed.stats
```

## Part V - Dereplication

I prepare the files for dereplication by adding sample names parsed from the filenames to the fasta headers using the rename_all_fastas command that calls the run_rename_fasta.sh.  Therein the rename_fasta command calls the rename_fasta_gzip.polx script.  The results are concatenated andcompressed.  The outfile is cat.fasta.gz .  I change all dashes with underscores in the fasta files using vi.  This large file is dereplicated with VSEARCH (Rognes et al., 2016) available at https://github.com/torognes/vsearch .  I use the default settings with the --sizein --sizeout flags to track the number of reads in each cluster.  I get read stats on the unique sequences using the stats_uniques command that calls the run_fastastats_parallel_uniques.sh script.  I count the total number of reads that were processed using the read_count_uniques command that calls the get_read_counts_uniques.sh script.

```linux
rename_all_fastas Rtrimmed.fasta.gz
vi -c "%s/-/_/g" -c "wq" cat.fasta.gz
vsearch --threads 10 --derep_fulllength cat.fasta.gz --output cat.uniques --sizein --sizeout
stats_uniques
read_count_uniques
```

## Part VI - Denoising

## Part VII - Taxonomic assignment

## Acknowledgements

I would like to acknowedge funding from the Canadian government through the Genomics Research and Development Initiative (GRDI) EcoBiomics project.

Last updated: June 27, 2018
