# Download SRR data
The accession number of the dataset is GSE138518.

Run `python downloadSRR_order.py` to automatically generate the `ascp` download command in `SRR_Acc_ascp.sh`.

```python
#!/usr/bin/python
# downloadSRR_order.py

InputFile = open("../data/raw/GSE138518_SRR_Acc_List.txt")
OutputFile = open("SRR_Acc_ascp.sh","w")

for line in InputFile:
    line = line.strip()
    print(line)
    print(line[3:6])
    if( len(line) % 9 == 0):

        OutputFile.write('ascp -QT -l 300m -P33001 -i ~/.aspera/connect/etc/asperaweb_id_dsa.openssh era-fasp@fasp.sra.ebi.ac.uk:vol1/srr/SRR' +line[3:6]+'/'+line +' ~/PCOS/data/raw/srr'+'\n')

    else:
        print(line[-1])
        OutputFile.write('ascp -QT -l 300m -P33001 -i ~/.aspera/connect/etc/asperaweb_id_dsa.openssh era-fasp@fasp.sra.ebi.ac.uk:vol1/srr/SRR' +line[3:6]+'/'+'00'+line[-1]+'/'+line +' ~/PCOS/data/raw/srr'+'\n')

InputFile.close()
OutputFile.close()
```

```shell
nohup bash SRR_Acc_ascp.sh &
```

## Transfer SRR to fastq

The fastq files were outputed to the  `~/PCOS/data/raw/srr/fastq` path.

```shell
#!/usr/bin/bash
# Usage: bash sra2fq.sh [sampleID.file]
# first need convert the path to where SRR files are
# then the script will create a `fastq` folder here
# all the converted fastq files will save in `fastq` folder

bak = $IFS
IFS=$'\n'

mkdir fastq
for file in `cat $1` # read the file line by line
do
        echo $file
        fastq-dump --gzip --split-files $file -O ./fastq
done

IFS=$bak
```

```shell
cd ../data/raw/srr
nohup bash ~/PCOS/program/sra2fq.sh ~/PCOS/data/raw/GSE138518_SRR_Acc_List.txt &
```

# Quality Control
The conda environment `rna-seq` were packaged in `rna-seq.yml`.

```shell
cd ~/PCOS/data/raw/srr/fastq/
conda activate rna-seq

nohup fastqc -t 6 -o ~/PCOS/data/raw/srr/fastq/ *.fastq.gz &
multiqc .

firefox multiqc_report.html
```

Remove the adaptors and low quality bases by `fastp`. The clean data were named as `*.clean.fq.gz`.

```shell
nohup cat ~/PCOS/data/raw/GSE138518_SRR_Acc_List.txt | parallel fastp -i ~/PCOS/data/raw/srr/fastq/{}_1.fastq.gz -I ~/PCOS/data/raw/srr/fastq/{}_2.fastq.gz -o ~/PCOS/data/cleanfq/{}_1.clean.fq.gz -O ~/PCOS/data/cleanfq/{}_2.clean.fq.gz -c -q 20 -u 30 -h ~/PCOS/data/cleanfq/{}.clean.html

# 批量查看
nohup fastqc -t 6 -o ~/PCOS/data/cleanfq/ *.clean.fq.gz &
multiqc .

firefox multiqc_report.html
```

# Mapping the reads to human genome using STAR 

Write a script that can pass parameters, just need to input the path of SRR file、index file and gtf file can be run in the trimmed fastp folder, the script will create a STAR folder in the current directory where the fastp folder exists, and all the comparison results will be stored in this folder.

```shell
#!/usr/bin/bash
# star.sh
# Usage: bash star.sh [SRR_list.txt] [STAR_index] [GTF_file]

mkdir ../STAR
echo "The SRR Run Number file is: $1";
echo "The index file is: $2";
echo "The reference file is: $3";

for file in `cat $1`
do
STAR --twopassMode Basic --quantMode TranscriptomeSAM GeneCounts --runThreadN 7 --readFilesCommand zcat --genomeDir $2 --sjdbGTFfile $3 --readFilesIn $file\_1.clean.fq.gz $file\_2.clean.fq.gz --outSAMtype BAM SortedByCoordinate --outFileNamePrefix ../STAR/$file;
done
```

```shell
cd ../data/cleanfq/
nohup bash ~/PCOS/program/star.sh ~/PCOS/data/raw/GSE138518_SRR_Acc_List.txt ~/reference/index/STAR_GRCh37_index150/ ~/annotation/gencode/gencode.v40lift37.annotation.gtf &
```

# Quantify the expression of genes using featureCounts

 
```shell
nohup featureCounts -t gene -f -O -s 2 -p -T 6 -F GTF -a ~/annotation/gencode/gencode.v40lift37.annotation.gtf -o ~/PCOS/data/processed/fcount_gse138518.txt *Aligned.sortedByCoord.out.bam &
```

# Differential alternative splicing events analysis: rMATS
Using rMATS to analyze differential alternative splicing events between two groups, activating previously configured `rmats` environment in conda. The environment was also packaged in the `rmats.yml`  file. 

When analyzing rmats, we should pay attention to the setting of two parameters, one is `--readLength`, the parameter needs to give the length of the reads before trimming. The other is the need to add `--variable-read-length`, so that quantification does not fail due to inconsistent read lengths after trimming.

```shell
conda activate rmats

cat ~/PCOS/data/processed/GSE138518.info.txt | awk -F "\t" '{ORS=","}{if($2=="Normal") print $1"Aligned.sortedByCoord.out.bam"}' | sed 's/.$//' > ~/PCOS/data/processed/normal_path.txt

cat ~/PCOS/data/processed/GSE138518.info.txt | awk -F "\t" '{ORS=","}{if($2=="Polycystic ovary syndrome") print $1"Aligned.sortedByCoord.out.bam"}' | sed 's/.$//' > ~/PCOS/data/processed/pcos_path.txt

# rmats analysis
cd /home/cjh/PCOS/data/STAR/
nohup rmats.py --b1 ~/PCOS/data/processed/pcos_path.txt --b2 ~/PCOS/data/processed/normal_path.txt --gtf ~/annotation/gencode/gencode.v40lift37.annotation.gtf --od ~/PCOS/results/pcos_rmats_out --nthread 2 --readLength 150 --variable-read-length -t paired --tmp ~/PCOS/results/pcos_rmats_tmp &
```

# Quantify the expression of transcripts using RSEM
The results was used to input the isoformswitchanalyzeR package. Use the `*Aligned.toTranscriptome.out.bam` from STAR as the input to the RSEM.

The conda environment `rsem` was packaged in `rsem.yml`.

```shell
conda activate rsem

nohup cat ~/PCOS/data/raw/GSE138518_SRR_Acc_List.txt | while read line; do rsem-calculate-expression --paired-end --alignments -p 5 -no-bam-output -q ${line}Aligned.toTranscriptome.out.bam ~/reference/index/RSEM/gencode.v40lift37/gencode.v40lift37 ~/PCOS/results/rsem_GRCh37_out/${line}; done &
```

