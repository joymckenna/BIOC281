# Tutorial 2
Today we will download a genome fasta and annotation and then build indexes with them for the STAR, cellranger, and kallisto aligners. We will then align cells with STAR and kallisto and compare the resulting counts tables.

### Login to FarmShare
```bash
ssh <SUNetID>@rice.stanford.edu
```

### Switch to the persistent terminal window
The resources we requested on Monday should be ready to go. You can tell if your terminal has switched from something like "(base) <SUNetID>@**rice**<XX>:~$" to "(base) <SUNetID>@**wheat**<XX>:~$"

You should also see the green tmux bar along the bottom of your terminal.
```bash
tmux attach
```
If the resources have been allocated, then proceed. If not, please let us know!

### Activate the single cell environment
```bash
conda activate singlecell
```

### Start jupyter
```bash
jupyter lab --no-browser
```

### Create an SSH tunnel (on your system/laptop) to connect to Jupyter lab running on rice

**On a Windows 10 PC:** Right click on the Windows "Start" icon on lower left and click "Windows PowerShell" to lauch powershell
Alternatively, If you installed SecureCRT then click "File-->Connect Local Shell"

**On a Mac:** Launch Terminal app as above 

On Mac or PC local shell run the following command
```bash
ssh -N -f -L <Port>:localhost:<Port> <SUNetID>@wheat<XX>.stanford.edu
```

### Login to jupyter in your browser, open a Terminal window (File > New... > Terminal), and activate the single cell environment
```bash
conda activate singlecell
```

### Pull the GitHub repository
```bash
cd ~/BIOC281
git pull
```

### Navigate to the current tutorial folder
```bash
cd ~/BIOC281/Classes/2
```

## Build a STAR index with and without masking psuedoautosomal regions (PAR)
The pseudoautosomal regions (PAR1 and PAR2) are short regions of homology between the mammalian X and Y chromosomes. If left unmasked, aligners cannot unamigbuously assign reads to either. When aligning female data the entire Y chromosome is masked, while only the PAR regions, specifically, are masked for male data. See Mangs et al (2007) Current Genomics

### Download NCBI Refseq sequences of human chromosomes 1, 2, X, Y, and M from UCSC
```bash
mkdir -p RefSeq_Oct2020/{Annotation,GenomeFasta,GenomeIndex/STARIndex/{100bp_PRMSK,100bp_YMSK}}

cd RefSeq_Oct2020/GenomeFasta && \
wget --quiet https://hgdownload.soe.ucsc.edu/goldenPath/hg38/chromosomes/chr1.fa.gz && \
wget --quiet https://hgdownload.soe.ucsc.edu/goldenPath/hg38/chromosomes/chr2.fa.gz && \
wget --quiet https://hgdownload.soe.ucsc.edu/goldenPath/hg38/chromosomes/chrX.fa.gz && \
wget --quiet https://hgdownload.soe.ucsc.edu/goldenPath/hg38/chromosomes/chrY.fa.gz && \
wget --quiet https://hgdownload.soe.ucsc.edu/goldenPath/hg38/chromosomes/chrM.fa.gz
```

### Concatenate the sequences to create a minigenome
```bash
cat chr1.fa.gz chr2.fa.gz chrX.fa.gz chrY.fa.gz chrM.fa.gz > hg38.RefSeq.mini.fa.gz
```
### Decompress the minigenome to get genome fasta file
```bash
gzip -d hg38.RefSeq.mini.fa.gz
```

### Download extrinsic spike in sequences (ERCCs)
Many single cell sequencing experiments add ERCC spike-ins during library prep. Download the ERCC fasta and annotation from ThermoFisher
```bash
wget --quiet https://assets.thermofisher.com/TFS-Assets/LSG/manuals/ERCC92.zip
unzip ERCC92.zip
mv ERCC92.gtf ../Annotation/
```

### Append ERCC sequences to the minigenome
```bash
cat hg38.RefSeq.mini.fa ERCC92.fa > hg38.RefSeq.mini.ERCC.fa
```

### Create a hard PAR mask version of genome as per the coordinates
```bash
echo -e "chrY\t10001\t2781479\nchrY\t56887903\t57217415" > PAR1_2.bed
echo -e "chrY\t0\t57227415" > chrY.bed

bedtools maskfasta -fi hg38.RefSeq.mini.ERCC.fa -bed PAR1_2.bed -fo hg38.RefSeq.mini.PARYhMSK.ERCC.fa
bedtools maskfasta -fi hg38.RefSeq.mini.ERCC.fa -bed chrY.bed -fo hg38.RefSeq.mini.YhMSK.ERCC.fa
```

