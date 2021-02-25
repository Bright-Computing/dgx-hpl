#!/bin/bash
#SBATCH -N 1
#SBATCH --time=20:00
#SBATCH --ntasks-per-node=8  
#SBATCH --job-name "test-hpl"
#SBATCH --output=slurm-%x.%J.%N.out


help() {

# selfscript=hpl-singularity-dgx-a100-mN_pmi.sub
selfscript="$(basename $0)"

cat << EOF
Description
===========
Orchestrates HPL runs via PMI using "nvcr.io/nvidia/hpc-benchmarks:20.10-hpl"
container converted to singularity. This singularity container path is set to:
    \${HOME}/singularity_images/hpc-benchmarks_20.10-hpl.sif
Override the path to the container via --container option.

This script is hard-coded for the HPL runs to be done on DGX-A100. The basedir
is set to:
    \${HOME}/projects/hpl_tests
Override the basedir via --basedir option.

The basedir is only used to lookup the path to the hpldatfiles directory.
    \${_basedir}/hpldatfiles
If an explicit path is set to hpldatdir via option --hpldatdir then basedir is
not used for anything.

The following non-standard parameters are set in the script to override
defaults within containers hpl.sh script:
    UCX_AFFINITY="mlx5_0:mlx5_1:mlx5_2:mlx5_3:mlx5_4:mlx5_5:mlx5_6:mlx5_7"
The above is for DGX-A100 with Slot 5 NOT populated. When Slot 5 is populated
these should be set to:
    UCX_AFFINITY="mlx5_0:mlx5_1:mlx5_2:mlx5_3:mlx5_6:mlx5_7:mlx5_8:mlx5_9"

The multinode orchestration is done by PMI (Process Management Interface).
Specifically the PMI-2 is set via option to srun (srun --mpi=pmi2). Refer to:
https://openpmix.github.io/support/faq/how-does-pmix-work-with-containers

HPL typically runs on sets of nodes of power of 2 i.e. 1, 2, 4, 8, 16, ...
The HPL dat files are parameters for HPL. These need to be placed at:
    \${_basedir}/hpldatfiles
Override the path to the HPL dat files via --hpldatdir

Four files are provided:
    HPL.dat_4x2_dgxa100_40G, HPL.dat_4x4_dgxa100_40G, HPL.dat_8x4_dgxa100_40G, HPL.dat_8x8_dgxa100_40G
Corresponding to 1, 2, 4 or 8 node runs. The files are automatically set in this
script based on the number of nodes allocated.

Usage Examples
==============
This script cannot be run via srun directly. Run it via sbatch. To run
interactively first salloc then just run the script. When run via sbatch the
results will be written to file:
    slurm-test-hpl.<JOBID>.<First-Node-Name>.out
For interactive runs the results will be printed to stdout.

Single node
-----------
Default SBATCH parameters in the script are:
#SBATCH -N 1
#SBATCH --time=20:00
#SBATCH --ntasks-per-node=8  
#SBATCH --job-name "test-hpl"
#SBATCH --output=slurm-%x.%J.%N.out

Submit via sbatch for default 1 node run.
$ sbatch $selfscript

To run interactively via salloc requires specifying Slurm options at command line:
$ salloc -N 1 --ntasks-per-node=8
$ ./$selfscript
$ exit # exit salloc session

The --ntasks-per-node option needs to match number of GPUs per node. On
DGX-A100 this is 8 GPUs.

Multinode
---------
The multinode runs are done by varying -N|--nodes=<minnodes[-maxnodes]> with
the Slurm sbatch or salloc submissions. 

Example two node run:
$ sbatch -N 2 --ntasks-per-node=8 $selfscript

Example with node selection. Running on dgx-05 to dgx-08:
$ sbatch --nodelist=dgx-[05-08] -N 4 --ntasks-per-node=8 $selfscript

Options
=======
$selfscript [options]

    -h|--help
        * Prints description and options.

    --basedir <Base directory>
        * Default: \${HOME}/projects/hpl_tests

    --container <Path to the singularity container>
        * Override default container path:
            \${HOME}/singularity_images/hpc-benchmarks_20.10-hpl.sif

          Getting the container:
            export SINGULARITY_DOCKER_USERNAME='$oauthtoken'
            export SINGULARITY_DOCKER_PASSWORD=<NVIDIA NGC API key>
            singularity pull docker://nvcr.io/nvidia/hpc-benchmarks:20.10-hpl

    --hpldatdir <Path to the directory with HPL dat files>
        * Default: \${_basedir}/hpldatfiles

Example with options:
$ sbatch -N 2 --ntasks-per-node=8 $selfscript --hpldatdir \${HOME}/hpldatfiles

EOF

}

# Parse options and set paths -------------------------------------------------
_basedir=${HOME}/projects/hpl_tests

