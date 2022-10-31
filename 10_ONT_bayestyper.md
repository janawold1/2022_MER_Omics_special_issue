# Genotyping SVs called using long-read data with BayesTyper 
BayesTyper is a powerful baysian based genotyping program. However, one draw back is that it is computationally intensive and has multiple intermediate steps to navigate. For more details on troubleshooting recommendations, please see [08_manta_bayestyper.md](https://github.com/janawold1/2022_MER_Omics_special_issue/blob/main/08_manta_bayestyper.md).

To begin, global variables were defined as below:
```
deepV=DEEPVARIANT:/kakapo-data/bayestyper/Trained_SV_scaffolds_renamed.sorted.vcf
out=/kakapo-data/bayestyper/
cute=/kakapo-data/bayestyper/cuteSV/
sniff=/kakapo-data/bayestyper/sniffles/q
ref=/kakapo-data/References/kakapo_autosomes.fa
trio=/kakapo-data/metadata/sample_trios.csv
```
See [09_ONT_SV_discovery.md](https://github.com/janawold1/2022_MER_Omics_special_issue/blob/main/09_ONT_SV_discovery.md) for more details on how SV calls using CuteSV and Sniffles were filtered prior to genotyping.

BayesTyper works best when SNPs are included. For this, the DeepVariant SNP call set available under the kākāpō125+ Sequencing Consortium were used. As outlined in [09_ONT_SV_discovery.md](https://github.com/janawold1/2022_MER_Omics_special_issue/blob/main/09_ONT_SV_discovery.md) all chromosome sizes were used to match kākāpō125+ and NCBI chromosome IDs.

## Running KMC and makeBloom
The third party program, KMC, and bayesTyperTools makeBloom are necessary to run BayesTyper cluster and BayesTyper genotype. It is independent of preparing the variant VCFs and can be done concurrrently.

It is really easy to over-resource KMC, that is to say if you provide it with too
much memory/threads the program freezes. I have had the best luck with 24Gb RAM and 48 threads as per below. It also found that I needed to run KMC one individual at a time. I had a lot of isls when I attempted to run multiple individuals at once since KMC uses the same temp file naming convention each time it runs. Basically temp files from different runs were overwriting each other and I needed to designate different locations for the temp files.
```
ulimit -n 2048
for mkmc in ${male}batch{01..14}/*_nodup.bam
    do
    indiv=$(basename ${mkmc} _nodup.bam)
    echo "Running KMC for ${indiv}..."
    kmc -k55 -ci1 -m24 -t48 -fbam ${mkmc} ${out}kmc/${indiv}_KMC.res ${out}kmc_tmp/
    echo "Running makeBloom for ${indiv}..."
    bayesTyperTools makeBloom -k ${out}kmc/${indiv}_KMC.res -p 8
done
for fkmc in ${female}batch{01..11}/*_nodup.bam
    do
    indiv=$(basename ${fkmc} _nodup.bam)
    echo "Running KMC for ${indiv}..."
    kmc -k55 -ci1 -m24 -t48 -fbam ${fkmc} ${out}kmc/${indiv}_KMC.res ${out}kmc_tmp/
    echo "Running makeBloom for ${indiv}..."
    bayesTyperTools makeBloom -k ${out}kmc/${indiv}_KMC.res -p 8
done
```

## Convert VCFs to remove symbolic alleles
Unlike the short-read SV call set generated by Manta, the long-read call sets already included sequence information and were not only represented by symbolic alleles. Therefore, the conversion from symbolic alleles to sequences was not necessary.

To combine the VCFs generated by CuteSV and Sniffles with the SNPs called in the kākāpō125+ data set, we first normalised the SV calls.
```
bcftools norm --threads 24 -f ${ref} -m -any \
    -O z -o ${cute}02_bayestyper_candidates_norm.vcf.gz \
    ${cute}01_bayestyper_candidates.vcf

bcftools norm --threads 24 -f ${ref} -m -any \
    -O z -o ${sniff}02_bayestyper_candidates_norm.vcf.gz \
    ${sniff}01_bayestyper_candidates.vcf
```
 The SV calls and SNP calls were then merged as per:
```
bayesTyperTools combine \
    -v ${deepV},CuteSV:${cute}bayestyper_candidates_norm.vcf.gz \
    -o ${cute}03)combined_variants -z

bayesTyperTools combine \
    -v ${deepV},CuteSV:${sniff}bayestyper_candidates_norm.vcf.gz \
    -o ${sniff}03)combined_variants -z
```
However, not all SVs were included once this `bayesTyperTools combine` step was complete. To find which SVs were dropped from the `03_combined_variants` we first extracted the sites associated with either CuteSV or Sniffles.
```
bcftools view --threads 24 -i 'ACO="CuteSV"' \
    -O z -o ${cute}combined_cuteSV_variants.vcf.gz \
    ${cute}03_combined_variants.vcf.gz
bcftools view --threads 24 -i 'ACO="SNIFFLES"' \
    -O z -o ${sniff}combined_sniffles_variants.vcf.gz \
    ${sniff}03_combined_variants.vcf.gz
```
This indicated that 2,059 out of 2,238 CuteSV calls were included in the `03_combined_variants.vcf.gz` file and 5,030 out of 5,229 Sniffles calls were included for genotyping.

To find the type and location of the 'missing' SVs, we used `bcftools isec`.
```
bcftools isec \
    ${cute}02_bayestyper_candidates_norm.vcf.gz \
    ${cute}combined_cuteSV_variants.vcf.gz \
    -p ${cute}

bcftools isec \
    ${sniff}02_bayestyper_candidates_norm.vcf.gz \
    ${sniff}combined_sniffles_variants.vcf.gz \
    -p ${sniff}
```
It is notable that although 179 variants in the `03_combined_variants.vcf.gz` file for CuteSV and 199 of Sniffles varints appeared to be excluded following `bayesTyperTools combine`. However, more SVs failed to intersect with the normalised candidate file `02_bayestyper_candidates_norm.vcf.gz` for both SV discovery tools. A summary of these SVs is below.

|   SV type   | CuteSV | Sniffles |
|:-----------:|:------:|:--------:|
|   Deletion  |   66   |    87    |
| Duplication |  138   |    61    |
|  Insertion  |   26   |   179    |
|  Inversion  |   12   |    95    |
|    Total    |  242   |   422    |

Closely examining the number of SVs unique to `03_combined_variants.vcf.gz` confirmed that 63 CuteSV calls and 223 Sniffles calls had their ending location shifted during `bayesTyperTools combine`.

These were found and and summarised as per below.
```
bcftools query -f '%CHROM\t%POS\n' ${cute}0001.vcf > ${cute}missing.txt
bcftools query -f '%CHROM\t%POS\n' ${sniff}0001.vcf > ${sniff}missing.txt

bcftools query -R ${cute}missing.txt \
    -f '%SVTYPE\n' \
    ${cute}02_bayestyper_candidates_norm.vcf.gz | sort | uniq -c
bcftools query -R ${cute}missing.txt \
    -f '%SVTYPE\n' \
    ${cute}02_bayestyper_candidates_norm.vcf.gz | sort | uniq -c
```
The counts of shifted SVs for CuteSV and Sniffles are:
|   SV type   | CuteSV | Sniffles |
|:-----------:|:------:|:--------:|
|   Deletion  |   47   |   103    |
| Duplication |   24   |     6    |
|  Insertion  |   26   |   162    |
|  Inversion  |    0   |     0    |
|    Total    |   97   |   271    |

These details were used to resolve SV types after genotyping.

## Genotyping Variants
As noted previously, all unplaced scaffolds where ploidy was unknown were already excluded from candidate VCFs, BAM files and the reference. Note, no more than 30 indivs can be genotyped at once. The <samples>.tsv file should contain one sample per row with columns:
<sample_id>, <sex> and <kmc_output_prefix> and no header.

```
for samps in ${out}sample_batches/sample_batch*.tsv
    do
    base=$(basename ${samps} .tsv)
    bayesTyper cluster \
        --variant-file ${cute}combined_variants.vcf.gz \
        --samples-file ${samps} --genome-file ${ref} \
        --output-prefix ${cute}clusters/${base}/ --threads 24
    bayesTyper cluster \
        --variant-file ${sniff}combined_variants.vcf.gz \
        --samples-file ${samps} --genome-file ${ref} \
        --output-prefix ${sniff}clusters/${base}/ --threads 24
    bayesTyper genotype \
        --variant-clusters-file ${cute}clusters/${base}_unit_1/variant_clusters.bin \
        --cluster-data-dir ${cute}clusters/${base}_cluster_data/ \
        --samples-file ${samps} --genome-file ${ref} \
        --output-prefix ${cute}genotypes/${base}_genotypes \
        --threads 24 --chromosome-ploidy-file ${out}ploidy.tsv
    bayesTyper genotype \
        --variant-clusters-file ${sniff}clusters/${base}_unit_1/variant_clusters.bin \
        --cluster-data-dir ${sniff}clusters/${base}_cluster_data/ \
        --samples-file ${samps} --genome-file ${ref} \
        --output-prefix ${sniff}genotypes/${base}_genotypes \
        --threads 24 --chromosome-ploidy-file ${out}ploidy.tsv
done
```
## Merging SV genotype batches
To merge genotyping batches, the outputs from Bayestyper were first bgzipped and indexed with tabix.
```
for vcf in ${cute}genotypes/sample_batch{1..6}/*_genotypes.vcf
    do
    batch=$(basename $vcf _genotypes.vcf)
    bgzip ${vcf}
    echo "Now indexing $vcf"
    tabix ${cute}genotypes/${batch}/${batch}_genotypes.vcf.gz
done &
for vcf in ${sniff}genotypes/sample_batch{1..6}/*_genotypes.vcf
    do
    batch=$(basename $vcf _genotypes.vcf)
    bgzip ${vcf}
    echo "Now indexing $vcf"
    tabix ${sniff}genotypes/${batch}/${batch}_genotypes.vcf.gz
done
wait
```
Then merged according to sample ID with bcftools.
```
bcftools merge -m id -O z -o ${cute}genotypes/04_raw_cuteSV_DeepV_genotypes.vcf.gz \
    ${cute}genotypes/sample_batch1/sample_batch1_genotypes.vcf.gz \
    ${cute}genotypes/sample_batch2/sample_batch2_genotypes.vcf.gz \
    ${cute}genotypes/sample_batch3/sample_batch3_genotypes.vcf.gz \
    ${cute}genotypes/sample_batch4/sample_batch4_genotypes.vcf.gz \
    ${cute}genotypes/sample_batch5/sample_batch5_genotypes.vcf.gz \
    ${cute}genotypes/sample_batch6/sample_batch6_genotypes.vcf.gz

bcftools merge -m id -O z -o ${sniff}genotypes/04_raw_sniffles_DeepV_genotypes.vcf.gz \
    ${sniff}genotypes/sample_batch1/sample_batch1_genotypes.vcf.gz \
    ${sniff}genotypes/sample_batch2/sample_batch2_genotypes.vcf.gz \
    ${sniff}genotypes/sample_batch3/sample_batch3_genotypes.vcf.gz \
    ${sniff}genotypes/sample_batch4/sample_batch4_genotypes.vcf.gz \
    ${sniff}genotypes/sample_batch5/sample_batch5_genotypes.vcf.gz \
    ${sniff}genotypes/sample_batch6/sample_batch6_genotypes.vcf.gz
```
This raw genotype file contains genotypes for both the SVs called by either CuteSV or Sniffles and DeepVariant SNPs. However, the SV class is not included in this output. To interpret the genotype results from BayesTyper, we have to first extract the SVs from this file for annotation. SVs were simultaneously filtered for missingness and to ensure variant sites were included.
```
 bcftools view --threads 24 \
    -i '(ACO="CuteSV") & ((N_PASS(GT=="mis") < 17) & (N_PASS(GT="alt")>=1))' \
    -O v -o ${cute}05_cuteSV_SV_genotypes.vcf \
    ${cute}04_raw_cuteSV_DeepV_genotypes.vcf.gz

 bcftools view --threads 24 \
    -i '(ACO="SNIFFLES") & ((N_PASS(GT=="mis") < 17) & (N_PASS(GT="alt")>=1))' \
    -O v -o ${sniff}05_sniffles_SV_genotypes.vcf \
    ${sniff}04_raw_sniffles_DeepV_genotypes.vcf.gz
```
We are now ready to annotate the VCF with SV types.

## Finding SV type
As mentioned above, BayesTyper removes symbolic alleles from the genotype files. To make comparisons among SV tools similar, the SV type called by either CuteSV or Sniffles were resoved as per below.
1) First identify the locations of genotyped variants:
```
bcftools query -f '%CHROM\t%POS\n' ${out}bayestyper/batch_filtered/08_batch_filtered_trios.vcf > ${out}bayestyper/batch_genos
bcftools query -f '%CHROM\t%POS\n' ${out}bayestyper/joint_filtered/08_joint_filtered_trios.vcf > ${out}bayestyper/joint_genos
```

2) Count overlaping SVtypes:
```
bcftools query -T ${out}bayestyper/batch_genos -f '%SVTYPE\n' \
    ${out}bayestyper/batch_filtered/02_batch_filtered_norm.vcf.gz | sort | uniq -c

bcftools query -T ${out}bayestyper/joint_genos -f '%SVTYPE\n' \
    ${out}bayestyper/joint_filtered/02_joint_filtered_norm.vcf.gz | sort | uniq -c
```

Found that 76 SVs don't overlap in the batch data and 126 SVs don't overlap in the joint data.

3) Find non-overlapping sites (i.e., those present in genotyped output, but absent in call set):
```

```