### To explore the newly created genomes efficiently, convert the fasta to 2bit format
```bash
faToTwoBit hg38.RefSeq.mini.PARYhMSK.ERCC.fa hg38.RefSeq.mini.PARYhMSK.ERCC.2bit
faToTwoBit hg38.RefSeq.mini.YhMSK.ERCC.fa hg38.RefSeq.mini.YhMSK.ERCC.2bit
```

### Explore the minigenome and also check that the PAR regions are correctly masked
The coordinates specified to extract 20 bases flanking each side of the PAR sequences

```bash
twoBitToFa hg38.RefSeq.mini.PARYhMSK.ERCC.2bit par1.fa -seq=chrY -start=9981 -end=2781499
twoBitToFa hg38.RefSeq.mini.PARYhMSK.ERCC.2bit par2.fa -seq=chrY -start=56887883 -end=57217435

# The less command allows us to open and view large text files in bash
# You can move through the file using the up and down keys
# You can skip to the end with shift+g and return to the start with g alone
# When you are done exploring, hit q to return to bash
less par1.fa
less par2.fa
```

**Question:** Are there Ns inside the 20bp regions flanking the Ns? If so, why do you think so?

### Check the original minigenome before masking
```bash
faToTwoBit hg38.RefSeq.mini.ERCC.fa hg38.RefSeq.mini.ERCC.2bit
twoBitToFa hg38.RefSeq.mini.ERCC.2bit par1_normal.fa -seq=chrY -start=9981 -end=2781499
twoBitToFa hg38.RefSeq.mini.ERCC.2bit par2_normal.fa -seq=chrY -start=56887883 -end=57217435

less par1_normal.fa
less par2_normal.fa
```

**Question:** After viewing the normal PAR regions, was your hypothesis correct?

### Download the refGene annotation GTF file from UCSC 
```bash
cd ../Annotation/
genePredToGtf hg38 refGene hg38.refGene.gtf
```

### Append ERCC annotations onto the refGene annotation
```bash
cat hg38.refGene.gtf ERCC92.gtf > hg38.refGene.ERCC.gtf
```

### You can use less to check and manually read and explore all the fasta and GTF file
**Note:** less can also read the compresse gzipped (xxx.gz) files as well

```bash
cd ..
less GenomeFasta/chrM.fa.gz
less GenomeFasta/ERCC92.fa
less Annotation/hg38.refGene.ERCC.gtf
```

### Build the STAR genome indices
We need both the genome fasta and the annotation GTF for this purpose. Instead of copying files, we can softlink them to new positions to save time and hard disk space
```bash
cd ./GenomeIndex/STARIndex/100bp_PRMSK/
ln -sv ../../../GenomeFasta/hg38.RefSeq.mini.PARYhMSK.ERCC.fa
ln -sv ../../../Annotation/hg38.refGene.ERCC.gtf
```
#### Create a STARIndex script
We have created text files using echo earlier, now we will learn how we can use "cat" to do the same. This is also the same command we used to concatenate compressed or uncompressed files. You can inspect the script by less as shown above or edit using vim/vi or your favorite text editor.
```bash
cat > ./createPARYMSKSTARIndex_scr << EOF
#!/bin/bash

STAR --runThreadN 4 --runMode genomeGenerate \
--genomeDir ./ --genomeFastaFiles ./hg38.RefSeq.mini.PARYhMSK.ERCC.fa \
--sjdbGTFfile ./hg38.refGene.ERCC.gtf \
--sjdbOverhang 100
EOF

less createPARYMSKSTARIndex_scr
```

#### Now run the bash script to create STAR Index
```bash
bash createPARYMSKSTARIndex_scr
```

#### While index generation is going on (approx ~10 min), open new Terminal window (File > New... > Terminal)
This way you can run multiple terminal sessions to work concurrently

#### From the new session activate "singlecell"

```bash
conda activate singlecell
```

#### Change to the appropriate directory and create script for creating Index, this time for the Complete Y masked minigenome
**Note:** that the annotation remains the same for different versions of the masked genomes
```bash
cd ~/BIOC281/Classes/2/RefSeq_Oct2020/GenomeIndex/STARIndex/100bp_YMSK

ln -sv ../../../GenomeFasta/hg38.RefSeq.mini.YhMSK.ERCC.fa
ln -sv ../../../Annotation/hg38.refGene.ERCC.gtf

cat > ./createYMSKSTARIndex_scr << EOF
STAR --runThreadN 4 --runMode genomeGenerate \
--genomeDir ./ --genomeFastaFiles ./hg38.RefSeq.mini.YhMSK.ERCC.fa \
--sjdbGTFfile ./hg38.refGene.ERCC.gtf \
--sjdbOverhang 100
EOF
```

