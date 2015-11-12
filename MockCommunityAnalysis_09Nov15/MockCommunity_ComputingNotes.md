# Mock community analysis Ashley Shade
## 09 Nov 2015
### Problem:  OTU overinflation in Centralia 2014 16S amplicon dataset
* We have ~300,000 OTUs  this is >20X more than we expect
* Previously we used PANDAseq to merge paired end reads with a really high t threshold = 0.9, and then used an [open-reference](http://www.ncbi.nlm.nih.gov/pubmed/25177538) with QIIME's usearch61 to pick OTUs
* There is a [new paper by Edgar](http://bioinformatics.oxfordjournals.org/content/31/21/3476.full) showing that, even with our high t value in PANDAseq, there is a merging of random sequences and a lot of missing of many errors with PANDAseq (errors > 8), especially in among rare taxa.  This paper suggests using a UNOISE pipeline, implemented in usearch8.1 offers a very good alternative.
* the only drawback to this is that we cannot use the open-reference approach to pick OTUs, and will have to pick them de novo in usearch8.1
* ...unless we can do the quality filtering using UNOISE but then move the OTU picking to the QIIME environment
* another thought is that, given the very few OTUs that match the database, we could make a strong case that there is not much information lost in using the de novo approach, BUT then it will make it difficult for us to compare these OTUs with subsequent analyses, which we anticipate because we've already collected another field year's worth of soil.
*  approach:  we're going to use our mock community to ground-truth our merging/quality filtering and OTU picking.  

### Getting started
* Made a new subdirectory for working on HPCC:   `/WorkingSpace/Shade_MockCom`

```
#Copy the file and to working directory
cp ../../Shade/20141230_16Stag_Centralia/20141230_B_16S_PE/Mock_Cmty_TCCTCTGTCGAC_L001_R* .

#unzip copy
gunzip *gz
```

* load usearch
```
module load USEARCH/8.0.1623
```

* Here are the commands provided from the supplementary files of Egar and 2015   

```
#USEARCH/M:   
usearch -fastq_mergepairs fwd.fq -reverse rev.fq -fastqout merged.fq

#USEARCH/F:   
usearch -fastq_filter input.fq -fastq_maxee 1.0 -fastqout filtered.fq

#UNOISE:   
usearch -derep_fulllength reads.fastq -fastqout uniques.fastq -sizeout

usearch -cluster_fast uniques.fastq -centroids_fastq denoised.fq -id 0.9 -maxdiffs 5 \-abskew 10 -sizein -sizeout -sort size
```

## 10 Nov 2015
### Backing up:  Mock community information
* Here is the LabGuru link to Sang-Hoon's construction [notes](https://my.labguru.com/knowledge/projects/81/milestones/261/experiments/1791)
* Mock community contains 6 strains, combined in an even concentration of 100,000 16S copies per uL.

| Strain       | No. 16S copies        |
| ------------- |:-------------:|
| D. radiodurans    | 3 |
| B. thailandensis   | 4 |
| B. cereus   | 13 |
| P. syringae  | 5 |
| F. johnsoniae  | 6 |
| E. coli MG1655  | 8 |

* Data:  Illumina MiSeq 2x150 bp paired-end reads with ~50 bp overlap expected
* HPCC has installed: USEARCH/8.0.1623 - submitted a ticket to update to the latest version (v.8.1.1803 - which seems to have most of the quality filter/merging updates described in the above 2015 paper)

```
#Step 1:  merge paired-end reads

#note: using older version of usearch for now, until HPCC updates
#note maxee value set to 1.0, as default recommended in Edgar and Flyvbjerg 2015.  They recommend informing maxee by the Q value distribution.

[shadeash@dev-intel14 Shade_MockCom]$ usearch -fastq_mergepairs Mock_Cmty_TCCTCTGTCGAC_L001_R1_001.fastq -reverse Mock_Cmty_TCCTCTGTCGAC_L001_R2_001.fastq -fastaout Mock.fa -relabel @ -fastq_merge_maxee 1.0
usearch v8.0.1623_i86linux32, 4.0Gb RAM (264Gb total), 20 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: johnj@msu.edu

00:01  70Mb    0.1% Converting
WARNING: Max OMP threads 1

00:14  71Mb  100.0% 78.8% merged

Merged length min 150, lowq 253, median 253, mean 253.0, higq 253, max 284
     Merge length    Reads
-----------------  -------
    150 -     174       97  
    175 -     199       38  
    200 -     224       47  
    225 -     249       80  
    250 -     274   321665  ********************************
    275 -     299       19  

    338399  Pairs
    266811  Converted (78.8%)
    211830  Exact overlaps (62.60%)
     16171  Not aligned (4.78%)
     55050  Exp.errs. too high (max=1.0) (16.27%)
        85  Too many gaps (max=0) (0.03%)
       257  Gaps
    321706  Mismatches
     86887  Fwd errs
    234819  Rev errs
       282  Staggered

WARNING: Option -relabel ignored
```
* Pairs is the total number of read pairs input, and the Converted is the number of "good" merges from those.  The Not aligned, Exp errs too high, and Too many Gaps are the merges that were thrown out.  So, Good merges (Converted) + Bad merges (Not aligned, Exp. errors to high, Too many gaps) =100%  .. almost 99.88% here.  But, the difference is made up by the "Staggered" value - 282 reads.  After reading [this](http://drive5.com/usearch/manual/cmd_fastq_mergepairs.html), seems like staggered is the number of "good" merges that have not-perfect overlap on the forward and reverse ends.
* So: Good merges (Converted + Staggered) + Bad merges (Not alighted, Exp. err too high, too many gaps) = total pairs

* Generally considering these results, it seems that the majority of our reads merged to the length expected, 250-274 bp long.  Though merges outside of these range are noted, there are very few, and, to be conservative, we may want to remove them from the dataset by setting  `-fastq_minmergelen 250` and `-fastq_maxmergelen 274`.  We may also want to remove the "staggered" reads, which can be forced in the new version of usearch (using the `-fastq_nostagger` option).

* I want to test the influence of changing maxee on the merging, the paper uses up to maxee 25 for some tests.  The larger number decreases error sensitivity.

|maxee| Converted| Exp. errs. too high|
| ------------- |:-------------:| :-----:|
| 1| 266811 (78.8%)| 55050 (16.27%)|
|1.5|  290589 (85.9%)| 31272 (9.24%) |
|2| 304567 (90.0%)| 17294 (5.11%)|
|5| 320535 (97%)|  1326 (0.39%)|


* Majority merge length (250-264), Pairs, Exact overlaps, Not aligned, Too many gaps (max=0),  Gaps, Mismatches, Fwd errs, Rev errs, Staggered values are the same across different maxee values.  
* Sweet update!  HPCC *ALREADY* has updated usearch right freakin now.  Sweet sweet sweet.
```
module load USEARCH/8.1.1803

[shadeash@dev-intel14 Shade_MockCom]$ usearch -fastq_mergepairs Mock_Cmty_TCCTCTGTCGAC_L001_R1_001.fastq -fastqout Mock.fastq -relabel @ \ -fastq_merge_maxee 1.0 -fastq_minmergelen 250 -fastq_maxmergelen 274 -fastq_nostagger

usearch v8.1.1803_i86linux32, 4.0Gb RAM (264Gb total), 20 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:00  71Mb    0.1% Converting
WARNING: Max OMP threads 1

00:04  71Mb  100.0% 77.9% merged
    338399  Pairs (338.4k)      
    263722  Merged (263.7k, 77.93%)
    214313  Alignments with zero diffs (63.33%)
     11077  Fwd tails Q <= 2 trimmed (3.27%)
     28693  Rev tails Q <= 2 trimmed (8.48%)
        43  Fwd too short (< 64) after tail trimming (0.01%)
      1194  Rev too short (< 64) after tail trimming (0.35%)
     17710  No alignment found (5.23%)
         0  Alignment too short (< 16) (0.00%)
       260  Merged too short (< 250)
        19  Merged too long (> 274)
     55175  Exp.errs. too high (max=1.0) (16.30%)
       276  Staggered pairs (0.08%) discarded
     46.77  Mean alignment length
    253.00  Mean merged read length
      0.11  Mean fwd expected errors
      0.34  Mean rev expected errors
      0.26  Mean merged expected errors
```

* I used FASTQC to assess the overall quality of the usearch merging
```
#sanity check - check that the number of merged seqs matches expectations from usearch merge output table (above)

grep -c "@Mock" Mock.fastq
#263722

#load fastqc tools on HPCC
module load fastqc
#FastQC/0.11.3

fastqc Mock.fastq
```

* open new terminal, move fastqc output to desktop to view html summary
```
scp shadeash@hpcc.msu.edu:/mnt/research/ShadeLab/WorkingSpace/Shade_MockCom/Mock_fastqc.html .

open Mock_fastqc.html

#actually looks really great - all Q scores > 32, avg = 37
```

* moving on:  UNOISE algorithm
* using default of d=5, w=10; note that w may be fine-tuned for mock communities as per Edgar and Flyvbjerg 2015
```
#UNOISE  
#step 1:  make a database of all unique sequences
[shadeash@dev-intel14 Shade_MockCom]$ usearch -derep_fulllength Mock.fastq -fastqout uniques_Mock.fastq -sizeout

#output file:  uniques_Mock.fastq


#step 2:  cluster sequences that are likely derived from the same parent
[shadeash@dev-intel14 Shade_MockCom]$ usearch -cluster_fast uniques_Mock.fastq -centroids_fastq denoised.fq -id 0.9 -maxdiffs 5 -abskew 10 -sizein -sizeout -sort size
usearch v8.1.1803_i86linux32, 4.0Gb RAM (264Gb total), 20 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:00  82Mb  100.0% Reading uniques_Mock.fastq
00:00  48Mb Pass 1...
WARNING: Max OMP threads 1

74199 seqs (tot.size 263722), 74199 uniques, 55891 singletons (75.3%)
00:00  53Mb Min size 1, median 1, max 36521, avg 3.55
00:00  53Mb  100.0% Writing
00:00  57Mb done.           
00:00  57Mb Sort size... done.
00:16 127Mb  100.0% 33611 clusters, max size 62184, avg 7.8
00:16 127Mb  100.0% Writing centroids to denoised.fq       

      Seqs  74199 (74.2k)
  Clusters  33611 (33.6k)
  Max size  62184 (62.2k)
  Avg size  7.8
  Min size  1
Singletons  24715 (24.7k), 33.3% of seqs, 73.5% of clusters
   Max mem  127Mb
      Time  17.0s
Throughput  4364.6 seqs/sec.

#output file:  denoised.fq
```
* From the above data, there are 33,611 "clusters", when we should have only ~6.
* Removing singleton sequences results in 8,896 clusters
* We could fine-tune w to be smaller... changed w (abskew) from 10 to 1.

```
[shadeash@dev-intel14 Shade_MockCom]$ usearch -cluster_fast uniques_Mock.fastq -centroids_fastq denoised.fq -id 0.9 -maxdiffs 5 -abskew 1 -sizein -sizeout -sort size
usearch v8.1.1803_i86linux32, 4.0Gb RAM (264Gb total), 20 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:00  82Mb  100.0% Reading uniques_Mock.fastq
00:00  48Mb Pass 1...
WARNING: Max OMP threads 1

74199 seqs (tot.size 263722), 74199 uniques, 55891 singletons (75.3%)
00:01  53Mb Min size 1, median 1, max 36521, avg 3.55
00:01  53Mb  100.0% Writing
00:01  57Mb done.           
00:01  57Mb Sort size... done.
00:10  93Mb  100.0% 15120 clusters, max size 61999, avg 17.4
00:10  93Mb  100.0% Writing centroids to denoised.fq        

      Seqs  74199 (74.2k)
  Clusters  15120 (15.1k)
  Max size  61999 (62.0k)
  Avg size  17.4
  Min size  1
Singletons  6768, 9.1% of seqs, 44.8% of clusters
   Max mem  93Mb
      Time  9.00s
Throughput  8244.3 seqs/sec.
```
* Indeed, decreasing w also decreases mock community clusters... but tuning this parameter on the mock community is potentially unhelpful for full community OTU picking, as stated in the Edgar and Flyvbjerg 2015 piece.

* taking a step back:  it is strongly recommended that singleton sequences are removed. Where should this happen?  Probably before UNOISE steps to improve computational efficiency.  I used the suggested script from the usearch manual  (here)[http://drive5.com/usearch/manual/singletons.html]

```
[shadeash@dev-intel14 Shade_MockCom]$ usearch -sortbysize uniques_Mock.fastq -fastqout Mock_nosigs.fastq -minsize 2
usearch v8.1.1803_i86linux32, 4.0Gb RAM (264Gb total), 20 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:01  82Mb  100.0% Reading uniques_Mock.fastq
00:01  48Mb Getting sizes                     
00:01  49Mb Sorting 18308 sequences
00:01  49Mb  100.0% Writing output
```

* now, will continue with UNOISE step 2 -  to pre-cluster sequences that are likely derived from the same parent
```
[shadeash@dev-intel14 Shade_MockCom]$ usearch -cluster_fast uniques_Mock_nosigs.fastq -centroids_fastq denoised.fq -id 0.9 -maxdiffs 5 -abskew 10 -sizein -sizeout -sort size
usearch v8.1.1803_i86linux32, 4.0Gb RAM (264Gb total), 20 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:00  51Mb  100.0% Reading uniques_Mock_nosigs.fastq
00:00  17Mb Pass 1...
WARNING: Max OMP threads 1

18308 seqs (tot.size 207831), 18308 uniques, 0 singletons (0.0%)
00:00  19Mb Min size 2, median 2, max 36521, avg 11.35
00:00  19Mb  100.0% Writing
00:00  23Mb done.           
00:00  23Mb Sort size... done.
00:02  47Mb  100.0% 8898 clusters, max size 56854, avg 23.4
00:02  47Mb  100.0% Writing centroids to denoised.fq       

      Seqs  18308 (18.3k)
  Clusters  8898
  Max size  56854 (56.9k)
  Avg size  23.4
  Min size  2
Singletons  0, 0.0% of seqs, 0.0% of clusters
   Max mem  51Mb
      Time  2.00s
Throughput  9154.0 seqs/sec.

#output files:  denoised.fq
```
* from the above results, we now have ~9K pre-clusters, which is better but still not anywhere close to the ~6 we are ultimately expecting after OTU picking
* where in this pipeline should we check for chimeras?  usually, those are determined during/after OTU picking...which should come next

```
#step 3 - pick OTUs with UPARSE, includes chimera detection

[shadeash@dev-intel14 Shade_MockCom]$ usearch -cluster_otus denoised.fq -otus mock_denoised_otus.fa -relabel OTU_ -sizeout -uparseout results.txt
usearch v8.1.1803_i86linux32, 4.0Gb RAM (264Gb total), 20 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:05  52Mb  100.0% 1717 OTUs, 4497 chimeras (50.5%)
#output files: results.txt, mock_denoised_otus.fa
```
* interesting results - we have >1K OTUs (when we should have 6).  However, this is a lot better than what we were doing (previously)[https://github.com/ShadeLab/DataTimeNotes/blob/master/20150429_DataTime_OTUInflationIssues.md].  
* Many chimeras detected - 50%.  This would be really high for a non-mock community, but I'm not sure for this mock community.  
* For the mock community, we can also run an additional ref-uchime chimera detection, though I'm not sure that I would recommend this step for the full dataset because of the novel diversity that we expect.

## 11 Nov 2015
### Continuing with mock community and usearch tools for decreasing OTU inflation
* If we inspect the top of the results file from the otu clustering (uparse algorithm), we see that the top 6 OTUs have the most sequences, and after that we go from OTUs with >12938 sequences to OTUs with <1000 sequences:
```
[shadeash@dev-intel14 Shade_MockCom]$ more results.txt
Mock.16;size=56854;	otu	*	*	OTU_1
Mock.25;size=30282;	otu	70.8	*	OTU_1	OTU_2
Mock.11;size=28182;	otu	80.6	*	OTU_1	OTU_3
Mock.18;size=14434;	otu	76.7	*	OTU_1	OTU_4
Mock.37;size=14008;	otu	89.3	*	OTU_3	OTU_5
Mock.26;size=12938;	otu	87.7	*	OTU_3	OTU_6
Mock.337;size=876;	otu	85.4	*	OTU_6	OTU_7
Mock.182;size=700;	chimera	94.1	100.0	OTU_3(1-149)+OTU_1(150-253)
Mock.1506;size=644;	otu	92.5	*	OTU_7	OTU_8
Mock.1203;size=625;	otu	78.3	*	OTU_2	OTU_9
Mock.706;size=618;	match	97.2	*	OTU_5
Mock.492;size=594;	match	97.2	*	OTU_3
Mock.2004;size=582;	otu	95.3	*	OTU_8	OTU_10
Mock.172;size=579;	chimera	94.1	100.0	OTU_1(1-149)+OTU_3(150-253)
Mock.12;size=570;	match	97.2	*	OTU_6
Mock.1564;size=539;	match	97.2	*	OTU_3
Mock.746;size=400;	chimera	96.4	100.0	OTU_3(1-157)+OTU_5(158-253)
```
* Does this suggest that we should (conservately) omit any OTUs with <1000 sequences?  This is a huge jump from omitting singleton sequences only.
* will try the additional chimera detection using UCHIME with gold reference database, as suggested (here)[http://drive5.com/usearch/manual/uparse_cmds.html]
* another note with the above linked workflow - it suggests re-mapping all reads, including the singleton sequences- to the OTUs, despite that elsewhere it suggests that these reads are errors.
* the "gold" ref db (here)[http://drive5.com/uchime/uchime_download.html] is from 2011, but for our mock community of type strains, it is probably okay.  I would NOT use this ref db for anything other than a mock community of type strains (e.g., not for environmental data) because it is not up to date.  I found a newer "gold" db from (mothur)[http://www.mothur.org/wiki/Silva_reference_files], but the format is for the chimeraslayer algorithm and it is not in the same format as needed for the uchime algorithm (chimeraslayer format includes gaps, misc)
* Generally concerned that not all usearch documentation is updated with each new version - these docs seem to be from usearchv4.

```
module load USEARCH/8.1.1803

# Chimera filtering using reference database
[shadeash@dev-intel14 Shade_MockCom]$ usearch -uchime_ref mock_denoised_otus.fa -db gold.fa -strand plus -nonchimeras mock_denoised_NoChimeraRef_otus.fa
usearch v8.1.1803_i86linux32, 4.0Gb RAM (264Gb total), 20 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:00  40Mb  100.0% Reading mock_denoised_otus.fa
00:00  59Mb  100.0% Reading gold.fa              
00:00  26Mb  100.0% Masking        
00:01  26Mb  100.0% Word stats
00:01  26Mb  100.0% Alloc rows
00:03  83Mb  100.0% Build index

WARNING: Max OMP threads 1

00:08  96Mb  100.0% Search 84/1717 chimeras found (4.9%)
00:08  96Mb  100.0% Writing 1633 non-chimeras           
[shadeash@dev-intel14 Shade_MockCom]$

#output files: mock_denoised_NoChimeraRef_otus.fa
```
* Results:  This detected an additional 84 chimeras, but the rest were not flagged. Generally, I don't think this step will be helpful for the full-community analysis, but it is good to know that the uparse chimera detection finds the majority of the chimeras
* the rest of the OTUs  must be either 1.  OTUs comprised of sequences with LOTs of errors or 2. contaminants.  Will run a quick taxonomic assignment to determine whether contaminants are a problem.
* A note:  it looks like the usearch tools have a (reference-based OTU clustering)[http://www.drive5.com/usearch/manual/cmd_cluster_otus_utax.html] called `-cluster_otus_utax`.  We should consider using this step first as our reference-based otus, and then taking the remaining OTUs through de novo clustering, and then combining both sets for the ultimate database for consistent OTU definitions.  However, there seem to be some disagreement between utax and RDP classifier that we should investigate before committing to using utax...
* I downloaded the 250bp 16S utax db (here)[http://www.drive5.com/usearch/manual/utax_downloads.html]

```
usearch -utax mock_denoised_NoChimeraRef_otus.fa -db rdp_16s_trainset15_250.udb -strand both -taxconfs rdp_16s_short.tc -utaxout tax.txt -rdpout mock_denoised_NoChimeraRef_otus_utaxRDP

---Fatal error---
ReadStdioFile failed, attempted 18398164 bytes, read 14503693 bytes, errno=0
```
* Seem to need more memory than what is available on the development nodes, will have to submit a job to figure this out.
* I made a job called exampled.qsub , but it failed with the same error despite allocating 264 Gb memory. Not sure of the problem, but Will move on for now.  UTAX seems to be in beta version on the usearch website?
* Next:  OTU mapping

```
[shadeash@dev-intel10 Shade_MockCom]$  usearch -usearch_global Mock.fastq -db mock_denoised_NoChimeraRef_otus.fa -strand plus -id 0.97 -uc map.uc -otutabout Mock_OTU_table.txt
usearch v8.1.1803_i86linux32, 4.0Gb RAM (24.6Gb total), 8 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:00  40Mb  100.0% Reading mock_denoised_NoChimeraRef_otus.fa
00:00 6.7Mb  100.0% Masking                                   
00:00 7.6Mb  100.0% Word stats
00:00 7.6Mb  100.0% Alloc rows
00:00 9.1Mb  100.0% Build index

WARNING: Max OMP threads 1

00:20  44Mb  100.0% Searching, 83.3% matched
219757 / 263722 mapped to OTUs (83.3%)      
00:20  44Mb Writing Mock_OTU_table.txt

#output files: Mock_OTU_table.txt, map.uc
```
* here is the head of the otu map, "map.uc":
```
[shadeash@dev-intel10 Shade_MockCom]$ more map.uc
H	0	253	99.2	+	0	0	253M	Mock.1	OTU_1;size=57978;
H	4	253	99.2	+	0	0	253M	Mock.2	OTU_5;size=15010;
H	0	253	99.2	+	0	0	253M	Mock.3	OTU_1;size=57978;
N	*	253	*	.	*	*	*	Mock.4	*
H	2	253	98.8	+	0	0	253M	Mock.5	OTU_3;size=30059;
H	27	253	100.0	+	0	0	253M	Mock.6	OTU_29;size=188;
N	*	253	*	.	*	*	*	Mock.7	*
H	27	253	98.0	+	0	0	253M	Mock.8	OTU_29;size=188;
H	0	253	98.8	+	0	0	253M	Mock.9	OTU_1;size=57978;
H	3	253	99.2	+	0	0	253M	Mock.10	OTU_4;size=14545;
```
* Results:  we match 83% of sequences to OTU definitions. It seems like HITS to the OTU definitions are given an H at the beginning of the line.
* should we be using our denoised dataset instead of our original?

```
[shadeash@dev-intel10 Shade_MockCom]$ usearch -usearch_global denoised.fq -db mock_denoised_NoChimeraRef_otus.fa -strand plus -id 0.97 -uc map_denoised.uc -otutabout Mock_OTU_table_denoised.txt
usearch v8.1.1803_i86linux32, 4.0Gb RAM (24.6Gb total), 8 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:00  40Mb  100.0% Reading mock_denoised_NoChimeraRef_otus.fa
00:00 6.7Mb  100.0% Masking                                   
00:00 7.6Mb  100.0% Word stats
00:00 7.6Mb  100.0% Alloc rows
00:00 9.1Mb  100.0% Build index

WARNING: Max OMP threads 1

00:01  44Mb  100.0% Searching, 49.1% matched
187323 / 207831 mapped to OTUs (90.1%)      
00:01  44Mb Writing Mock_OTU_table_denoised.txt

#output files: Mock_OTU_table_denoised.txt, map_denoised.uc
```
* Here, we decrease our number of reads (because we used the denoised dataset), but we map 90.1% to OTUs (the remaining N's should be the chimeras, which were excluded) from the OTU definitions.  This seems more appropriate - here is the head of the OTU map "map_denoised.uc."

```
H	0	253	100.0	+	0	0	253M	Mock.16;size=56854;	OTU_1;size=57978;
H	1	253	100.0	+	0	0	253M	Mock.25;size=30282;	OTU_2;size=30348;
H	2	253	100.0	+	0	0	253M	Mock.11;size=28182;	OTU_3;size=30059;
H	3	253	100.0	+	0	0	253M	Mock.18;size=14434;	OTU_4;size=14545;
H	4	253	100.0	+	0	0	253M	Mock.37;size=14008;	OTU_5;size=15010;
H	5	253	100.0	+	0	0	253M	Mock.26;size=12938;	OTU_6;size=14217;
H	6	253	100.0	+	0	0	253M	Mock.337;size=876;	OTU_7;size=1005;
```
* Results:  we can see that the top 6 OTUs seem to be the are mock community strains.  Their proportions relative to each other are not as expected (they should be even).
* Wonder if there is a way that we can account for errors to know if, after grouping all the errors from each strain with its parent strain, they would be even.
* In the end, we end up with 1624 OTUs...   

* Okay, back to taxonomy We need to determine if any of these are contaminants.  It seems that the utax algorithm is definitely (in development and not peer-reviewed yet)[http://www.drive5.com/usearch/manual/tax_conf.html].  This makes me worried about using it.  I'll try to submit our sequences to the (RDP Classifier)[http://rdp.cme.msu.edu/classifier/classifier.jsp].  
* Using the recommended 0.5 confidence for short-read sequences
* Found info on command-line RDP Classifier (here)[https://github.com/rdpstaff/classifier]

```
module load RDPClassifier/2.9

java -jar $RDP_JAR_PATH/classifier.jar classify -c 0.5 -o mock_denoised_classified.txt -h mock_hier.txt mock_denoised_NoChimeraRef_otus.fa

#output files: mock_hier.txt, mock_denoised_classified.txt
```
* Results:  Top 6 OTUs are classified as we expect
    * OTU_1;size=57978;		Root	rootrank	1.0	Bacteria	domain	1.0	Firmicutes	phylum	1.0	Bacilli	class	1.0	Bacillales	order	0.98	Bacillaceae 1	family	0.37	Bacillus	genus	0.36

    * OTU_2;size=30348;		Root	rootrank	1.0	Bacteria	domain	1.0	"Bacteroidetes"	phylum	1.0	Flavobacteriia	class	1.0	"Flavobacteriales"	order	1.0	Flavobacteriaceae	family	1.0	Flavobacterium	genus	1.0

    * OTU_3;size=30059;		Root	rootrank	1.0	Bacteria	domain	1.0	"Proteobacteria"	phylum	1.0	Gammaproteobacteria	class	1.0	"Enterobacteriales"	order	1.0	Enterobacteriaceae	family	1.0	Escherichia/Shigella	genus	0.85

    * OTU_4;size=14545;		Root	rootrank	1.0	Bacteria	domain	1.0	"Deinococcus-Thermus"	phylum	1.0	Deinococci	class	1.0	Deinococcales	order	1.0	Deinococcaceae	family	1.0	Deinococcus	genus	1.0

    * OTU_5;size=15010;		Root	rootrank	1.0	Bacteria	domain	1.0	"Proteobacteria"	phylum	1.0	Betaproteobacteria	class	1.0	Burkholderiales	order	1.0	Burkholderiaceae	family	1.0	Burkholderia	genus	1.0

    * OTU_6;size=14217;		Root	rootrank	1.0	Bacteria	domain	1.0	"Proteobacteria"	phylum	1.0	Gammaproteobacteria	class	1.0	Pseudomonadales	order	1.0	Pseudomonadaceae	family	1.0	Pseudomonas	genus	1.0

* What about the rest of the thousands of OTUs?  Generally, they are not  close relatives of these strains. We have 37 different phyla represented in our dataset, and only 4 of those are our mock community strains.

### What to do next?   
1.  We could use the mock community to identify contaminants?   
    - We could use an open-reference approach where we try to hit the "contaminants" database with high confidence (100%), and set-aside and remove any sequences that match the crap database.  We would do this potentially BEFORE matching to a reference database; or we could add this to the end of the OTU picking pipeline
    - we can add a second step of contaminant scrutiny by also using the abundance of that potential contaminant in the mock-community as a way to distinguish "real" contaminants from closely related but real taxa in our samples
    - Should we expect contaminants (e.g., from kit or PCR) to be consistently detected across all samples?  If they are in near-equal abundance, then we have further confidence that they are real contaminants that can be removed.  The paper by (Salter et al. 2014)[http://www.biomedcentral.com/1741-7007/12/87] suggests that even different (identical) DNA extraction kits from the same company can have different contaminants, so perhaps this isn't a good idea after all.  
    -  See (also)[http://www.genomebiology.com/2014/15/12/564]
2.  Eliminate any taxa with less than some sequence abundance cut-off as possible contaminants.
    - For our mock community, taxa with less than <1000 seqs are either really erroneous reads or contaminants. An abudance cut-off of 1000 or less seems really conservative for not-mock samples.
    - Alternatively, we could search for an equivalent "break point" of ranked abundance.  In the mock community, there was a 15-fold decrease in the number of sequences affiliated with real taxa and the number of sequences affiliated with the suspect taxa.
    - should this be considered for each individual sample, or for the dataset as a whole?  It would be more precise to curate each individual sample, but potentially only after considering occurrence patterns across samples to inform spurious results (errors and contaminants).
3.  Generally, the usearch "Denoising" error cloud and dereplicating with chimera-detection worked well to curb (though not eliminate) our OTU inflation problem.  We should test these approaches moving forward.
4.  Denoising/Dealing with the whole dataset - can we inform improved d and w (beyond the default) by our error structure (Q scores) for the whole dataset?
5.  Other thoughts   
    - how to extend usearch to be open reference
    - using the RDP classifier instead of uclust
    - combining the chimera checking with denoising step - could this be done by using the `-cluster_otus` command instead of the `-cluster_fast` one?
```
[shadeash@dev-intel10 Shade_MockCom]$ usearch -cluster_otus uniques_Mock_nosigs.fastq -centroids_fastq denoised_chimera.fq -id 0.9 -maxdiffs 5 -abskew 10 -sizein -sizeout -sort size -otus mock_denoised_chim_otus.fa -relabel OTU_ -uparseout results_chim.txt
usearch v8.1.1803_i86linux32, 4.0Gb RAM (24.6Gb total), 8 cores
(C) Copyright 2013-15 Robert C. Edgar, all rights reserved.
http://drive5.com/usearch

Licensed to: billspat@msu.edu

00:06  54Mb  100.0% 2752 OTUs, 3432 chimeras (18.7%)

WARNING: Option -centroids_fastq ignored


WARNING: Option -sort ignored


WARNING: Option -maxdiffs ignored


WARNING: Option -abskew ignored
```
* Didn't work - will have to cluster OTUs and detect chimeras in separate steps.

### Making a database of craptaminants - contaminants and gross multi-error OTUs remaining after the top 6 type strains are set aside.
* Include all OTUs not among the top 6 - there are 1623 OTUs total, so the remaining 1617 will be the craptaminants db.
* I did this manually by editing (in nano) the mock_denoised_NoChimeraRef_otus.fa to remove the top 6 OTUs' representative sequence (centroid) and leave the remaining ones.  The resulting sequence file is called `mock_craptaminant_OTU_db.fa`.