Copyright 2012 Dmitri Pervouchine (dp@crg.eu), Lab Roderic Guigo
Bioinformatics and Genomics Group @ Centre for Genomic Regulation
Parc de Recerca Biomedica: Dr. Aiguader, 88, 08003 Barcelona

This file is a part of the 'bam2ssj' package.
'bam2ssj' package is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

'bam2ssj' package is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with 'bam2ssj' package.  If not, see <http://www.gnu.org/licenses/>.

============================================================================

DESCRIPTION

bam2ssj is a utility for fast quantification of SJ from BAM files. An
annotation-agnostic version is found at https://github.com/pervouchine/sjcount

============================================================================

INSTALLATION

To install, enter the directory and type:
make all

Prerequisites:
	You need to install samtools

	Get it by svn:
	svn co https://samtools.svn.sourceforge.net/svnroot/samtools/trunk/samtools
	enter the directory and type 'make all'

	Samtools package Version: 0.1.18-dev (r982:313) is compartible (and likely 
	oder versions also are)

(!)	don't forget to update the SAMDIR varibale in the makefile

============================================================================

USAGE: 

./bam2ssj -cps <cps_file> -bam <bam_file> [-out <out_file>] [-log <log_file>] [-maxlen <max_intr_len>] 
	[-minlen <min_intr_len>] [-v suppress verbose] [-read1 0/1] [-read2 0/1] [-g] [-u] [-f]


Input:  (1) sorted BAM file
	(2) CPS (chromosome-position-strand) tab-delimited file sorted by position (chr1 100 + etc)

	In order to get CPS file from gtf, use the utility gtf2cps.sh
	Important: CPS must be sorted by position ONLY!

	If the 4th column contains (a numeric) gene label then only splice junctions within the same gene will be considered (unless the '-g' option is active)
	The utility to generate CPS with gene labels is gtf2cps_with_gene_id.sh (or update the script accordingly if you are using genome other than human)

Options:
	-maxlen <upper limit on intron length>; 0 = no limit (default=0)
	-minlen <lower limit on intron length>; 0 = no limit (default=0)
	-margin <length> minimum number of flanking nucleotides in the read in order to support SJ or cover EB, (default = 4)
	-read1 0/1, reverse complement read1 no/yes (default=1)
	-read2 0/1, reverse complement read2 no/yes (default=0)
	-g ignore gene labels (column 4 of cps), default=OFF
	-u ignore strand (all reads map to the correct strand), default=OFF
	-f count only reads that are flagged 0x800 (uniquely mapped reads), default=OFF

Output:	tab-delimited, sent to stdout by default
	Column 1 is splice_junction_id
	Columns 2-6 are counts of 53, 5X, X3, 50, and 03 reads for the correct (annotated) strand
	Columns 7-11 are similar counts for the incorrect (opposite to annotated) strand

============================================================================

Comment on the notation in bam2ssj output:
	Suppose you have a splice junction between exon boundaries D and A, where D is a donor splice site (5'-SS) and A is an acceptor splice site (3'-SS).

	The number in column 2 (denoted by 53) is the number of reads supporting SJ from D to A.
	The number in column 3 (denoted by 5X) is the number of reads supporting SJ from D to any 
	    other acceptor site A', NOT including A ($\sum\limits_{A'{\ne}A}n(D,A')$)
	The number in column 4 (denoted by X3) is the number of reads supporting SJ from any 
	    other donor site D', NOT including D, to A ($\sum\limits_{D'{\ne}D}n(D',A)$)
	The number in column 5 (denoted by 50) is the number of reads that cover D (i.e., overlap it and go into the intron by at least 1 nt)
	The number in column 6 (denoted by 03) is the number of reads that cover A

	That is,
	psi_5 = column_2/(column_2+column_3)
	psi_3 = column_2/(column_2+column_4)

	theta_5 = (column_2+column_3)/(column_2+column_3+column_5)
	theta_3 = (column_2+column_4)/(column_2+column_4+column_6)

	You might want to impose some thresholding on counts in each of the column in order to get rid of low-confidence values such as psi=1/(1+1)=0.5
	We find it is necessary to require that the denominator in psi_5, psi_3, theta_5, and theta_3, i.e., is greater than 15.

Example:
	To obtain a working example type 'make -f example.mk'
	The script will download all necessary files and execute the pipeline

============================================================================

ALGORITHM

1) Sort genomic coordinates of exon boundaries (the first and the last numcleotides of the exon) by position and thore them in an array, separate for each of ref_id and strand.
   For instance, for 22 human chromosomes there will be 44 such ordered arrays.
2) Each array gets a non-decreasing pointer which is set to the first element of each array initially.
3) Read sorted BAM file sequentially. 
4) For each newcoming read, look up the arrays with the appropriate ref_id.
5) For each strand (+ or -), increment the pointer of the corresponding array until the exon boundary to which it points is to the right of the start of the read (since there are two strands, and each read can be in 'correct' and 'incorrect' orientation with respect to the strand, you have to run it for each of the two strands).
6) Loop up the CIGAR string and find the split coordinates, if there was 'N', or the end of the read, if there were only matches, insertions and deletions. 
7) If there was an 'N', keep looking up the elements of the ordered array from the pointer onward until you run to the right of the rightmost coordinate of the split. You might have finished earlier if you have an upper limit on the intron length or you are restricted to one gene. If you found both ends of the intron among the elements of the array, you take the leftmost one (the donor) and run through a list attached to it looking for the acceptor pair that you find. If you found the pair, increment the counter. If you haven't found the pair, add new element to the list.
8) If there were only matches, insertions and deletions in the read, it means that you got your read mapped to the genome. Starting from the pointer onward until you run to the right end of the read, take all exon boundaried which were on the way and increment their respective counters.

