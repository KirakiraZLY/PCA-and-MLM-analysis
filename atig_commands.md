  ## Download data
https://dougspeed.com/1000-genomes-project/
1.   
wget https://www.dropbox.com/s/y6ytfoybz48dc0u/all_phase3.pgen.zst
wget https://www.dropbox.com/s/odlexvo8fummcvt/all_phase3.pvar.zst
wget https://www.dropbox.com/s/6ppo144ikdzery5/phase3_corrected.psam
2.   
D:/plink/plink2.0/plink2 --zst-decompress all_phase3.pgen.zst > all_phase3.pgen
3.   
echo "." > exclude.snps
4. The genotype data will now be stored in binary PLINK format in the files raw.bed, raw.bim and raw.fam. The following commands insert population information and sex into the fam file and replace predictor names with generic names of the form Chr:BP (the latter is not required, but I find this format more convenient). They also save the original nanes.   
5.   
wget https://www.dropbox.com/s/slchsd0uyd4hii8/genetic_map_b37.zip
unzip genetic_map_b37.zip
D:/plink/plink1.9/plink --bfile clean --cm-map genetic_map_b37/genetic_map_chr@_combined_b37.txt --make-bed --out 1000g


awk '(NR==FNR){arr[$1]=$5"_"$6;ars[$1]=$4;next}{$1=$2;$2=arr[$1];$5=ars[$1];print $0}' phase3_corrected.psam raw.fam > clean.fam
awk < raw.bim '{$2=$1":"$4;print $0}' > clean.bim
awk < raw.bim '{print $1":"$4, $2}' > 1000g.names
cp raw.bed clean.bed
## Simulate Phenotype with LDAK
	./ldak5.XXX --make-phenos 1000g_out --bfile 1000g_out --ignore-weights YES --power -1 --her 0.5 --num-phenos 1 --num-causals 1000


# 2022.12.14
## estimate genetic relatedness matrix
	D:/plink/plink1.9/plink --bfile 1000g_out --make-rel triangle
	
## Some QC and PCA based on .bim .bed .fam
	SEE in .rmd

## Calculating kinship-matrix
https://dougspeed.com/calculate-kinships/   

	./ldak5.XXX --thin thin --bfile 1000g_out --window-prune .98 --window-kb 100
awk < thin.in '{print $1, 1}' > weights.thin   
   
   To compute the kinship matrix using the direct method, run
    ./ldak5.XXX --calc-kins-direct LDAK-Thin --bfile 1000g_out --weights weights.thin --power -.25   
	
.bin .detail .adjust .id files contain info about kinship

To instead calculate a kinship matrix assuming the GCTA Model, run   
./ldak.out --calc-kins-direct GCTA --bfile human --ignore-weights YES --power -1

## Calculating kinship matrix by GEMMA
location at /home/zly/Aarhus/ldak5.XXX/1000g   
/home/zly/Aarhus/GEMMA/gemma-0.98.5-linux-static-AMD64/gemma -bfile 1000g_out -gk 2 -o 1000g_out

.sXX.txt records the kinship matrix

Then do the association analysis:   
/home/zly/Aarhus/GEMMA/gemma-0.98.5-linux-static-AMD64/gemma -bfile 1000g_out -k 1000g_out.sXX.txt -lmm 1 -o yield

## To calculate λgc
D:\plink\plink1.9\plink --bfile 1000g_out --assoc --adjust --out 1000g_out_lambdagc

## Bayesian MLM
gemma -bfile 1000g_out -bslmm 1 -n 1 -w 5000000 -s 20000000 -o 1000g_out_bslmm









## Kinship -> REML(Doug)
./ldak5.XXX --reml 1000g_out_reml --pheno 1000g_out.pheno --grm 1000g_out   
By viewing 1000g_out_reml.reml, we see that the estimate of the heritability contributed by the kinship matrix is 0.53 (SD 0.13).

## Kinship -> BLUP
./ldak5.XXX --calc-blups 1000g_out_blup --remlfile 1000g_out_reml.reml --grm 1000g_out --bfile 1000g_out
