# Final Project Raw Analysis Juliane Mora 

### The following are steps I used to visualize my data on QIIME2 via HPC command line

## Adding the raw sequence files to Bluewaves
All of the raw sequences were located on my google drive (I have uploaded them to github as well so they are avaiable for others to analyze). I set up a connection between my google drive and Bluewaves, then copied the files from my google drive to a new folder in Bluewaves. There are other ways to do this such as using scp, wget, etc.
```
#copying subset of data from google drive to a new folder
#cp -r means copying recursive (copies all files within a file)
#because of name differences first nine files are before the pipe tenth file is after the pipe
cp -r CP0* ../raw_data_subset | cp -r CP10_L* ../raw_data_subset
```

## Importing data using QIIME2
The sequences I imported are already demultiplexed. This script takes the demultiplexed fastq.qz files and generates a .qza file of the demultiplexed paired end sequences. I ran this script from the folder location containing the fastq.qz files.
```
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path /data/pradalab/jmora/edna_2021/raw_data_subset/subset_fastq \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt \
  --output-path /data/pradalab/jmora/edna_2021/raw_data_subset/subset_fastq/demux-paired-end.qza

#type copied from QIIME2.org
#input-path is where the file is located
#input-format copied from QIIME2.org
#output-path is where the output file is going
```

## Creating a metadata file
A metadata file is used to import important information about each sample into QIIME2. I created the metadata file in google sheets. I downloaded an extension on google sheets called Keemei to verify the format was compatible with QIIME2. Then I downloaded the metadata file to my local drive and saved it as a .tsv.

### Moving metadata file from local drive to github and using wget to import into Bluewaves 
```
git add subsetmetadata.tsv
git commit -m "subset metadata file"
git push main origin
```
```
wget https://github.com/julianemora/bigdata_finalproject.git
```

## Creating a summary of demultiplexed sequences
This step is to create a summary file of the demultiplexed paired readings. It changes the file from a .qza to .qzv so that it can be visualiazed in QIIME2view.
```
#after creating the .qza file from importing, now summarize

qiime demux summarize \
  --i-data demux-paired-end.qza \
  --o-visualization demux-paired-end-summary.qzv

#input is the paired end sequences
#output is the vizualization file
```

## Denoising sequences
All scripts were run in the file location with fastq.qz files and all other .qza and .qzv files

After creating the .qza and .qzv files for demultiplexed paired end sequences, the next step is to denoise the sequences.
```
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux-paired-end.qza \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 35 \
  --p-trunc-len-r 35 \
  --o-table subset-denoise-table.qza \
  --o-representative-sequences subset-rep-seqs.qza \
  --o-denoising-stats subset-denoising-stats.qza

#fragment size for TELE02 is ~400 basepairs
#minimum sequence length observed during subsampling is 35 bases
```
## Creating visualizations after denoising
All scripts were run in the file location with fastq.qz files and all other .qza and .qzv files

After denoising sequences, it is time to generate visualizations.
```
#Feature table summary of denoised sequences

qiime feature-table summarize \
  --i-table subset-denoise-table.qza \
  --o-visualization feature-table-summarize.qzv \
  --m-sample-metadata-file /data/pradalab/jmora/edna_2021/raw_data_subset/subset_fastq/subsetmetadata.tsv

#using the metadata and denoise table you can generate a feature table summary
#input is the denoise table.qza generated from denoising
#output is the visualization of the feature table summary
#m is the metadata file retrieved from github (wget) and path to metadata file
```

```
#visualization of representative sequences after denoising

qiime feature-table tabulate-seqs \
  --i-data subset-rep-seqs.qza \
  --o-visualization subset-representative-seqs.qzv

#input is the representative-sequences.qza file that was generated from denoisi$
#output is the visualization of the .qza file (becomes .qzv)
```
```
#visualization for metadata and denoising stats

qiime metadata tabulate \
  --m-input-file subset-denoising-stats.qza \
  --o-visualization subset-denoising-stats.qzv

#input is the previous denoising-stats.qza file made from denoising paired ends
#output is the denoising-stats.qzv file
```
## Generating visualizations in QIIME2view
QIIME2view is part of QIIME.org and allows for dragging and dropping of .qzv files in order to visualize results. To do this, I had to send files from HPC BLuewaves to a github repository and from there download the files to my local computer. From there I dragged and dropped files into QIIME2view and downloaded the vizualizations as .csv files.
```
git add nameoffile.qzv
git commit -m "QIIME Visualization"
git push main origin
```
## Visualizations and R analysis
Once I dragged and dropped files into QIIME2view, I saved the file datasets to my computer as excel files. These are the codes I used in R to look at the excel files and plot them. I plotted sample ID, frequencies, secquence counts, and locations in different graphs to show various information about the data.
```
descriptive_stats <- read_excel("descriptive_stats.xlsx")
feature_frequency <- read_excel("feature-frequency-detail.xlsx")
persample_fastqcounts <- read_excel("per-sample-fastq-counts.xlsx")
sample_frequency <- read_excel("sample-frequency-detail.xlsx")
sequences <- read_excel("Sequences.xlsx")
metadata <- read_excel("subsetmetadata.xlsx")
sequence_frequency <- merge(feature_frequency, sequences, by = "Feature")

sample_sequencecount_frequency <- merge(sample_frequency, persample_fastqcounts, by.x =c("Sample"),by.y=c("Sample name"))
location_sequencecount_freq <- merge(sample_sequencecount_frequency, metadata, by.x =c("Sample"),by.y=c("sampleid"))
names(location_sequencecount_freq)[names(location_sequencecount_freq) == "Sequence count"] <- "sequencecount"

ggplot(location_sequencecount_freq, mapping= aes (location, Frequency))+geom_point() +
  labs(x="Location",y="Frequency")
#Location and Frequency. Haines point boat launch has the highest frequency, followed by India point park.
#Conimucut point park has the lowest, which is probably due to only one sample from that location being included.

ggplot(location_sequencecount_freq, mapping = aes (location, sequencecount)) + geom_point() +
  labs (x = "Location", y = "Sequence Count")
#Location and Sequence count. Haines point boat lauch has the highest sequence count.
#India point park and Sabin point park are similar. The sample from Conimucut point park also has a high sequence count.

ggplot(location_sequencecount_freq, mapping = aes (Sample, Frequency)) + geom_point() +
  labs (x = "Sample ID", y = "Frequency")
#the sample with the highest frequency is W-19-HP2 which was sampled from Hains point boat launch.
#the samples with the lowest frequency are W-19-IPT1 and W-19-SP1, which were sampled from
#India point park and Sabin point park

ggplot(location_sequencecount_freq, mapping = aes (Sample, sequencecount)) + geom_point() +
  labs (x = "Sample ID", y= "Sequence Count") +
  scale_y_continuous(breaks = c(10000, 20000, 30000, 40000, 50000))
#The samples with the highest sequence count were W-19-CP1 and W-19-HP2. CP1 was sampled from
#Conimucut point park and W-19-HP2 was sampled from Haines point boat launch.
```
