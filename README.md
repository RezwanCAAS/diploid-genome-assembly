# dragon-fruit-genome-assembly

# 1-PacBio HiFi reads are assembled using hifiasm

module load hifiasm/0.19.8

hifiasm -o condor_assembly -t 32 -s 0.30 -D 10 \
 --h1 /hifireads/hic_dir/read_1_trim.fastq.gz \
 --h2 /hifireads/hic_dir/read_2_trim.fastq.gz \
 hifi_reads_*
 

# 2-convert .gfa into fasta formate
	awk '/^S/{print ">"$2"\n"$3}' assembly.bp.p_ctg.gfa > assembly.bp.p_ctg.fasta

# 3-evaluate the assembly with quast
	module load quast/5.2.0
	quast.py condor_assembly.bp.p_ctg.fasta -o assembly_stats

# 4-calculate BUSCO using compleasm
	module load compleasm/0.2.2
	
	compleasm.py run -a condor_assembly.bp.p_ctg.fasta \
	-l embryophyta_odb10 \
	-L path_library/miniBUSCO_embryophyta/ \
	-o condor_miniBUSCO_embryophyta_odb10 \
	-m busco \
	-t 30 \

# 5-Juicer and Juicebox for chromosomal scaffolding
#step1_index the assembly

#!/bin/bash
#SBATCH --job-name=bwa_index
#SBATCH --output=bwa_index.%j.out
#SBATCH --partition=batch
#SBATCH --cpus-per-task=10
#SBATCH --time=10:00:00
#SBATCH --mem=100G

module load bwa/0.7.17/gnu-12.2.0

bwa index assembly.hic.p_ctg.fasta assembly

#step2_indentify the Arima restriction sites in assembly
#!/bin/bash
#SBATCH --job-name=enzyme
#SBATCH --output=enzyme.%j.out
#SBATCH --partition=batch
#SBATCH --cpus-per-task=30
#SBATCH --time=25:00:00
#SBATCH --mem=200G

module load python/3.11.0
export PATH=/ibex/project/c2141/dragon-fruit/hifireads_condor/purge_haplotigs/juicer_pipeline/:$PATH

generate_site_positions.py Arima assembly assembly.hic.p_ctg.fasta

#step3_run Juicer for making the files for scaffolding
#!/bin/bash
#SBATCH --job-name=juicer
#SBATCH --output=juicer.%j.out
#SBATCH --partition=batch
#SBATCH --cpus-per-task=32
#SBATCH --time=80:00:00
#SBATCH --mem=200G

module load juicer/1.6

juicer.sh -g condor_assembly_haplotigs -s Arima -z assembly.hic.p_ctg.fasta \
 -y condor_Arima.txt \
 -p assembly \
 -t 32


#step4_run 3dna to make the hic plot and assembly for juicebox
#!/bin/bash
#SBATCH --job-name=assembly_pipeline
#SBATCH --output=assembly_pipeline.%j.out
#SBATCH --partition=batch
#SBATCH --cpus-per-task=32
#SBATCH --time=30:00:00
#SBATCH --mem=200G

module load 3d-dna/180419

run-asm-pipeline.sh -r 0 assembly.hic.p_ctg.fasta ~/juicer/aligned/merged_nodups.txt



#step4_improve the scaffolds using juicebox

#step5_make the fasta sequence after improving the scaffolds using juicebox
#!/bin/bash
#
#SBATCH --job-name=post_assembly
#SBATCH --output=post_assembly.%j.out
#SBATCH --partition=batch
#SBATCH --cpus-per-task=32
#SBATCH --time=150:00:00
#SBATCH --mem=200G

module load 3d-dna/180419 python/3.11.0
export PATH=/ibex/user/tariqr/hic_assembly/purge_haplotigs/3d-dna-master/:$PATH

run-asm-pipeline-post-review.sh -r condor_assembly_haplotigs.0_1.review.assembly assembly.hic.p_ctg.fasta \
 /ibex/user/tariqr/hic_assembly/purge_haplotigs/merged_nodups.txt