4) Annotate VCF for individual counts:
First prepare the annotation file by identifying the resolvable SVs and their types:
```
bcftools query -f '%CHROM\t%POS\n' batch_filtered/07_batch_filtered_genotypes.vcf > batch_geno_sites
bcftools query -T batch_geno_sites -f '%CHROM\t%POS\t%SVTYPE\t%SVLEN\n' batch_filtered/02_batch_filtered_norm.vcf.gz > batch_conversion

bcftools query -f '%CHROM\t%POS\n' joint_filtered/07_joint_filtered_genotypes.vcf > joint_geno_sites
bcftools query -T joint_geno_sites -f '%CHROM\t%POS\t%SVTYPE\t%SVLEN\n' joint_filtered/02_joint_filtered_norm.vcf.gz > joint_conversion
```

Then edit the ```batch_conversion``` file so the first line in this file contains ```#CHROM POS SVTYPE  SVLEN``` with nano. 

And create a file that captures the information to be appended into the header of the VCF.

Contents of annots.hdr for the Sniffles SV calls for example:
```
##INFO=<ID=SVTYPE,Number=1,Type=String,Description="Type of structural variant called by Sniffles">
##INFO=<ID=SVLEN,Number=1,Type=String,Description="Length of structural variant called by Sniffles">
```
Now time to annotate the file to continue with the genotype based analyses.
```
bgzip ${cute}conversion
bgzip ${sniff}conversion

tabix -s1 -b2 -e2 ${cute}conversion.gz
tabix -s1 -b2 -e2 ${sniff}conversion.gz

bcftools annotate -a ${cute}conversion.gz -h ${cute}annots.hdr \
    -c CHROM,POS,SVTYPE,SVLEN -O v -o ${cute}06_cuteSV_annotated.vcf \
    ${cute}05_cuteSV_SV_genotypes.vcf

bcftools annotate -a ${sniff}conversion.gz -h ${sniff}annots.hdr \
    -c CHROM,POS,SVTYPE,SVLEN -O v -o ${sniff}06_sniffles_annotated.vcf \
    ${sniff}05_sniffles_SV_genotypes.vcf
```

