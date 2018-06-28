# README

This repository outlines how Illumina MiSeq COI metabarcodes are processed by Teresita M. Porter. **SCVUC** refers to the programs, algorithms, and reference datasets used in this data flow: **S**EQPREP, **C**UTADAPT, **V**SEARCH, **U**NOISE, **C**OI classifier.

## How to cite

If you use this dataflow or any of the provided scripts, consider citing the CO1 classifier paper (Porter & Hajibabaei, 2018 Sci Rep) if you use it and provide a link to this page https://github.com/EcoBiomics-Zoobiome/SCVUC_COI_metabarcode_pipeline in your publication.

## Outline

[Part I - Link to raw files](#part-i---link-to-raw-files)  
[Part II - Forward and reverse read number check](#part-ii---forward-and-verse-read-number-check)  
[Part III - Read pairing](#part-iii---read-pairing)   
[Part IV - Primer trimming](#part-iv---primer-trimming)  
[Part V - Dereplication](#part-v---dereplication)  
[Part VI - Denoising](#part-vi---denoising)  
[Part VII - Taxonomic assignment](#part-vii---taxonomic-assignment)  
[Implementation notes](#implementation-notes)  
[References](#references)  

## Part I - Link to raw files

This pipeline is meant to process Illumina paired-end reads from COI metabarcoding.  To save space in my directory, I create symbolic links to the raw .gz files.  The command linkfiles calls the script link_files.sh

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

I remove primers using the program CUTADAPT (Martin, 2011) available at http://cutadapt.readthedocs.io/en/stable/index.html .  This is a two-step process that first removes forward primers, takes the output, then removes the reverse primers from paired reads.  I run this command with GNU parallel using as many cores as possible (Tang, 2011).  GNU parallel is available at https://www.gnu.org/software/parallel/ .  The forward primer is trimmed with the -g flag.  I use default settings but require a minimum length after trimming of at least 150 bp, minimum read quality of Phred 20 at the ends of the sequences, and I allow a maximum of 3 N's.  I get read stats by running the gz_stats command described in Part II.  The reverse primer is inidicated with the -a flag and the primer sequence should be reverse-complemented when analyzing paired-reads.  CUTADAPT will automatically detect compressed fastq.gz files for reading and will convert these to .fasta.gz files based on the file extensions provided.  I get read stats by running the fasta_gz_stats command that calls the run_bash_fasta_gz_stats.sh script.  Therein the stats3 command links to the fasta_gz_stats.plx script.

```linux
ls | grep gz | parallel -j 20 "cutadapt -g <INSERT FOWARD PRIMER SEQ>  -m 150 -q 20,20 --max-n=3 --discard-untrimmed -o {}.Ftrimmed.fastq.gz {}"
gz_stats gz > Ftrimmed.stats
ls | grep .Ftrimmed.fastq.gz | parallel -j 20 "cutadapt -a <INSERT REVCOMP REVERSE PRIMER SEQ> -m 150 -q 20,20 --max-n=3  --discard-untrimmed -o {}.Rtrimmed.fasta.gz {}"
fasta_gz_stats gz > Rtrimmed.stats
```

## Part V - Dereplication

I prepare the files for dereplication by adding sample names parsed from the filenames to the fasta headers using the rename_all_fastas command that calls the run_rename_fasta.sh.  Therein the rename_fasta command calls the rename_fasta_gzip.polx script.  The results are concatenated and compressed.  The outfile is cat.fasta.gz .  I change all dashes with underscores in the fasta files using vi.  This large file is dereplicated with VSEARCH (Rognes et al., 2016) available at https://github.com/torognes/vsearch .  I use the default settings with the --sizein --sizeout flags to track the number of reads in each cluster.  I get read stats on the unique sequences using the stats_uniques command that calls the run_fastastats_parallel_uniques.sh script.  I count the total number of reads that were processed using the read_count_uniques command that calls the get_read_counts_uniques.sh script.

```linux
rename_all_fastas Rtrimmed.fasta.gz
vi -c "%s/-/_/g" -c "wq" cat.fasta.gz
vsearch --threads 10 --derep_fulllength cat.fasta.gz --output cat.uniques --sizein --sizeout
stats_uniques
read_count_uniques
```

## Part VI - Denoising

I denoise the reads using USEARCH with the UNOISE2 algorithm (Edgar, 2016) available at https://www.drive5.com/usearch/ .  With this program, denoising involves removing sequences with putative sequencing errors, PhiX sequences, putative chimeric sequences, as well as low frequency reads (just singletons and doubletons here).  This step can take quite a while to run for large files and I like to submit as a job on its own or use linux screen when working interactively so that I can detach the screen.  I get ESV stats using stats_denoised that links to run_fastastats_parallel_denoised.sh.  Therein the command stats links to fasta_stats_parallel.plx .  I get a count of reads retained in ESVs using the read_count_denoised command that links to get_read_counts_denoised.sh .  I generate an ESV/OTU table by mapping the primer-trimmed reads in cat.fasta to the ESVs in cat.denoised using an identity cutoff of 1.0 .

```linux
usearch -unoise2 cat.uniques -fastaout cat.denoised -minampsize 3 > log
stats_denoised
read_count_denoised
usearch -usearch_global cat.fasta -db cat.denoised -strand plus -id 1.0 -otutabout cat.denoised.table
```

## Part VII - Taxonomic assignment

I make taxonomic assignments using the RDP Classifier (Wang et al., 2007) available at https://sourceforge.net/projects/rdp-classifier/ .  I use this with the COI files ready to be used with the classiier (Porter & Hajibabaei, 2018 Sci Rep) available at https://github.com/terrimporter/CO1Classifier/releases .  This step can take a while depending on the filesize so I like to submit this as a job on its own or using Linux screen so that I can safely detach the session while it is running.  I like to map read number from the ESV/OTU table to the taxonomic assignments using the add_abundance_to_rdp_out3.plx script.

```linux
java -Xmx8g -jar /path/to/rdp_classifier_2.12/dist/classifier.jar classify -t /path/to/rRNAClassifier.properties -o cat.denoised.out cat.denoised
perl add_abundance_to_rdp_out3.plx cat.denoised.table cat.denoised.out
```

I like to use the MINIMUM recommended cutoffs for bootstrap support values according to fragment size and rank described in Porter & Hajibabaei, 2018 Sci Rep.  Use your own judgement as to whether these should be increased according to how well represented your target taxa are in the reference set.  This can be determined by exploring the original reference files used to train the classifier that is also available at https://github.com/terrimporter/CO1Classifier/releases .

## Implementation notes

To keep the dataflow here as clear as possible, I have ommitted file renaming and clean-up steps.  I also use shortcuts to link to scripts as described above in numerous places.  This is only helpful if you will be running this pipeline often.  I describe, in general, how I like to do this here:

### Batch renaming of files

Note that I am using Perl-rename (Gergely, 2018) that is available at https://github.com/subogero/rename not linux rename.  I prefer the Perl implementation so that you can easily use regular expressions.  I first run the command with the -n flag so you can review the changes without making any actual changes.  If you're happy with the results, re-run without the -n flag.

```linux
rename -n 's/PATTERN/NEW PATTERN/g' *.gz
```

### File clean-up

At every step, I place outfiles into their own directory, then cd into that directory.  I also delete any extraneous outfiles that may have been generated but are not used in subsequent steps to save disk space.

### Symbolic links

Instead of continually traversing nested directories to get to files, I create symbolic links to target directories in a top level directory.  Symbolic links can also be placed in your ~/bin directory that point to scripts that reside elsewhere on your system.  So long as those scripts are executable (e.x. chmod 755 script.plx) then the shortcut will also be executable without having to type out the complete path or copy and pasting the script into the current directory.

```linux
ln -s /path/to/target/directory shortcutName
ln -s /path/to/script/script.sh commandName
```

## References

Edgar, R. C. (2016). UNOISE2: improved error-correction for Illumina 16S and ITS amplicon sequencing. BioRxiv. doi:10.1101/081257  
Gergely, S. (2018, January). Perl-rename. Retrieved from https://github.com/subogero/rename  
Martin, M. (2011). Cutadapt removes adapter sequences from high-throughput sequencing reads. EMBnet. Journal, 17(1), pp–10.  
Porter, T. M., & Hajibabaei, M. (2018). Automated high throughput animal CO1 metabarcode classification. Scientific Reports, 8, 4226.  
Rognes, T., Flouri, T., Nichols, B., Quince, C., & Mahé, F. (2016). VSEARCH: a versatile open source tool for metagenomics. PeerJ, 4, e2584. doi:10.7717/peerj.2584  
St. John, J. (2016, Downloaded). SeqPrep. Retrieved from https://github.com/jstjohn/SeqPrep/releases  
Tange, O. (2011). GNU Parallel - The Command-Line Power Tool. ;;Login: The USENIX Magazine, February, 42–47.  
Wang, Q., Garrity, G. M., Tiedje, J. M., & Cole, J. R. (2007). Naive Bayesian Classifier for Rapid Assignment of rRNA Sequences into the New Bacterial Taxonomy. Applied and Environmental Microbiology, 73(16), 5261–5267. doi:10.1128/AEM.00062-07  

## Acknowledgements

I would like to acknowedge funding from the Canadian government through the Genomics Research and Development Initiative (GRDI) EcoBiomics project.

Last updated: June 28, 2018