#### Create the Y masked index
**Important:** Wait for the first index generation in the previous terminal to finish before running your new script. "STAR --runmode genomegenerate" takes 21-25G of RAM even with the minigenome, and we requested only 30G as memory for our use
```bash
bash createYMSKSTARIndex_scr
```

### Move back to your first terminal and prepare an index for 10x's cellranger, which also uses STAR

#### Create directories and softlinks for appropriate gennome fasta and annotation GTF
```bash
cd ~/BIOC281/Classes/2/RefSeq_Oct2020/GenomeIndex/
mkdir 10xIndex && cd 10xIndex
ln -sv ../../GenomeFasta/hg38.RefSeq.mini.PARYhMSK.ERCC.fa
ln -sv ../../Annotation/hg38.refGene.ERCC.gtf
```

#### First clean up annotation GTF to meet cellranger's requirements
```bash
cellranger mkgtf ./hg38.refGene.ERCC.gtf ./hg38.refGene.ERCC.filtered.gtf --attribute=gene_biotype:protein_coding
```

#### Now we use the filtered annotation to prepare 10x reference genome index
```bash
cellranger mkref --genome=hg38.PARYhMSK.ERCC --fasta=hg38.RefSeq.mini.PARYhMSK.ERCC.fa --genes=hg38.refGene.ERCC.filtered.gtf
```
#### You should see the following output indicating how to use reference index while mapping reads generated after a 10x sequencing run
```
>>> Reference successfully created! <<<
You can now specify this reference on the command line:
cellranger --transcriptome=/home/<SUNetID>/BIOC281/Classes/2/RefSeq_Oct2020/GenomeIndex/10xIndex/hg38.PARYhMSK.ERCC ...
```

### Build a kallisto index
Either open a new Terminal window while 10x STAR Index is being built, or move back to a previously open terminal that is idle and proceed

#### Download the whole genome fasta
```bash
cd ~/BIOC281/Classes/2/RefSeq_Oct2020/GenomeFasta
wget --quiet http://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.2bit
twoBitToFa ./hg38.2bit ./hg38.RefSeq.fa
cat ./hg38.RefSeq.fa ./ERCC92.fa > hg38.RefSeq.ERCC.fa
```

#### Build the transcriptome fasta using hg38 genome fasta and annotation
```bash
mkdir -p ~/BIOC281/Classes/2/RefSeq_Oct2020/GenomeIndex/kallistoIndex
cd ~/BIOC281/Classes/2/RefSeq_Oct2020/GenomeIndex/kallistoIndex

ln -sv ../../GenomeFasta/hg38.RefSeq.ERCC.fa
ln -sv ../../Annotation/hg38.refGene.ERCC.gtf

gtf_to_fasta ./hg38.refGene.ERCC.gtf ./hg38.RefSeq.ERCC.fa hg38.refGene.ERCC.transcriptome.fa
```

#### Clean messy transcript names with a perl oneliner
From "https://github.com/trinityrnaseq/Griffithlab_rnaseq_tutorial_wiki/blob/master/Kallisto.md"
```bash
cat hg38.refGene.ERCC.transcriptome.fa | perl -ne 'if ($_ =~/^\>\d+\s+\w+\s+(ERCC\S+)[\+\-]/){print ">$1\n"}elsif($_ =~ /\d+\s+([A-Z]+_\d+_?\d+?)/){print ">$1\n"}else{print $_}' > hg38.refGene.ERCC.transcriptome.clean.fa

less hg38.refGene.ERCC.transcriptome.fa
less hg38.refGene.ERCC.transcriptome.clean.fa
```

#### Now build kallisto transcriptome index
```bash
kallisto index -i hg38.refSeq.transcriptome.ERCC.idx ./hg38.refGene.ERCC.transcriptome.clean.fa --make-unique
```

**Note:** all mappers/aligners of NGS data (fastq) require a step where you create a genome index which speeds up the mapping process. Some mappers like STAR also insert annotation information in the genome index so while mapping the program is aware of known (user provided) splice-junctions-- further speeding up the mapping/alignment process. We encourage you to look up other popular mappers and their genome index generation process like bowtie2, tophat, hisat2, bwa, all of which are genome based mappers, and salmon, which is another transcriptome based mapper similar to kallisto.