## Mendelian Inheritance Tests
To test the consistency of genotyping results, tests of mendelian inheritance were used. Sites that conformed to mendelian expectations for 120 trios were found with:
```
bcftools +mendelian -m a -T ${trio} -O v \
    -o ${cute}07_cuteSV_trios.vcf \
    ${cute}06_cuteSV_annotated.vcf
bcftools +mendelian -m a -T ${trio} -O v \
    -o ${sniff}07_sniffles_trios.vcf \
    ${sniff}06_sniffles_annotated.vcf
```
The proportion of genotyped sites that adhered to Mendelian inheritance expectations were found with:
```
for i in {0,0.05,0.1,0.2}
    do
    bcftools query -i "(MERR / N_PASS(GT!="mis")) <=${i}" \
        -f '%CHROM\t%POS\t%END\t%SVLEN\t%SVTYPE\t${i}_fail_cuteSV_genofilter\n' \
        ${cute}07_cuteSV_trios.vcf >> /kakapo-data/bayestyper/summary/cuteSV_mendel.tsv
    bcftools query -i "(MERR / N_PASS(GT!="mis")) <=${i}" \
        -f '%CHROM\t%POS\t%END\t%SVLEN\t%SVTYPE\t${i}_fail_sniffles_genofilter\n' \
        ${sniff}07_sniffles_trios.vcf >> /kakapo-data/bayestyper/summary/sniffles_mendel.tsv
done
```
## Summarising the number of SVs carried by individuals

