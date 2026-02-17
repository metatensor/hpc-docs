# Running molecular dynamics simulations on Alps with `metatomic` and `i-pi`

Last update 17-02-2026.

CSCS recommends running machine learning-related applications in containers, see [this page](https://docs.cscs.ch/software/ml/#optimizing-data-loading-for-machine-learning) for reference. In this tutorial, we use the [Pytorch NGC container](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch) provided by Nvidia to fully exploit the power of the GH200s installed on daint.alps.

## Selecting and Building a Base Container

To start, first pick a version of the container that you like by clicking the `Get Container` button of [this page](https://docs.cscs.ch/software/ml/#optimizing-data-loading-for-machine-learning) and copy the image path. Then, create a new folder, say `MD-container` for the following steps.

```bash
mkdir MD-container
cd MD-container
```

Inside `MD-container`, create a file `Containerfile`. Later on, we will create a custom container base on the NGC container that you chose and the custom commands written in the `Containerfile`. The `Containerfile` for now looks like:

```Dockerfile
FROM <image_path:you_chose>
```

Build the container:

```bash
podman build -t <image_name:you_like> .
```

At this stage, the container is identical to the upstream NGC image. You can enter this container with:

```bash
podman run --rm -it --gpus all <image_name:you_like> bash
```

## Making Persistent Modifications

After entering the container, you can start configure your simulation environment as usual. However, be sure that every command are recorded, because your modification to this loaded container is temporary and will lose after you exit. To do permenant modifications, you should write down the commands in the `Containerfile`. The way to do so is to add them with the `RUN` keyword:

```Dockerfile
FROM <image_path:you_chose>

RUN command_0 \
    && command_1 \
    && ...
```

It is also possible to setup the environmental variables in the `Containerfile` too, with:

```Dockerfile
ENV VARIABLE_NAME=value
```

Rebuild the container:

```bash
podman build -t <image_name:you_like> .
```

You can save the modified container:

```bash
podman save -o /path/to/container.tar <image_name:you_like>
```

Next time, you can use the container by first loading it with:

```bash
podman load -i /path/to/container.tar
```

and then enter it with:

```bash
podman run --rm -it --gpus all <image_name:you_like> bash
```

## Example Containerfile for `metatomic + i-Pi + PLUMED`

Below is an example `Containerfile` that

- installs micromamba
- creates a Python environment
- compiles `metatomic[torch]`
- builds PLUMED

```Dockerfile
# The image path of the nvidia container that you want to use
FROM nvcr.io/nvidia/pytorch:26.01-py3 

# Install micromamba and create conda environment
RUN curl -Ls https://micro.mamba.pm/api/micromamba/linux-aarch64/latest | tar -xvj bin/micromamba \
    && eval "$(./bin/micromamba shell hook -s posix)" \
    && ./bin/micromamba shell init -s bash -r ~/micromamba \
    && source ~/.bashrc \
    && micromamba activate \
    && micromamba create -n sim python==3.12.* -y \
# Ensure system torch is accessible in the conda env
    && echo "/usr/local/lib/python3.12/dist-packages" > /root/micromamba/envs/sim/lib/python3.12/site-packages/system_torch.pth \
# Build your environment
    && micromamba activate sim \
    && pip install ipython metatensor metatomic ipi ase \
    && pip install metatensor[torch] --no-build-isolation  --no-binary=metatensor-torch \
    && pip install metatomic[torch] --no-build-isolation  --no-binary=metatomic-torch \
    && pip install vesin[torch] --no-build-isolation  --no-binary=vesin-torch \
    && wget https://github.com/plumed/plumed2/archive/refs/tags/v2.10.0.tar.gz -O /tmp/plumed.tar.gz \
    && tar -xvf /tmp/plumed.tar.gz -C /tmp \
    && cd /tmp/plumed2-2.10.0 \
    && CPPFLAGS=$(python src/metatomic/flags-from-python.py --cppflags) \
    && LDFLAGS=$(python src/metatomic/flags-from-python.py --ldflags) \
    && ./configure PYTHON_BIN=python --enable-libtorch --enable-libmetatomic --enable-modules=+metatomic LDFLAGS="$LDFLAGS" CPPFLAGS="$CPPFLAGS" \
    && make -j \
    && make install

ENV PLUMED_KERNEL=/usr/local/lib/libplumedKernel.so
ENV PYTHONPATH="/usr/local/lib/plumed/python:$PYTHONPATH"

WORKDIR /workspace
```

## Example SLURM Submission Script

The following is an example job submission script, in which we launch four tasks, each on a unique GPU.

```bash
#!/bin/bash -l
#SBATCH -o job.%j.out
#SBATCH --job-name=metad-run
#SBATCH --nodes 1
#SBATCH --ntasks-per-node=4
#SBATCH --cpus-per-task=40
#SBATCH --time 1-00:00:00
#SBATCH --gres=gpu:4


set -e

# Load the container image
IMAGE_TAR=/path/for/saving_the_container
IMAGE_NAME=<image_name:you_like>
podman load -i $IMAGE_TAR

run_job () {
    local NAME=$1
    local GPU=$2
    WORKDIR=/your/work_dir/simulation_${NAME}/
    IPIFILE=/your/work_dir/parameters/${NAME}_input.xml

    cd $WORKDIR

    srun --cpus-per-task=$SLURM_CPUS_PER_TASK --ntasks=1 --gpus=1 \
        --gpu-bind=single:1 \
        --export=ALL,CUDA_VISIBLE_DEVICES=$GPU,NVIDIA_VISIBLE_DEVICES=$GPU \
        podman run --rm \
        --gpus all \
        --env CUDA_VISIBLE_DEVICES=$GPU \
        --env NVIDIA_VISIBLE_DEVICES=$GPU \
        # map your directories to the directories in the container
        -v /users/<your_username>:/users/<your_username> \
        -v /capstor/scratch/cscs/<your_username>:/capstor/scratch/cscs/<your_username> \
        -w "${WORKDIR}" \
        "${IMAGE_NAME}" \
        bash -lc "
            export OMP_NUM_THREADS=1
            export MKL_NUM_THREADS=1
            export TORCH_NUM_THREADS=1
            /workspace/bin/micromamba run -n sim i-pi ${IPIFILE}
        " &
}

run_job H3PO4   0
run_job NaH2PO4 1
run_job Na2HPO4 2
run_job Na3PO4  3

wait

exit 0
```