### Mapping fastq reads using STAR

#### Download fastq data
We will be downloading paired end reads from two Hematopoietic Stem Cells (HSCs) captured with SmartSeq2 (one male and one female)
```bash
cd ~/BIOC281/Classes/2/
wget -r -np -nH --cut-dirs=1 --reject=index* http://hsc.stanford.edu/resources/fastq/
```

#### Create STAR mapping script. This script is adapted from ENCODE long-mRNA protocol
There is a lot too this script, read through it carefully and try to understand what each section is doing. 
```bash
cat > STARmale_scr << EOF
#!/bin/bash

## Set reference genome, genome indexes, junction and annotation database directory paths
REF=\$HOME/BIOC281/Classes/2/RefSeq_Oct2020/GenomeIndex/STARIndex/100bp_PRMSK
GFA=\$REF/hg38.RefSeq.mini.PARYhMSK.ERCC.fa
ANN=\$HOME/BIOC281/Classes/2/RefSeq_Oct2020/Annotation/hg38.refGene.ERCC.gtf
FTQ=\$HOME/BIOC281/Classes/2/fastq/male

## Set parameters, see https://github.com/alexdobin/STAR/blob/master/doc/STARmanual.pdf
## to see explanation for the paramters used
CmmnPrms="--runThreadN 4 --outSJfilterReads Unique --outFilterType BySJout --outSAMunmapped Within \\
--outSAMattributes NH HI AS nM NM MD jM jI XS MC ch --outFilterMultimapNmax 20 --outFilterMismatchNmax 999 --alignIntronMin 20 \\
--outFilterMismatchNoverReadLmax 0.04 --alignIntronMax 1000000 --alignMatesGapMax 1000000 \\
--alignSJoverhangMin 8 --alignSJDBoverhangMin 1 --sjdbScore 1"
AdtlPrms="--outSAMtype BAM SortedByCoordinate --outBAMcompression 10 --limitBAMsortRAM 57000000000 \\
--quantMode TranscriptomeSAM GeneCounts --quantTranscriptomeBAMcompression 10 --outSAMstrandField intronMotif"

## Define directory structure for run
export OWD=\`pwd\`
export SCR=\$HOME/scratch_male
export STR=\$SCR/STARun
export RDIR=\$SCR/reads

## create scratch run directory and read directory 
mkdir -p \$RDIR

## concatenate read files if they are split into multiple files
cat \$FTQ/*R1*.fastq.gz > \$SCR/male_R1.fastq.gz
cat \$FTQ/*R2*.fastq.gz > \$SCR/male_R2.fastq.gz

## decompress read files
gzip -d \$SCR/*.fastq.gz

## Start skewer to trim reads for quality of base calls and remove adapter sequences at 3' ends
## While STAR employs soft clipping to remove ends of reads that do not align, best practice is to
## NOT leave this to chance and to "hard clip" the sequences before alignment
cd \$RDIR

## If Truseq Illumina adapters (obsolete) were used to prepare the library then use the following adapter sequences
#skewer -x AGATCGGAAGAGCACACGTCTGAACTCCAGTCACNNNNNNATCTCGTATGCCGTCTTCTGCTTG \\
#-y AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGTAGATCTCGGTGGTCGCCGTATCATT \\
#-t 4 -q 21 -l 31 -n -u -o male -f sanger --quiet \$SCR/male_R1.fastq \$SCR/male_R2.fastq

## If Nextera Illumina primers were used to prepare the library then use the following adapter sequences
skewer -x CTGTCTCTTATACACATCTCCGAGCCCACGAGACNNNNNNNNATCTCGTATGCCGTCTTCTGCTTG \\
-y CTGTCTCTTATACACATCTGACGCTGCCGACGANNNNNNNNGTGTAGATCTCGGTGGTCGCCGTATCATT \\
-t 4 -q 21 -l 31 -n -u -o male -f sanger --quiet \$SCR/male_R1.fastq \$SCR/male_R2.fastq

## If custom CZBiohub IDT primers (Index=12bp) were used to prepare the library then use the following adapter sequences
#skewer -x CTGTCTCTTATACACATCTCCGAGCCCACGAGACNNNNNNNNNNNNATCTCGTATGCCGTCTTCTGCTTG \\
#-y CTGTCTCTTATACACATCTGACGCTGCCGACGANNNNNNNNNNNNGTGTAGATCTCGGTGGTCGCCGTATCATT \\
#-t 4 -q 21 -l 31 -n -u -o male -f sanger --quiet \$SCR/male_R1.fastq \$SCR/male_R2.fastq

## Start STAR alignment
mkdir -p \$STR/male_1p
cd \$STR

STAR \$CmmnPrms \$AdtlPrms --genomeDir \$REF --outFileNamePrefix \$STR/male_1p/male.1p. \\
--readFilesIn \$RDIR/male-trimmed-pair1.fastq \$RDIR/male-trimmed-pair2.fastq
 
## Create index for the mapped bam file; this will make it easier to browse the mapped bam file in IGV browser
samtools index \$STR/male_1p/male.1p.Aligned.sortedByCoord.out.bam

## Compress skwer-processed reads to save space.
gzip --best \$RDIR/*

## Compress files that are needed for downstream analysis
find \$STR -type f \( -name "*.out" -o -name "*.tab" -o -name "*.sjdb" -o -name "*.results" \) | xargs gzip -9

## Cleanup and copy back all important files
if [ ! -d \$OWD/reads ];
then 
mkdir -p \$OWD/{reads,STAResults}
fi

## Copy back important files
cp -a \$STR/male_1p \$OWD/STAResults/
cp -a \$RDIR/* \$OWD/reads/

## Remove scratch directory
rm -rf \$SCR
exit 0
EOF
```