```
while read -r line
    do
    indiv=$(echo $line | awk '{print $1}')
    gen=$(echo $line | awk '{print $2}')
    echo "Counting SVs for $indiv..."
    bcftools view -s ${indiv} ${out}bayestyper/batch_filtered/09_batch_annotated_trios.vcf | bcftools query -i 'GT!= "RR" & GT!="mis"' -f '[%SAMPLE]\t%CHROM\t%POS\t%END\t%SVLEN\t%SVTYPE\n' | awk -v var="$gen" '{print $0"\t"var}' >> ${out}bayestyper/summary/manta_batch_generations.tsv
    bcftools view -s ${indiv} ${out}bayestyper/joint_filtered/09_joint_annotated_trios.vcf | bcftools query -i 'GT!= "RR" & GT!="mis"' -f '[%SAMPLE]\t%CHROM\t%POS\t%END\t%SVLEN\t%SVTYPE\n' | awk -v var="$gen" '{print $0"\t"var}' >> ${out}bayestyper/summary/manta_joint_generations.tsv
done < /kakapo-data/metadata/generations.tsv
```


## Lineage Comparisons
```
mkdir -p ${out}bayestyper/lineage_{batch,joint}_comparisons

bcftools view -s M /kakapo-data/bwa/manta/raw_variants/batch_total.vcf.gz | \
    bcftools view -i 'GT=="alt"' -O z -o ${out}bayestyper/lineage_batch_comparisons/unfiltered/RH_unfiltered_variants.vcf.gz
bcftools view -s M /kakapo-data/bwa/manta/raw_variants/joint_total.vcf.gz | \
    bcftools view -i 'GT=="alt"' -O z -o ${out}bayestyper/lineage_joint_comparisons/unfiltered/RH_unfiltered_variants.vcf.gz

bcftools view -s ^M,G,F,R,N,P,O,S /kakapo-data/bwa/manta/raw_variants/batch_total.vcf.gz |\
    bcftools view -i 'GT=="alt"' -O z -o ${out}bayestyper/lineage_batch_comparisons/unfiltered/SI_unfiltered_variants.vcf.gz
bcftools view -s ^M,G,F,R,N,P,O,S /kakapo-data/bwa/manta/raw_variants/joint_total.vcf.gz |\
    bcftools view -i 'GT=="alt"' -O z -o ${out}bayestyper/lineage_joint_comparisons/unfiltered/SI_unfiltered_variants.vcf.gz

bcftools index ${out}bayestyper/lineage_batch_comparisons/unfiltered/RH_unfiltered_variants.vcf.gz
bcftools index ${out}bayestyper/lineage_batch_comparisons/unfiltered/SI_unfiltered_variants.vcf.gz

bcftools index ${out}bayestyper/lineage_joint_comparisons/unfiltered/RH_unfiltered_variants.vcf.gz
bcftools index ${out}bayestyper/lineage_joint_comparisons/unfiltered/SI_unfiltered_variants.vcf.gz

bcftools isec ${out}bayestyper/lineage_batch_comparisons/unfiltered/RH_unfiltered_variants.vcf.gz \
    ${out}bayestyper/lineage_batch_comparisons/unfiltered/SI_unfiltered_variants.vcf.gz \
    -p ${out}bayestyper/lineage_batch_comparisons/unfiltered/

bcftools isec ${out}bayestyper/lineage_joint_comparisons/unfiltered/RH_unfiltered_variants.vcf.gz \
    ${out}bayestyper/lineage_joint_comparisons/unfiltered/SI_unfiltered_variants.vcf.gz \
    -p ${out}bayestyper/lineage_joint_comparisons/unfiltered/

bcftools query -f '%CHROM\t%POS\n' \
    ${out}bayestyper/lineage_batch_comparisons/unfiltered/0000.vcf > ${out}bayestyper/lineage_batch_comparisons/unfiltered/Fiordland_unfiltered_private_sites.txt
bcftools query -f '%CHROM\t%POS\n' \
    ${out}bayestyper/lineage_batch_comparisons/unfiltered/0001.vcf > ${out}bayestyper/lineage_batch_comparisons/unfiltered/Rakiura_unfiltered_private_sites.txt
bcftools query -f '%CHROM\t%POS\n' \
    ${out}bayestyper/lineage_batch_comparisons/unfiltered/0002.vcf > ${out}bayestyper/lineage_batch_comparisons/unfiltered/Shared_unfiltered_sites.txt

bcftools query -f '%CHROM\t%POS\n' \
    ${out}bayestyper/lineage_joint_comparisons/unfiltered/0000.vcf > ${out}bayestyper/lineage_joint_comparisons/unfiltered/Fiordland_unfiltered_private_sites.txt
bcftools query -f '%CHROM\t%POS\n' \
    ${out}bayestyper/lineage_joint_comparisons/unfiltered/0001.vcf > ${out}bayestyper/lineage_joint_comparisons/unfiltered/Rakiura_unfiltered_private_sites.txt
bcftools query -f '%CHROM\t%POS\n' \
    ${out}bayestyper/lineage_joint_comparisons/unfiltered/0002.vcf > ${out}bayestyper/lineage_joint_comparisons/unfiltered/Shared_unfiltered_sites.txt

while read -r line
    do
    indiv=$(echo $line | awk '{print $1}')
    gen=$(echo $line | awk '{print $2}')
    echo "Counting SVs for $indiv..."
    bcftools view -s ${indiv} -R ${out}bayestyper/lineage_batch_comparisons/unfiltered/Fiordland_unfiltered_private_sites.txt raw_variants/batch_total.vcf.gz | bcftools query -i 'GT=="alt"' -f '[%SAMPLE]\t%CHROM\t%POS\t%END\t%SVTYPE\tFiordland_unfiltered_lineage\n' >> ${out}mantaB_lineage_counts.tsv
    bcftools view -s ${indiv} -R ${out}bayestyper/lineage_batch_comparisons/unfiltered/Rakiura_unfiltered_private_sites.txt raw_variants/batch_total.vcf.gz | bcftools query -i 'GT=="alt"' -f '[%SAMPLE]\t%CHROM\t%POS\t%END\t%SVTYPE\tRakiura_unfiltered_lineage\n' >> ${out}mantaB_lineage_counts.tsv
    bcftools view -s ${indiv} -R ${out}bayestyper/lineage_batch_comparisons/unfiltered/Shared_unfiltered_sites.txt raw_variants/batch_total.vcf.gz | bcftools query -i 'GT=="alt"' -f '[%SAMPLE]\t%CHROM\t%POS\t%END\t%SVTYPE\tShared_unfiltered_lineage\n' >> ${out}mantaB_lineage_counts.tsv
done < /kakapo-data/metadata/generations.tsv

while read -r line
    do
    indiv=$(echo $line | awk '{print $1}')
    gen=$(echo $line | awk '{print $2}')
    echo "Counting SVs for $indiv..."
    bcftools view -s ${indiv} -R ${out}bayestyper/lineage_joint_comparisons/unfiltered/Fiordland_unfiltered_private_sites.txt ${out}raw_variants/joint_total.vcf.gz | bcftools query -i 'GT=="alt"' -f '[%SAMPLE]\t%CHROM\t%POS\t%END\t%SVTYPE\tFiordland_unfiltered_lineage\n' >> ${out}mantaJ_lineage_counts.tsv
    bcftools view -s ${indiv} -R ${out}bayestyper/lineage_joint_comparisons/unfiltered/Rakiura_unfiltered_private_sites.txt ${out}raw_variants/joint_total.vcf.gz | bcftools query -i 'GT=="alt"' -f '[%SAMPLE]\t%CHROM\t%POS\t%END\t%SVTYPE\tRakiura_unfiltered_lineage\n' >> ${out}mantaJ_lineage_counts.tsv
    bcftools view -s ${indiv} -R ${out}bayestyper/lineage_joint_comparisons/unfiltered/Shared_unfiltered_sites.txt ${out}raw_variants/joint_total.vcf.gz | bcftools query -i 'GT=="alt"' -f '[%SAMPLE]\t%CHROM\t%POS\t%END\t%SVTYPE\tShared_unfiltered_lineage\n' >> ${out}mantaJ_lineage_counts.tsv
done < /kakapo-data/metadata/generations.tsv

```