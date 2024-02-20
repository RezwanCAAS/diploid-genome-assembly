# dragon-fruit-genome-assembly

# PacBio HiFi reads are assembled using hifiasm

	module load hifiasm/0.18.5
	hifiasm -o condor_assembly -t 32 condor_hifi_reads_*
 
# Canu is also a second option for assembly