#### Start mapping
```bash
bash STARmale_scr
```

#### Move to another idle wheat terminal where "singlecell" environment is already activated and start mapping the reads froms female sample
```bash
cat > STARfemale_scr << EOF
#!/bin/bash

## Set reference genome, genome indexes, junction and annotation database directory paths
REF=\$HOME/BIOC281/Classes/2/RefSeq_Oct2020/GenomeIndex/STARIndex/100bp_YMSK
GFA=\$REF/hg38.RefSeq.mini.PARYhMSK.ERCC.fa
ANN=\$HOME/BIOC281/Classes/2/RefSeq_Oct2020/Annotation/hg38.refGene.ERCC.gtf
FTQ=\$HOME/BIOC281/Classes/2/fastq/female

## Set parameters, see https://github.com/alexdobin/STAR/blob/master/doc/STARmanual.pdf
## to see explanation for the paramters used
CmmnPrms="--runThreadN 4 --outSJfilterReads Unique --outFilterType BySJout --outSAMunmapped Within \\
--outSAMattributes NH HI AS nM NM MD jM jI XS MC ch --outFilterMultimapNmax 20 --outFilterMismatchNmax 999 --alignIntronMin 20 \\
--outFilterMismatchNoverReadLmax 0.04 --alignIntronMax 1000000 --alignMatesGapMax 1000000 \\
--alignSJoverhangMin 8 --alignSJDBoverhangMin 1 --sjdbScore 1"
AdtlPrms="--outSAMtype BAM SortedByCoordinate --outBAMcompression 10 --limitBAMsortRAM 57000000000 \\
--quantMode TranscriptomeSAM GeneCounts --quantTranscriptomeBAMcompression 10 --outSAMstrandField intronMotif"

## Define directory structure for run
export OWD=\`pwd\`
export SCR=\$HOME/scratch_female
export STR=\$SCR/STARun
export RDIR=\$SCR/reads

## create scratch run directory and read directory
mkdir -p \$RDIR

## concatenate read files if they are split into multiple files
cat \$FTQ/*R1*.fastq.gz > \$SCR/female_R1.fastq.gz
cat \$FTQ/*R2*.fastq.gz > \$SCR/female_R2.fastq.gz

## decompress read files
gzip -d \$SCR/*.fastq.gz

## Start skewer to trim reads for quality of base calls and remove adapter sequences at 3' ends
## While STAR employs soft clipping to remove ends of reads that do not align, best practice is to
## NOT leave this to chance and to "hard clip" the sequences before alignmentcd \$RDIR

## If Truseq Illumina adapters (obsolete) were used to prepare the library then use the following adapter sequences
#skewer -x AGATCGGAAGAGCACACGTCTGAACTCCAGTCACNNNNNNATCTCGTATGCCGTCTTCTGCTTG \\
#-y AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGTAGATCTCGGTGGTCGCCGTATCATT \\
#-t 4 -q 21 -l 31 -n -u -o female -f sanger --quiet \$SCR/female_R1.fastq \$SCR/female_R2.fastq

## If Nextera Illumina primers were used to prepare the library then use the following adapter sequences
skewer -x CTGTCTCTTATACACATCTCCGAGCCCACGAGACNNNNNNNNATCTCGTATGCCGTCTTCTGCTTG \\
-y CTGTCTCTTATACACATCTGACGCTGCCGACGANNNNNNNNGTGTAGATCTCGGTGGTCGCCGTATCATT \\
-t 4 -q 21 -l 31 -n -u -o female -f sanger --quiet \$SCR/female_R1.fastq \$SCR/female_R2.fastq

## If custom CZBiohub IDT primers (Index=12bp) were used to prepare the library then use the following adapter sequences
#skewer -x CTGTCTCTTATACACATCTCCGAGCCCACGAGACNNNNNNNNNNNNATCTCGTATGCCGTCTTCTGCTTG \\
#-y CTGTCTCTTATACACATCTGACGCTGCCGACGANNNNNNNNNNNNGTGTAGATCTCGGTGGTCGCCGTATCATT \\
#-t 4 -q 21 -l 31 -n -u -o female -f sanger --quiet \$SCR/female_R1.fastq \$SCR/female_R2.fastq

## Start STAR alignment
mkdir -p \$STR/female_1p
cd \$STR

STAR \$CmmnPrms \$AdtlPrms --genomeDir \$REF --outFileNamePrefix \$STR/female_1p/female.1p. \\
--readFilesIn \$RDIR/female-trimmed-pair1.fastq \$RDIR/female-trimmed-pair2.fastq

## Create index for the mapped bam file; this will make it easier to browse the mapped bam file in IGV browser
samtools index \$STR/female_1p/female.1p.Aligned.sortedByCoord.out.bam

## compress skwer-processed reads to save space.
gzip --best \$RDIR/*

## compress files that are needed for downstream analysis
find \$STR -type f \( -name "*.out" -o -name "*.tab" -o -name "*.sjdb" -o -name "*.results" \) | xargs gzip -9

## Cleanup and Copy back all important files
if [ ! -d \$OWD/reads ];
then
mkdir -p \$OWD/{reads,STAResults}
fi

## Copy back important files
cp -a \$STR/female_1p \$OWD/STAResults/
cp -a \$RDIR/* \$OWD/reads/

## Remove scratch directory
rm -rf \$SCR
exit 0
EOF

#### Start mapping
```bash
bash STARfemale_scr
```

#### Let's now use the reads that were cleaned up using skewer for one of the samples and do trancriptome based mapping using kallisto
Run this in a separate terminal with the single cell environment turned on as soon as you see a reads folder appear in ~/BIOC281/Classes/2/ that contains male-trimmed-pair1.fastq.gz and male-trimmed-pair2.fastq.gz
```bash
cd ~/BIOC281/Classes/2/
mkdir kallistoResults && cd kallistoResults

