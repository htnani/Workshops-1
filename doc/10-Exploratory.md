# Exploratory analysis of cancer data
This workshop will be a mix of different methods to look and explore your data.

We will be working on the same BAMs you generated from the SNV part.
Again, for practical reasons we subsampled the reads from the sample because running the whole dataset would take way too much time and resources.
This leads to some strange results in this part.

This work is licensed under a [Creative Commons Attribution-ShareAlike 3.0 Unported License](http://creativecommons.org/licenses/by-sa/3.0/deed.en_US). This means that you are able to copy, share and modify the work, as long as the result is distributed under the same license.

### Environment setup
We will need an updated bvatools for these exercises

``` {.bash}
cd $HOME/ebiCancerWorkshop201407
wget "https://bitbucket.org/mugqic/bvatools/downloads/bvatools-dev.jar"

export BVATOOLS_JAR=$HOME/ebiCancerWorkshop201407/bvatools-dev.jar
export APP_ROOT=/home/training/Applications/
export PATH=$PATH:$APP_ROOT/bedtools2/bin
export PICARD_HOME=$APP_ROOT/picard-tools-1.115/
export SNPEFF_HOME=$APP_ROOT/snpEff/
export GATK_JAR=$APP_ROOT/gatk/GenomeAnalysisTK.jar
export TRIMMOMATIC_JAR=$APP_ROOT/Trimmomatic-0.32/trimmomatic-0.32.jar
export STRELKA_HOME=$APP_ROOT/strelka-1.0.13/
export REF=/home/training/ebiCancerWorkshop201407/references/

cd $HOME/ebiCancerWorkshop201407

```

### Software requirements
These are all already installed, but here are the original links.

  * [Genome MuSiC](http://gmt.genome.wustl.edu/genome-shipit/genome-music/current/)
  * [BVATools](https://bitbucket.org/mugqic/bvatools/downloads)
  * [SAMTools](http://sourceforge.net/projects/samtools/)
  * [IGV](http://www.broadinstitute.org/software/igv/download)
  * [Genome Analysis Toolkit](http://www.broadinstitute.org/gatk/)
  * [Picard](http://picard.sourceforge.net/)
  * [SnpEff](http://snpeff.sourceforge.net/)

# Exome AI
Calling CNVs in exome is not simple. We have a tool that can help.
Follow the instructions from the [ExomeAI pdf](https://github.com/lletourn/Workshops/blob/ebiCancerWorkshop201407/doc/ExomeAI.pdf), we will try it out.

http://genomequebec.mcgill.ca/exomeai

# Telomeres
In this first step we will try to qualitatively see if the normal and tumor have different
telomere lengths.

One way to do this is find the telomere motif.

A good link to get various telomere repeats is the [Telomerase Database](http://telomerase.asu.edu/sequences_telomere.html)

First step, count the number of reads with these repeats.

``` {.bash}
# Aligned or not, we want them all
samtools view alignment/normal/normal.sorted.bam | awk '{if($10 ~ /TTAGGGTTAGGGTTAGGG/) {SUM++}} END {print "NbTeloReads",SUM}'
samtools view alignment/tumor/tumor.sorted.bam | awk '{if($10 ~ /TTAGGGTTAGGGTTAGGG/) {SUM++}} END {print "NbTeloReads",SUM}'
```

Why did we put multiple copied of the repeat in the search? [Solution](https://github.com/lletourn/Workshops/blob/ebiCancerWorkshop201407/solutions/_telo.ex1.md)  

Next look in the alignments summary file we generated yesterday and extract the number of aligned reads.

``` {.bash}
less -S alignment/normal/normal.sorted.dup.recal.metric.alignment.tsv
less -S alignment/tumor/tumor.sorted.dup.recal.metric.alignment.tsv
```

Now the rest can be done in good old excel.

Open a sheet up, load up your numbers and compute the fold change between normal and tumor.

This case is rather boring, there practically is no change.

# Significantly Mutated Genes
For this part we will try to find significantly mutated genes.
To do this we will use Genome MuSiC

First off there are few things are needed to make MuSiC work

``` {.bash}
mkdir genomeMusic
cd genomeMusic
```

Then:  

1. A region of interest file (ROI)
2. A Mutation Annotation Format file (MAF)
3. A bam list

For the ROI we could use UCSC or BioMart to extract these. We will use Ensembl biomart.
Go to www.ensembl.org/biomart/martview/  

- Choose Database Ensembl Genes 75
- Choose Homo Sapiens Genes
- In filters (click en the left heading), Limit to Genes with RefSeq mRNA ID(s), and choose only chromosome 19 to make the statistical test run faster.
- Under Attributes
    - Choose Structure. Now click in this order (to have a BED file tyoe output)
    - Chromosome Name
    - Exon Start
    - Exon End
    - Associated Gene Name

Download the mart_results.txt file 
put the file in your genomeMusic directory.

Removed the header line (the first line from the file)
Then we need to convert the tab file to a bed.

``` {.bash}
awk '{OFS="\t"} {print $1,$2-3,$3+2,$4}' mart_export.txt | sort -k1V,1V -k2n,2n > exons.roi.bed
```

Why did we substract 3 from the start column? [Solution](https://github.com/lletourn/Workshops/blob/ebiCancerWorkshop201407/solutions/_music.ex1.md)

Merge overlaps to not count twice.

``` {.bash}
bedtools merge -nms -i exons.roi.bed | sed 's/;.*//g' > exons.roi.merged.bed
```

The sed command is to remove GENE;GENE;GENE when bedtools merges.
This is not ideal though when genes overlap. A more complicated way should be used

Now for the bam list

``` {.bash}
echo -e \
"Sample\t$HOME/ebiCancerWorkshop201407/alignment/normal/normal.sorted.dup.recal.bam\t$HOME/ebiCancerWorkshop201407/alignment/tumor/tumor.sorted.dup.recal.bam" \
> bam_list.fofn
```

Let's compute the coverage to see what's usable between samples (we'll come back to this)
```
mkdir musicOutput
gmt music bmr calc-covg --roi-file exons.roi.merged.bed \
  --bam-list bam_list.fofn --output-dir musicOutput/ \
  --reference-sequence $REF/b37.fasta
```

Now let's generate the MAF file

``` {.bash}
java7 -Xmx3G -jar $BVATOOLS_JAR vcf2maf \
  --vcf ../pairedVariants/mpileup.snpeff.vcf \
  --minQual 70 --minCLR 45 \
  --forceTumorName Sample \
  --output ../pairedVariants/mpileup.snpeff.maf

# Usually you'll have one maf per patient so you'll need to merge them
cat ../pairedVariants/*.maf > genomeMusic.maf
```

Now there's a problem here. This version of bvatools requires a mapper which we didn't generate. A new version will come out shortly that fixes this.
In the mean time, modify the MAF  

- Add SPIB to the first entry
- Add KLK1,KLKP1,ETFB,SIGLEC8 to the last 4

Now compute the background rate

``` {.bash}
gmt music bmr calc-bmr --roi-file exons.roi.merged.bed \
  --reference-sequence $REF/b37.fasta --bam-list bam_list.fofn \
  --output-dir musicOutput/ --maf-file genomeMusic.maf
```

And finally compute the significance list

``` {.bash}
gmt music smg --gene-mr-file musicOutput/gene_mrs --output-file musicOutput/smg
```

Now you should have 2 files
smg
smg_details

smg_details has the p-values computed for all genes in your ROI file.
smg only contains the genes for which at least 2 out of the 3 statistical test have an FDR <= max-fdr which is 0.2 by default.

In this case with one subsampled sample, the results don't make much sens.

# Substitution plots
After calling somatic mutations on WGS, we often want to see  

- What is the transition counts vs transversion counts
- Where do these mutations fall, Intergenic, UTR5', CDS, etc

There are a milion ways to do this, we will try one.

First, as we did with MuSiC we need to figure out what parts of the genome are callable accross all the samples.
To do this we will use a GATK tool called CallableLoci

``` {.bash}
for i in normal tumor
do
  java7 -Xmx1G -jar ${GATK_JAR} \
    -T CallableLoci  -R $REF/b37.fasta \
    -o alignment/${i}/${i}.callable.bed \
    --summary alignment/${i}/${i}.callable.summary.txt \
    -I alignment/${i}/${i}.sorted.dup.recal.bam \
    --minDepth 10 --maxDepth 1000 --minDepthForLowMAPQ 10 \
    --minMappingQuality 10 --minBaseQuality 15 \
    -L 19 &
done
wait
```

Now that we have this let's remove/intersect all the samples callable region from the whole genome

``` {.bash}
mkdir substitutions/
cd substitutions/

cat $REF/b37.dict | perl -n -e \
  'if(/^@SQ/){my ($chr,$pos) = /.*SN:([^\t]+)\t.*LN:([0-9]+)\t.*/; print $chr."\t0\t".$pos."\n"}' \
  | grep -v GL | tail -n+2 > wholeGenomeTrack.bed
cp wholeGenomeTrack.bed tmp.bed
for i in ../alignment/*/*.callable.bed
do
  echo $i 
  grep "POOR_MAPPING_QUALITY\|CALLABLE" $i |grep -v "^GL" > sampleTmp.bed
  bedtools intersect -a tmp.bed -b sampleTmp.bed > tmp2.bed
  mv tmp2.bed tmp.bed
done
rm sampleTmp.bed
mv tmp.bed projectCallableRegions.bed
cat projectCallableRegions.bed | awk '{SUM+=$3-$2} END {print SUM}'
```

What are the main causes of lost of callability? [Solution](https://github.com/lletourn/Workshops/blob/ebiCancerWorkshop201407/solutions/_subst.ex1.md)

Now let's get the rest of the tracks per region.
We will extract them from snpEffs database

``` {.bash}
# Remove track header
java7 -Xmx1G -jar $BVATOOLS_JAR extractSnpEffTracks -c $SNPEFF_HOME/snpEff.config hg19 | sed '/^track/d' > snpEffRegions.bed

awk '{print > "snpEff.hg19."_$4_".bed"}' snpEffRegions.bed
```

Now we have one bed track per region. But we are missing one, do you know which? [Solution)[https://github.com/lletourn/Workshops/blob/ebiCancerWorkshop201407/solutions/_subst.ex2.md)

``` {.bash}
cp wholeGenomeTrack.bed tmp.bed
for i in snpEff.hg19.[UCD]*.bed snpEff.hg19.INTRON.bed
do
  bedtools subtract -a tmp.bed -b $i > tmp2.bed
  mv tmp2.bed tmp.bed
 done
mv tmp.bed snpEff.hg19.INTERGENIC.bed
```

Let's check the size of each region from our project

``` {.bash}
for i in snpEff.hg19.*.bed
do
  echo $i
  ORG=`cat $i | awk '{SUM+=$3-$2} END {print SUM}'`
  bedtools intersect -a $i -b projectCallableRegions.bed > callable.${i%.bed}.bed
  CALL=`cat callable.${i%.bed}.bed | awk '{SUM+=$3-$2} END {print SUM}'`
  echo $ORG
  echo $CALL
  echo "$(($CALL*100/$ORG))%"
done
```

Again, here since we down sampled the representation is pretty bad. But in normal WGS you might still find that some regions, like UTR5' are quite lower than expected.
This is were kits like the no PCR help quite a bit.

Now the good part, let's extract the counts

``` {.bash}
java7 -Xmx5G -jar ~/bvatools-dev.jar mutationrates --vcf ../pairedVariants/mpileup.vcf \
  --bed Callable:projectCallableRegions.bed \
  --bed CDS:snpEff.hg19.CDS.bed \
  --bed DOWNSTREAMcallable.:snpEff.hg19.DOWNSTREAM.bed \
  --bed INTRON:callable.snpEff.hg19.INTRON.bed \
  --bed UPSTREAM:callable.snpEff.hg19.UPSTREAM.bed \
  --bed UTR5:callable.snpEff.hg19.UTR5.bed \
  --bed INTERGENIC:callable.snpEff.hg19.INTERGENIC.bed \
  --bed UTR3:callable.snpEff.hg19.UTR3.bed \
  --minQual 70 --minCLR 45 --threads 1 \
  --exclude MT,GL000207.1,GL000226.1,GL000229.1,GL000231.1,GL000210.1,GL000239.1,GL000235.1,GL000201.1,GL000247.1,GL000245.1,GL000197.1,GL000203.1,GL000246.1,GL000249.1,GL000196.1,GL000248.1,GL000244.1,GL000238.1,GL000202.1,GL000234.1,GL000232.1,GL000206.1,GL000240.1,GL000236.1,GL000241.1,GL000243.1,GL000242.1,GL000230.1,GL000237.1,GL000233.1,GL000204.1,GL000198.1,GL000208.1,GL000191.1,GL000227.1,GL000228.1,GL000214.1,GL000221.1,GL000209.1,GL000218.1,GL000220.1,GL000213.1,GL000211.1,GL000199.1,GL000217.1,GL000216.1,GL000215.1,GL000205.1,GL000219.1,GL000224.1,GL000223.1,GL000195.1,GL000212.1,GL000222.1,GL000200.1,GL000193.1,GL000194.1,GL000225.1,GL000192.1 \
  --output all.hg19.callable.tsv
```

Now open the file in excel, and plot the Base substitution percent histogram.

# Finding contamination
This is more to QC but it can be very helpful to find strange patterns in your samples.

Extract positions of somatic variants

``` {.bash}
grep -v INDEL pairedVariants/mpileup.vcf \
 | perl -ne 'my @values=split("\t"); my ($clr) = $values[7] =~ /CLR=(\d+)/; if(defined($clr) && $clr >= 45 && $values[5] >= 70) {print "$values[0]\t$values[1]\t$values[3]\t$values[4]\n"}' \
 > pairedVariants/mpileup.snpPos.tsv
```

Now we have our positions, we need the read counts *per lane* for these positions.
BVATools does this

``` {.bash}
for i in normal tumor
do
  java7 -Xmx2G -jar $BVATOOLS_JAR basefreq \
    --pos pairedVariants/mpileup.snpPos.tsv \
    --bam alignment/${i}/${i}.sorted.dup.recal.bam \
    --out alignment/${i}/${i}.somaticSnpPos \
    --perRG
done
```

We can look at one of the files to see what basefreq extracted

``` {.bash}
less -S alignment/normal/normal.somaticSnpPos.normal_C0LWRACXX_1.alleleFreq.csv
```

Now we need to extract and format the data so we can create a PCA and some hierarchical clusters

``` {.bash}
# Generate a part of the command
for i in alignment/*/*.somaticSnpPos*_?.alleleFreq.csv
do
  NAME=`echo $i | sed 's/.*somaticSnpPos.\(.*\).alleleFreq.csv/\1/g'`
  echo "--freq $NAME $i";done | tr '\n' ' '
done

# Copy this output and paste it at the end of the command like so
java7 -Xmx2G -jar ~/bvatools-dev.jar clustfreq \
--snppos pairedVariants/mpileup.snpPos.tsv \
--threads 3 \
--prefix sampleComparison \
--outputFreq \
--freq normal_C0LWRACXX_1 alignment/normal/normal.somaticSnpPos.normal_C0LWRACXX_1.alleleFreq.csv \
--freq normal_C0LWRACXX_6 alignment/normal/normal.somaticSnpPos.normal_C0LWRACXX_6.alleleFreq.csv \
--freq normal_C0PTAACXX_6 alignment/normal/normal.somaticSnpPos.normal_C0PTAACXX_6.alleleFreq.csv \
--freq normal_C0PTAACXX_7 alignment/normal/normal.somaticSnpPos.normal_C0PTAACXX_7.alleleFreq.csv \
--freq normal_C0PTAACXX_8 alignment/normal/normal.somaticSnpPos.normal_C0PTAACXX_8.alleleFreq.csv \
--freq normal_C0R2BACXX_6 alignment/normal/normal.somaticSnpPos.normal_C0R2BACXX_6.alleleFreq.csv \
--freq normal_C0R2BACXX_7 alignment/normal/normal.somaticSnpPos.normal_C0R2BACXX_7.alleleFreq.csv \
--freq normal_C0R2BACXX_8 alignment/normal/normal.somaticSnpPos.normal_C0R2BACXX_8.alleleFreq.csv \
--freq normal_D0YR4ACXX_1 alignment/normal/normal.somaticSnpPos.normal_D0YR4ACXX_1.alleleFreq.csv \
--freq normal_D0YR4ACXX_2 alignment/normal/normal.somaticSnpPos.normal_D0YR4ACXX_2.alleleFreq.csv \
--freq tumor_BC0TV0ACXX_8 alignment/tumor/tumor.somaticSnpPos.tumor_BC0TV0ACXX_8.alleleFreq.csv \
--freq tumor_C0LVJACXX_6 alignment/tumor/tumor.somaticSnpPos.tumor_C0LVJACXX_6.alleleFreq.csv \
--freq tumor_C0PK4ACXX_7 alignment/tumor/tumor.somaticSnpPos.tumor_C0PK4ACXX_7.alleleFreq.csv \
--freq tumor_C0PK4ACXX_8 alignment/tumor/tumor.somaticSnpPos.tumor_C0PK4ACXX_8.alleleFreq.csv \
--freq tumor_C0R29ACXX_7 alignment/tumor/tumor.somaticSnpPos.tumor_C0R29ACXX_7.alleleFreq.csv \
--freq tumor_C0R29ACXX_8 alignment/tumor/tumor.somaticSnpPos.tumor_C0R29ACXX_8.alleleFreq.csv \
--freq tumor_C0TTBACXX_3 alignment/tumor/tumor.somaticSnpPos.tumor_C0TTBACXX_3.alleleFreq.csv \
--freq tumor_D114WACXX_8 alignment/tumor/tumor.somaticSnpPos.tumor_D114WACXX_8.alleleFreq.csv
```

Now you should have 2 files
sampleComparison.freq.csv
sampleComparison.dist.csv

One contains vectors of snp frequences, the other contains the pairwise Euclidean distance
Now plot the result in R
``` {.r}
fileName <- "sampleComparison"
normalName <- "normal"


distFile <- paste(fileName, ".dist.csv",sep="")

#distName.noext = sub("[.][^.]*$", "", distName, perl=TRUE)

dataMatrix <- read.csv(distFile, row.names=1, header=TRUE)
hc <- hclust(as.dist(dataMatrix));
hcd = as.dendrogram(hc)

colLab <- function(n) {
    if (is.leaf(n)) {
        a <- attributes(n)
        labCol = c("blue");
        if(grepl(normalName, a$label)) {
          labCol = c("red");
        }
        attr(n, "nodePar") <- c(a$nodePar, lab.col = labCol)
    }
    n
}

# using dendrapply
clusDendro = dendrapply(hcd, colLab)
cols <- c("red","blue")


freqFile <- paste(fileName, ".freq.csv",sep="")
data <- read.csv(freqFile, header=FALSE,row.names=1, colClasses=c("character", rep("numeric",4))
colLanes <- rownames(data)
colLanes[grep(normalName, colLanes, invert=TRUE)] <- "blue"
colLanes[grep(normalName, colLanes)] <- "red"
pca <- prcomp(data)

# make plot
pdfFile <- paste(fileName,".laneMix.pdf", sep="")
pdfFile
pdf(pdfFile)
par(mar=c(3,3,1,12))
plot(clusDendro, main = "Lane distances", horiz=TRUE)
legend("top", legend = c("Normal","Tumor"), fill = cols, border = cols)

par(mar=c(1,1,1,1))
plot(pca$x[,1:2])
text(pca$x[,1:2], rownames(data), col=colLanes)
screeplot(pca, type="lines")
plot(pca$x[,2:3])
text(pca$x[,2:3], rownames(data), col=colLanes)
dev.off()

pngFile <- paste(fileName,".laneMix.png", sep="")
pngFile
png(pngFile, type="cairo", width=1920, height=1080)
par(mar=c(3,3,1,12))
par(mfrow=c(2,1))
plot(clusDendro, main = "Lane distances", horiz=TRUE)
legend("top", legend = c("Normal","Tumor"), fill = cols, border = cols)

plot(pca$x[,1:2])
text(pca$x[,1:2], rownames(data), col=colLanes)
dev.off()

```

Look at the graphs.

You could do this directly in R but
1- The basefreq format is not simple to parse
2- When you have thousands of somatics, and/or hundreds of samples, R struggles to build de pairwise distance and the PCA. This is why we precompute it in java before.


# Plot Depth Ratio and BAF
For this step we will need a bigger dataset.

Get the data from penelope (shared drive)
Copy these files:

```
sampleFreq.tsv
chr3.ref.tsv
```
in your work directory or

```
cp /media/sf_shared/Cancer\ Genomics/Louis\ Letourneau/*.tsv $HOME/ebiCancerWorkshop201407/
```

Then we will use BVATools to generate the graphs
```
java7 -Xmx5G -jar $BVATOOLS_JAR ratiobaf --snppos chr3.ref.tsv --basefreq sampleFreq.tsv --refdict references/b37.dict --plot --prefix sampleDepthRatioBaf
```

And now look at the results

```
eog *.png
```

What do you see?

## Aknowledgments
The format for this tutorial has been inspired from Mar Gonzalez Porta of Embl-EBI, who I would like to thank and acknowledge. I also want to acknowledge Mathieu Bourgey, Francois Lefebvre, Maxime Caron and Guillaume Bourque for the help in building these pipelines and working with all the various datasets.
