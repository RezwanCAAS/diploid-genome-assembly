# dragon-fruit-genome-assembly

# 1-PacBio HiFi reads are assembled using hifiasm

	module load hifiasm/0.18.5
	hifiasm -o condor_assembly -t 32 condor_hifi_reads_*
 
# Canu is also a second option for assembly
	#!/bin/bash
	#
	#SBATCH --job-name=condor_canu
	#SBATCH --output=condor_canu.%j.out
	#SBATCH --partition=batch
	#SBATCH --cpus-per-task=32
	#SBATCH --time=25:00:00
	#SBATCH --mem=700G
	
	module load canu/1.8/gnu6.4.0
	
	time -p canu -p condor_assembly -d condor_assembly_canu genomeSize=2g -pacbio-hifi condor_hifi_reads_* usegrid=1 gridOptions="--time=25:00:00 -p batch" gridOptionsJobName=my-grid-job

# 2-convert .gfa into fasta formate
	awk '/^S/{print ">"$2"\n"$3}' condor_assembly.bp.p_ctg.gfa > condor_assembly.bp.p_ctg.fasta

# 3-evaluate the assembly with quast
	module load quast/5.2.0
	quast.py condor_assembly.bp.p_ctg.fasta -o assembly_stats

# 4-calculate BUSCO using compleasm
	module load compleasm/0.2.2
	
	compleasm.py run -a condor_assembly.bp.p_ctg.fasta \
	-l embryophyta_odb10 \
	-L /ibex/scratch/projects/c2141/dragon-fruit/hifireads_condor/condor_hifiasm/miniBUSCO_embryophyta/ \
	-o condor_miniBUSCO_embryophyta_odb10 \
	-m busco \
	-t 30 \