# IMP NOTE: download the SIF image of the container using the following
# command before executing this batch script:
#     export SINGULARITY_DOCKER_USERNAME='$oauthtoken'
#     export SINGULARITY_DOCKER_PASSWORD=<NVIDIA NGC API key>
#     singularity pull docker://nvcr.io/nvidia/hpc-benchmarks:20.10-hpl
CONT="${HOME}/singularity_images/hpc-benchmarks_20.10-hpl.sif"


while [ $# -gt 0 ]; do
    case "$1" in
        -h|--help) help ; exit 0 ;;
        --basedir) _basedir="$2"; shift 2 ;;
        --container) CONT="$2"; shift 2 ;;
        --hpldatdir) HPLDATDIR="$2"; shift 2 ;;
        *) echo "Option <$1> Not understood" ; exit 1 ;;
    esac
done

if [ x"${HPLDATDIR}" == x"" ]; then
    HPLDATDIR="${_basedir}/hpldatfiles"
fi

# Print some basic info -------------------------------------------------------
DATESTRING=`date "+%Y-%m-%dT%H:%M:%S"`
echo "Running on hosts: $(echo $(scontrol show hostname))"
echo "$DATESTRING"


# Parse slurm env vars and set HOSTLIST ---------------------------------------
NNODES=${SLURM_NNODES}
NGPUS=${SLURM_NTASKS_PER_NODE}

# Setup HPL Dat file -----------------------------------------------------------
system=dgxa100_40G

errmsg="ERROR: There is no defined mapping for ${NNODES} nodes for system "\
"${system}.  Exiting"

case ${NNODES} in
    1) PxQ=4x2 ;;
    2) PxQ=4x4 ;;
    4) PxQ=8x4 ;;
    8) PxQ=8x8 ;;
    *) echo ${errmsg} && exit 1 ;; 
esac

HPLDATFILE=HPL.dat_${PxQ}_${system}
# Path to mount in singularity. 
MOUNT="${HPLDATDIR}:/datfiles"

# Setup singularity -----------------------------------------------------------
module load singularity

export SINGULARITYENV_SHELL=/bin/bash

# Launch via PMI2 -------------------------------------------------------------

# DGX-A100 config
CPU_AFFINITY="32-47:48-63:0-15:16-31:96-111:112-127:64-79:80-95"
CPU_CORES_PER_RANK=16
GPU_AFFINITY="0:1:2:3:4:5:6:7"
MEM_AFFINITY="2:3:0:1:6:7:4:5"

# UCX affinities for DGX-A100 when Slot 5 is populated
# UCX_AFFINITY="mlx5_0:mlx5_1:mlx5_2:mlx5_3:mlx5_6:mlx5_7:mlx5_8:mlx5_9"

# UCX affinities for DGX-A100 when Slot 5 is NOT populated
UCX_AFFINITY="mlx5_0:mlx5_1:mlx5_2:mlx5_3:mlx5_4:mlx5_5:mlx5_6:mlx5_7"

# To unset ucx affinities
# export UCX_TLS=all
# export SINGULARITYENV_UCX_TLS=${UCX_TLS}
# export UCX_MAX_RNDV_RAILS=1
# export SINGULARITYENV_UCX_MAX_RNDV_RAILS=${UCX_MAX_RNDV_RAILS}
# UCX_AFFINITY=":::::::"

# The clock setting is commented out within hpl.sh. Modifying clock settings is
# done when trying to maximize performance such as for TOP500 submission, but
# in daily operations clocks should be left at default settings.
GPU_CLOCK="1380,1410"

# export OMPI_MCA_hwloc_base_binding_policy=none

SRUNCMD=$(cat <<EOF

srun --export=ALL \
 --mpi=pmi2 \
 -N ${NNODES} \
 --ntasks-per-node=${NGPUS} \
 singularity run  --nv -B ${MOUNT} ${CONT} \
   /workspace/hpl.sh \
    --cpu-affinity "$CPU_AFFINITY" \
    --cpu-cores-per-rank $CPU_CORES_PER_RANK \
    --gpu-affinity "$GPU_AFFINITY" \
    --mem-affinity "$MEM_AFFINITY" \
    --ucx-affinity "$UCX_AFFINITY" \
    --clock "$GPU_CLOCK" \
    --dat "/datfiles/${HPLDATFILE}"

EOF
)

# Above options are manually set to override ucx-affinity. Default dgx-a100
# config could be set via:
#  /workspace/hpl.sh \
#   --config dgx-a100 \

# The --config sets cpu, gpu, mem, and ucx affinities.

# Some hpl dat files are included in the container:
#   /workspace/hpl-linux-x86_64/sample-dat/HPL-dgx-a100-1N.dat
#   /workspace/hpl-linux-x86_64/sample-dat/HPL-dgx-a100-2N.dat

echo SRUNCMD: $SRUNCMD

$SRUNCMD




