# Running molecular dynamics simulations on Alps with `metatomic` and `i-pi`

Last update 02-03-2026.

Containers are powerful tools that give you almost full control over the environment of your application and ensure the reproducibility of the environment.

CSCS is equipped with [the Container Engine (CE) toolset](https://docs.cscs.ch/software/container-engine/). This toolset is tightly integrated into the Slurm workload manager, allowing running the tasks inside the containers with little extra effort.

Container images are generally available online. Specifically for CSCS, there are [images](https://docs.cscs.ch/software/alps-extended-images/) modified for fully leverage the ability of `daint.alps` based on the [NGC containers](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch?version=26.01-py3-igpu).

## Selecting and Building a Base Container

To start, first pick a version of the container at this [Alps Extended Images](https://docs.cscs.ch/software/alps-extended-images/) page and copy the image URL. Then, create a new folder, say `MD-container` for the following steps.

```bash
mkdir MD-container
cd MD-container
```

> [!note]
> Building the image consumes lots of computational resources, so it is recommended to do the following steps on an interactive computational node. To do so, use this command `srun -t 01:00:00 --pty bash`.

Inside `MD-container`, create a file `Containerfile`. Later on, we will create a custom container base on the container that you chose and the custom commands written in the `Containerfile`. The `Containerfile` for now looks like:

```Dockerfile
FROM <image_path:you_chose>
```

Build the container, and also choose a name for it:

```bash
IMAGE_NAME=<image_name:you_like>
podman build -t ${IMAGE_NAME} .
```

At this stage, the container is identical to the upstream image. You can enter this container with `srun`, but we need to:

1. save the container image;
2. write down the configuration of this container for Container Engine.

To save:

```bash
SQSH_IMAGE_FILE=/path/to/container.sqsh
enroot import -x mount  -o ${SQSH_IMAGE_FILE} "podman://${IMAGE_NAME}"
```

To configure, we need to write the following contents in to `$HOME/.edf/<container_nae>.toml`

```toml
image = "/path/to/container.sqsh"
# Make your folders accessible inside the container, format "/path/in/container:/path/on/your/disk"
mounts = ["${SCRATCH}:${SCRATCH}", "/your/workdir:/your/workdir"]
workdir = "/your/workdir"
```

Then, enter the container with

```bash
srun --environment=<container_name> --pty bash
```

This opens an interactive node and loads the container specified in `$HOME/.edf/<container_nae>.toml` for you.

## Making Persistent Modifications

After entering the container, you can play with it and configure your simulation environment as usual. However, be sure that every command are recorded, because your modification to this loaded container is *temporary and will lose* after you exit.

To do permenant modifications, you should write down the commands in the `Containerfile`. The way to do so is to add them with the `RUN` keyword:

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
podman build -t ${IMAGE_NAME} .
```

You can save the modified container, but you need to first remove the previous image file:

```bash
rm ${SQSH_IMAGE_FILE}
enroot import -x mount  -o ${SQSH_IMAGE_FILE} "podman://${IMAGE_NAME}"
```

Next time, you can use the container by first loading it with:

```bash
srun --environment=<container_name> --pty bash
```

## Example Containerfile for `metatomic + i-Pi + PLUMED`

Below is an example `Containerfile`

```Dockerfile
# The image path of the nvidia container that you want to use
FROM jfrog.svc.cscs.ch/docker-group-csstaff/alps-images/ngc-pytorch:26.01-py3-alps2

RUN pip install --no-cache-dir \
        ase \
        metatensor \
        metatrain \
        ipi \
        mace-torch \
        sphericart-torch \
        torch-spex \
        torch-ema \
    && pip install --no-cache-dir --no-build-isolation --no-binary=metatensor-torch \
        metatensor[torch] \
    && pip install --no-cache-dir --no-build-isolation --no-binary=metatomic-torch \
        metatomic[torch] \
    && pip install --no-cache-dir --no-build-isolation --no-binary=vesin-torch \
        vesin[torch]

WORKDIR /workspace
```

## Example SLURM Submission Script

The following is an example job submission script, in which we launch four tasks, each on a unique GPU.

```bash
#!/bin/bash
#SBATCH -o job.%j.out
#SBATCH -J example_job
#SBATCH --time=24:00:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH --cpus-per-task=4
#SBATCH --gpus-per-task=1
#SBATCH --exclusive

run_job () {
    local system=$1
    srun --gpus=1 --gpu-bind=single:1 --environment=<container_name> bash -lc "
        mkdir ${SCRATCH}/results
        cd ${SCRATCH}/results
        mkdir $system
        cd $system
        i-pi /your/workdir/${system}.xml
    " &
}

run_job water_0
run_job water_1
run_job water_2
run_job water_3

wait

exit 0
```
