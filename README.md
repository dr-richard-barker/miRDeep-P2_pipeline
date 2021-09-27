# miRDeep-P2_pipeline

This is a general pipeline for performing miRNA prediction and differential expression analysis using small RNA sequencing (sRNA-seq) data of plant samples. In this pipeline, the tool for miRNA prediction from sRNA-seq data is miRDeep-P2 (http://www.srnaworld.com/miRDP2/index.html). The output of miRDeep-P2 is parsed by parse_miRDP2_prediction.pl script in this pipeline. The parsed predicted miRNA sequences are searched against known miRNA sequences from the public domain for annotation purpose. In differential expression analysis, the sRNA-seq read data are mapped to the predicted miRNA sequences using Bowtie (http://bowtie-bio.sourceforge.net/index.shtml) and the read count for each miRNA sequence is generated. Finally, with the miRNA count data, DESeq2 (https://bioconductor.org/packages/release/bioc/html/DESeq2.html) is used to perform data filtering, normalization and differential expression analysis. In addition, it is recommended to merge the predicted miRNA sequences with those present in the public domain, such as miRBase, for differential expression analysis, in order to increase the sensitivity. 

The details of using this pipeline are described as following. And the pipeline assumes that all files and the directory of this pipeline are in the same directory. It is also assumed that all the tools can readily be called in command line.

This pipeline was firstly used in the article, https://doi.org/10.1002/tpg2.20103.

## Computational environment

- LINUX command line system

- Perl (https://www.perl.org/)

- BioPerl modules including Bio::SeqIO, Bio::Seq and Bio::SearchIO (https://bioperl.org/INSTALL.html)

- miRDeep-P2 (http://www.srnaworld.com/miRDP2/index.html)

- blast (https://blast.ncbi.nlm.nih.gov/Blast.cgi?PAGE_TYPE=BlastDocs&DOC_TYPE=Download)

- Bowtie (http://bowtie-bio.sourceforge.net/index.shtml)

- samtools (http://www.htslib.org/)

- R and DESeq2 (https://bioconductor.org/packages/release/bioc/html/DESeq2.html)

## Installation of the pipeline

`git clone https://github.com/TF-Chan-Lab/miRDeep-P2_pipeline.git`

## Files

- Clean sRNA-seq data in .fq format for different samples, named `sample1.fq`, `sample2.fq` and etc.

  - The clean sRNA-seq data are usually generated from the raw data after adapter trimming using a proper tool, e.g. Trimmomatic (http://www.usadellab.org/cms/?page=trimmomatic). This pipeline assumes that the input sRNA-seq data are already trimmed for adpaters and clean.

- Reference genome sequence in `.fa` format, named `ref.fa`

  - Index file of the reference genome seqeunce generated by Bowtie using the following command
  
  `bowtie-build ref.fa ref.fa`

- Known miRNA sequences from the public domian in `.fa` format, e.g `mature.fa` from miRBase (http://www.mirbase.org/ftp.shtml)

  - If sequences in miRBase are used, the `mature.fa` file needs to be properly preprocessed as following.
  
    - To convert all "U" nucleotide to "T"
    
    `perl ./miRDeep-P2_pipeline/script/fasta_U2T.pl mature.fa mature_wo_U.fa`
  
    - To collapse unique mature sequences in mature_wo_U.fa
    
    `perl ./miRDeep-P2_pipeline/script/unique_fasta_v1.2.pl mature_wo_U.fa mature_wo_U_uniq.fa mature_uniq`

  - Index file of the known miRNA sequences, e.g. the miRBase sequences, generated by blast using the following command
  
  `makeblastdb -in mature_wo_U_uniq.fa -dbtype 'nucl'`

## 1. miRNA prediction using miRDeep-P2

The pipeline has been tested with miRDeep-P2 (v1.1.4), which is used for demonstration, and should be compatible with the other versions. Please refer to the tool's page for isntallation. To run miRDeep-P2, the following command can be used.

`miRDP2-v1.1.4_pipeline.bash -g ref.fa -x ref.fa -q -b fastq_list.txt -o result`

The file `fastq_list.txt` contains the absolute path to each of the .fastq file, with one line for one file. The output directory `result` contains the resulted results in its subdirectory for all samples, named sample1, sample2 and etc. Particularly, the files with suffix `_predictions`, e.g. `result/sample1/sample1_predictions`, are parsed to extract the predicted miRNA seqeunces and their corresponding precursor information. These files are parsed using the following command.

`perl ./miRDeep-P2_pipeline/script/parse_miRDP2_prediction.pl miRDP2_predictions_list.txt miRDP2`

The file `miRDP2_predictions_list.txt` contains file names of the `_predictions` files, e.g. `sample1_predictions`, to be analyzed together. Each line contains one file name. There are two resulted files, named `miRDP2_mature.fa` and `miRDP2_arms.txt`, respectively, with `miRDP2` as the prefix. The sequence IDs in `miRDP2_mature.fa` always begin with the prefix `miRDP2_mature_`, followed by order numbers of the unique miRNA mature sequences. The genomic orgin, precursor and primary miRNA sequence inforamtion can be found in the description field of each sequence entry in `miRDP2_mature.fa`. There are three tab-delineated columns in the `miRDP2_arms.txt` file. The first two columns refer to paris of mature miRNA sequences that are predicted from the same precursors. The third column refer to information of the genomic origin, precursor and primary sequences, in the format of `genomic_origin:precursor_sequence:primary_sequence;`. There may be multiple genomic origins, precursor or primary sequences corresponding to the same pairs of miRNA mature sequences.

## 2. Annotation of miRNA

After getting the predicted miRNA sequences, it is informative to annotated the sequences with those already deposited in the public domain, e.g. miRBase. In this pipeline, the predicted miRNA sequences are first searched against the database of interest using blast using the following commands.

`blastn -db mature_wo_U_uniq.fa -query miRDP2_mature.fa -out miRDP2_mature_blastn.txt -word_size 4 -num_alignment 1`

`perl ./miRDeep-P2_pipeline/script/general_blast_parser.pl miRDP2_mature_blastn.txt miRDP2_mature_blastn_parsed.txt`

`perl ./miRDeep-P2_pipeline/script/parse_parsed_blast_known_plants.pl miRDP2_mature_blastn_parsed.txt miRDP2_mature`

The search results are then parsed to classify the predicted miRNA sequences into the following three categories.

- Known miRNA that is already in the database of interest

- miRNA variant that contains at most 2 mismatched nucleotides but no insertion or deletion with respect to sequences in the database of interest

- Novel miRNA that does not fall into any of the above two categories

Two files are generated at this point, namely `miRDP2_mature_known.txt` and `miRDP2_mature_variant.txt`. The following command is used to get the IDs for the known miRNA sequences.

`cut -f2 miRDP2_mature_known.txt | sort | uniq > miRDP2_mature_known_id.txt`

It is a bit tricky to get the IDs for the miRNA variant sequences and here comes the commands.

`cut -f2 miRDP2_mature_variant.txt | sort | uniq > temp.txt`

`perl ./miRDeep-P2_pipeline/script/filter_lines_by_key_words_list.pl temp.txt miRDP2_mature_variant_id.txt miRDP2_mature_known_id.txt 0`

`rm temp.txt`

The remaining predicted miRNA sequences are novel miRNAs.

## 3. Differnetial expression anlaysis

To perform differential expression analysis, the clean read is first mapped to the reference miRNA sequences. In step **1**, a miRNA sequences file, `miRDP2_mature.fa`, is generated. This file can be used as the reference for mapping. Alternatively, a combination of sequences in `miRDP2_mature.fa` and those present in the pubic domain, e.g. miRBase, but missed by miRDeep-P2 can be also served as the reference. In the following analysis, the file of reference miRNA sequences is named `miRNA_ref.fa` and indexed using the following command.

`bowtie-build miRNA_ref.fa miRNA_ref.fa`

Then each sample, e.g. `sample1.fq`, is mapped to the reference miRNA sequences using the following command.

`bowtie -v 0 --norc -S miRNA_ref.fa sample1.fq | samtools view -Sb - > sample1.bam`

With the alignment `.bam` file, the number of reads perfectly mapped to each reference miRNA sequence is generated using the following command.

`perl ./miRDeep-P2_pipeline/script/bam2ref_counts.pl -bam sample1.bam -f miRNA_ref.fa > sample1_count.txt`

To combine the read counts data for each sample into a table, the following command is used.

`perl ./miRDeep-P2_pipeline/script/combine_htseq_counts.pl count_list.txt count_table.txt`

The file, `count_list.txt`, contains files names of the count files, e.g. `sample1_count.txt`, to be combined and analyzed together. Each line contains one file name. In the output file, `count_table.txt`, the last four columns represent the mean, median, variance and coefficient of variation. Prior to differential expression analysis, the last four columns in `count_table.txt` should be removed. The count table is now ready for normalization and differential expression analysis using DESeq2. It is recommended to follow the **Beginner's guide to using the DESeq2 package** (https://bioc.ism.ac.jp/packages/2.14/bioc/vignettes/DESeq2/inst/doc/beginner.pdf).
