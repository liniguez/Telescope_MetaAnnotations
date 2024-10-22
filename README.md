Telescope Meta-annotations
================
Luis P Iniguez
10/16/2020

Here we provide a meta annotations for
[Telescope](https://github.com/mlbendall/telescope) transposable
elements. The main *annotation* from this repository is:

[Meta\_annotations\_TE](https://github.com/LIniguez/Telescope_MetaAnnotations/raw/main/TE_annotation.v2.0.tsv)

The annotation is based on ENSEMBL HG38 release 99.

``` r
library(ensembldb)
library(tidyr)
library(BSgenome)
library(BSgenome.Hsapiens.UCSC.hg38)
library(Biostrings)
```

``` r
build_ens_DB<-function(){
  system("curl -q ftp://ftp.ensembl.org/pub/release-99/gtf/homo_sapiens/Homo_sapiens.GRCh38.99.gtf.gz --output Homo_sapiens.GRCh38.99.gtf.gz")
  DB<-ensembldb::ensDbFromGtf("Homo_sapiens.GRCh38.99.gtf.gz")
}
ifelse(!file.exists("Homo_sapiens.GRCh38.99.sqlite"),
       yes=build_ens_DB(), 
       no= "Annotations already downloaded")
```

    ## [1] "Annotations already downloaded"

``` r
hg38_ens99<- ensembldb::EnsDb("Homo_sapiens.GRCh38.99.sqlite")
```

Then we extract all the information from the human genome anotations.
(genes, exons, 5’UTR, 3’UTR)

``` r
uscs_style <- mapSeqlevels(seqlevels(genes(hg38_ens99,filter=SeqNameFilter(c(1:22,"X","Y")))), "UCSC")
hg38_genes <- renameSeqlevels(genes(hg38_ens99,filter=SeqNameFilter(c(1:22,"X","Y")), columns=c("gene_name","gene_biotype")), uscs_style)
hg38_exons <- renameSeqlevels(exons(hg38_ens99, columns = c("tx_id", "tx_biotype", "gene_name", "gene_id"),filter =SeqNameFilter(c(1:22,"X","Y"))), uscs_style)

hg38_5utr <- renameSeqlevels(fiveUTRsByTranscript(hg38_ens99, columns = c("tx_id", "tx_biotype", "gene_name", "gene_id"),filter =SeqNameFilter(c(1:22,"X","Y"))), uscs_style)
hg38_3utr <- renameSeqlevels(threeUTRsByTranscript(hg38_ens99, columns = c("tx_id", "tx_biotype", "gene_name", "gene_id"),filter =SeqNameFilter(c(1:22,"X","Y"))), uscs_style)
```

TE annoations are based on Telescope recommended
[annotations](https://github.com/mlbendall/telescope_annotation_db/tree/master/builds):

``` r
build_tr_DB<-function(){
  system("curl -q --output HERV.gtf https://media.githubusercontent.com/media/mlbendall/telescope_annotation_db/master/builds/HERV_rmsk.hg38.v2/transcripts.gtf")
  system("curl -q --output L1.gtf https://media.githubusercontent.com/media/mlbendall/telescope_annotation_db/master/builds/L1Base.hg38.v1/transcripts.gtf")
  system("grep -p '\tgene\t' HERV.gtf | awk '{ split($10,locus,\"\\\"\");
class=\"HERV\"; split(locus[2],family,\"_\"); split($16,category,\"\\\"\");
chrom=$1;start=$4;end=$5;strand=$7;
print locus[2],class,family[1],category[2],chrom,start,end,strand;}' OFS='\t' >Retro_annotation.tsv")
  system(command = "grep -p '\texon\t' L1.gtf | awk '{ split($10,locus,\"\\\"\");
class=\"L1\"; family=\"L1\"; split($16,category,\"\\\"\");
chrom=$1;start=$4;end=$5;strand=$7;
print locus[2],class,family,category[2],chrom,start,end,strand;}' OFS='\t' >>Retro_annotation.tsv")
}
```

``` r
ifelse(!file.exists("Retro_annotation.tsv"),
       yes=build_tr_DB(), 
       no= "Retro Annotations already downloaded")
```

    ## [1] "Retro Annotations already downloaded"

``` r
hg38_retro<- read.table("Retro_annotation.tsv")
colnames(hg38_retro)<-c("Locus","Class","Family","Category","Chrom","Start","End","Strand")
rownames(hg38_retro)<-hg38_retro$Locus

herv_gr<-GRanges(seqnames = hg38_retro$Chrom, ranges = IRanges(start=hg38_retro$Start, end=hg38_retro$End), strand=hg38_retro$Strand)
names(herv_gr)<- hg38_retro$Locus


teseqs<- getSeq(BSgenome.Hsapiens.UCSC.hg38, herv_gr)
names(teseqs)<-paste0(names(herv_gr),"::",as.character(seqnames(herv_gr)),
                      ":",start(herv_gr),"-",end(herv_gr),"(",strand(herv_gr),")")
writeXStringSet(teseqs, "Retro.fa")
```

The coding potential for each of the TE was calculated with
[CNIT](http://cnit.noncode.org/CNIT/)

``` bash
python CNIT.py -f Retro.fa -o RetroCNIT -m 've'
mv RetroCNIT/CNIT.index Coding_Potential_CNIT_TE.txt
```

``` r
cdpot<-read.delim("Coding_Potential_CNIT_TE.txt")
cdpot$ID<-sapply(strsplit(cdpot$Transcript.ID,split = ":"),'[[',1)

hg38_retro$TE_CODING<- NA
hg38_retro[cdpot$ID,"TE_CODING"]<-cdpot$index
```

[Fantom5](https://fantom.gsc.riken.jp/5/) was used for annotating
enhancers and CAGE.

``` r
system("curl -q --output F5.hg38.enhancers.bed.gz https://fantom.gsc.riken.jp/5/datafiles/reprocessed/hg38_latest/extra/enhancer/F5.hg38.enhancers.bed.gz")
system("gunzip F5.hg38.enhancers.bed.gz")
enhancers<-read.delim("F5.hg38.enhancers.bed", header = F)
enhancers<-GRanges(seqnames = enhancers[,1],
                   ranges = IRanges(start=enhancers[,2], end=enhancers[,3]), strand='*')
names(enhancers)<-paste0(seqnames(enhancers),":",start(enhancers),"-",end(enhancers))
```

``` r
system("curl -q --output F5.hg38.CAGE.bed.gz https://fantom.gsc.riken.jp/5/datafiles/reprocessed/hg38_latest/extra/CAGE_peaks/hg38_liftover+new_CAGE_peaks_phase1and2.bed.gz")
system("gunzip F5.hg38.CAGE.bed.gz")
cage<-read.delim("F5.hg38.CAGE.bed", header = F)
cage<-GRanges(seqnames = cage[,1],
                   ranges = IRanges(start=cage[,2], end=cage[,3]), strand=cage[,6])
names(cage)<-paste0(seqnames(cage),":",start(cage),"-",end(cage))
```

The actual annotations:

``` r
pairs_g_h <- findOverlapPairs(hg38_genes, herv_gr, ignore.strand = TRUE)
pairs_e_h <- findOverlapPairs(hg38_exons, herv_gr, ignore.strand = TRUE)

pairs_enh_h <- findOverlapPairs(enhancers, herv_gr, ignore.strand = TRUE)
pairs_cage_h <- findOverlapPairs(cage, herv_gr, ignore.strand = TRUE)

retrosense <- findOverlapPairs(hg38_genes, herv_gr, ignore.strand = FALSE)%>% second() %>% names() %>% unique()
retro5UTR <- findOverlapPairs(hg38_5utr, herv_gr, ignore.strand = TRUE) %>% second() %>% names() %>% unique()
retro3UTR <- findOverlapPairs(hg38_3utr, herv_gr, ignore.strand = TRUE) %>% second() %>% names() %>% unique()
```

Compared to genes:

``` r
tempDF<-data.frame(Gene_name=first(pairs_g_h)$gene_name,
                   Gene_type=first(pairs_g_h)$gene_biotype,
                   Gene_id=first(pairs_g_h)$gene_id,
                   Gene_strand=strand(first(pairs_g_h)),
                   HERV=names(second(pairs_g_h)),
                   HERV_strand=strand(second(pairs_g_h)))

tempDF$Sense<-"SENSE"
tempDF[tempDF$Gene_strand != tempDF$HERV_strand,"Sense"]<-"ANTISENSE"
temp_genename<-aggregate(tempDF[,1], by=list(tempDF[,5]), FUN="paste",collapse = ",")
temp_type<-aggregate(tempDF[,2], by=list(tempDF[,5]), FUN="paste",collapse = ",")
temp_id<-aggregate(tempDF[,3], by=list(tempDF[,5]), FUN="paste",collapse = ",")
temp_sense<-aggregate(tempDF$Sense, by=list(tempDF[,5]), FUN="paste",collapse = ",")

hg38_retro$IntersectedGene<-"None"
hg38_retro[temp_genename[,1],"IntersectedGene"]<-temp_genename[,2]
hg38_retro$IntersectedGeneType<-NA
hg38_retro[temp_type[,1],"IntersectedGeneType"]<-temp_type[,2]
hg38_retro$IntersectedOrientation<-NA
hg38_retro[temp_sense[,1],"IntersectedOrientation"]<-temp_sense[,2]
hg38_retro$IntersectedGeneID<-"None"
hg38_retro[temp_id[,1],"IntersectedGeneID"]<-temp_id[,2]

hg38_retro$TE_type<-"INTERGENIC"
hg38_retro[temp_type[,1],"TE_type"]<-"INTRONIC"
hg38_retro[unique(names(second(pairs_e_h))),"TE_type"]<-"EXONIC"

hg38_retro$TE_5UTR_Intersect<-NA
hg38_retro[unique(names(second(pairs_e_h))),"TE_5UTR_Intersect"]<-"FALSE"
hg38_retro[retro5UTR,"TE_5UTR_Intersect"]<-"TRUE"
hg38_retro$TE_3UTR_Intersect<-NA
hg38_retro[unique(names(second(pairs_e_h))),"TE_3UTR_Intersect"]<-"FALSE"
hg38_retro[retro3UTR,"TE_3UTR_Intersect"]<-"TRUE"

query_pos<-subset(herv_gr[subset(hg38_retro,TE_type=="INTERGENIC")%>% rownames(),],strand=="+")
query_neg<-subset(herv_gr[subset(hg38_retro,TE_type=="INTERGENIC")%>% rownames(),],strand=="-")

upstream_id_pos<-follow(query_pos, hg38_genes,ignore.strand=TRUE)
upstream_id_neg<-precede(query_neg, hg38_genes,ignore.strand=TRUE)
downstream_id_pos<-precede(query_pos, hg38_genes,ignore.strand=TRUE)
downstream_id_neg<-follow(query_neg, hg38_genes,ignore.strand=TRUE)

upstream<-data.frame(herv=names(c(query_pos,query_neg)),
                     herv_strand=strand(c(query_pos,query_neg)),
                     gene_name=NA,
                     gene_id=NA,
                     distance=NA)
logi<-!is.na(c(upstream_id_pos,upstream_id_neg))
upstream[logi,"gene_name"]<-hg38_genes[c(upstream_id_pos,upstream_id_neg)[logi],]$gene_name
upstream[logi,"gene_id"]<-hg38_genes[c(upstream_id_pos,upstream_id_neg)[logi],]$gene_id

dist_upstream_neg<-start(hg38_genes[subset(upstream,herv_strand=="-" & logi)$gene_id,])-end(herv_gr[subset(upstream,herv_strand=="-" & logi)$herv,])
dist_upstream_pos<-start(herv_gr[subset(upstream,herv_strand=="+" & logi)$herv,])-end(hg38_genes[subset(upstream,herv_strand=="+" & logi)$gene_id,])
upstream[upstream$herv_strand=="-" & logi,"distance"]<-dist_upstream_neg
upstream[upstream$herv_strand=="+" & logi,"distance"]<-dist_upstream_pos

downstream<-data.frame(herv=names(c(query_pos,query_neg)),
                       herv_strand=strand(c(query_pos,query_neg)),
                       gene_name=NA,
                       gene_id=NA,
                       distance=NA)
logi<-!is.na(c(downstream_id_pos,downstream_id_neg))
downstream[logi,"gene_name"]<-hg38_genes[c(downstream_id_pos,downstream_id_neg)[logi],]$gene_name
downstream[logi,"gene_id"]<-hg38_genes[c(downstream_id_pos,downstream_id_neg)[logi],]$gene_id

dist_downstream_pos<-start(hg38_genes[subset(downstream,herv_strand=="+" & logi)$gene_id,])-end(herv_gr[subset(downstream,herv_strand=="+" & logi)$herv,])
dist_downstream_neg<-start(herv_gr[subset(downstream,herv_strand=="-" & logi)$herv,])-end(hg38_genes[subset(downstream,herv_strand=="-" & logi)$gene_id,])
downstream[downstream$herv_strand=="-" & logi,"distance"]<-dist_downstream_neg
downstream[downstream$herv_strand=="+" & logi,"distance"]<-dist_downstream_pos


hg38_retro$ClosestUpstream_gn<-NA
hg38_retro[upstream$herv,"ClosestUpstream_gn"]<-upstream$gene_name
hg38_retro$ClosestUpstream_id<-NA
hg38_retro[upstream$herv,"ClosestUpstream_id"]<-upstream$gene_id
hg38_retro$DistUpstream<-NA
hg38_retro[upstream$herv,"DistUpstream"]<-upstream$distance
hg38_retro$ClosestDownstream_gn<-NA
hg38_retro[downstream$herv,"ClosestDownstream_gn"]<-downstream$gene_name
hg38_retro$ClosestDownstream_id<-NA
hg38_retro[downstream$herv,"ClosestDownstream_id"]<-downstream$gene_id
hg38_retro$DistDownstream<-NA
hg38_retro[downstream$herv,"DistDownstream"]<-downstream$distance
```

Compared to fantom5:

``` r
tempDF<-data.frame(Enh_Pos=names(first(pairs_enh_h)),
                   HERV=names(second(pairs_enh_h)))
temp_enhan<-aggregate(tempDF[,1], by=list(tempDF[,2]), FUN="paste",collapse = ",")
hg38_retro$FANTOM_enhancer<-"None"
hg38_retro[temp_enhan[,1],"FANTOM_enhancer"]<-temp_enhan[,2]

tempDF<-data.frame(Enh_Pos=names(first(pairs_cage_h)),
                   HERV=names(second(pairs_cage_h)))
temp_cage<-aggregate(tempDF[,1], by=list(tempDF[,2]), FUN="paste",collapse = ",")
hg38_retro$FANTOM_CAGE<-"None"
hg38_retro[temp_cage[,1],"FANTOM_CAGE"]<-temp_cage[,2]
```

``` r
write.table(hg38_retro, "TE_annotation.v2.0.tsv", row.names = F,sep="\t",quote=F)
```

Additionally to the main annotation file here is another file for
analyzing enrichment of chip-seq data from
[ReMap 2020](http://remap.univ-amu.fr/):

[Telescope and
ReMap2020](https://github.com/LIniguez/Telescope_MetaAnnotations/raw/main/Telescope_overlap_ReMap2020.csv.bz2)

This file was created with the following scripts:

``` bash

wget -O ReMap2020.bed.gz http://remap.univ-amu.fr/storage/remap2020/hg38/MACS2/remap2020_crm_macs2_hg38_v1_0.bed.gz
gunzip ReMap2020.bed.gz
```

``` r
temp<-read.delim("ReMap2020.bed", header = F)
remap<-GRanges(seqnames = temp[,1],
                   ranges = IRanges(start=temp[,2], end=temp[,3]), strand='*',
                ReMap2020=temp[,4])

pairs_remap_h <- findOverlapPairs(remap, herv_gr, ignore.strand = TRUE)

data<-data.frame(ReMap2020=pairs_remap_h@first$ReMap2020,TE =names(pairs_remap_h@second))

db_remap2020<-aggregate(data$ReMap2020, by=list(TE=data$TE), paste0,collapse = ",")
temp<-strsplit(db_remap2020$x,split = ",")
names(temp)<-db_remap2020$TE
db_remap2020_mod<-lapply(names(temp),function(x){
                                      n<-length(temp[[x]])
                                      return(data.frame(remap2020=temp[[x]],
                                                        TE=rep(x,n)))
                                      })
db_remap2020_mod<- do.call("rbind", db_remap2020_mod)
write.table(db_remap2020_mod,"Telescope_overlap_ReMap2020.csv",quote = F,row.names = F,sep=",")
system("bzip2 Telescope_overlap_ReMap2020.csv")
```