ln -sv ~/BIOC281/Classes/2/reads/male*fastq.gz ./
ln -sv  ~/BIOC281/Classes/2/RefSeq_Oct2020/GenomeIndex/kallistoIndex/hg38.refSeq.transcriptome.ERCC.idx

## Notice how we can use zcat to pass compressed reads into kallisto
kallisto quant -t 4 -i hg38.refSeq.transcriptome.ERCC.idx -o output -b 100 <(zcat male-trimmed-pair1.fastq.gz) <(zcat male-trimmed-pair2.fastq.gz)
```

### Decompress STAR results
```bash
gzip -d ~/BIOC281/Classes/2/STAResults/male_1p/male.1p.ReadsPerGene.out.tab.gz
```

## We're now going to read in and look at how the different aligners compare, please open BIOC281/Classes/2/Tutorial 2.ipynb
When finished, please save the outputs of Tutorial 2.ipynb (File > Export Notebook As... > Export Notebook to HTML) when you have completed it and upload it to Canvas.

## Request resources for the next class
After saving Tutorial 2.ipynb, return to the original terminal window where you launched jupyter. Quit jupyter by pressing command+c twice.

### Leave the current wheat allocation
After running the command below, your terminal should switch back to rice from wheat.
```bash
exit
```

### Run salloc
You should see the green tmux bar
```bash
salloc --ntasks-per-node=1 --cpus-per-task=4 --mem=30G --time=0-3:00:00 --begin="13:30:00 10/23/20" --qos=interactive srun --pty bash -i -l
```
    
You can now detach from tmux with control+b and then press "d". If the green bar on the bottom disappears, you can safely close your terminal window. See you next class!