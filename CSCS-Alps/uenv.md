# uenv


Software on Alps is provided via [uenv] (user environments).

## Spack

uenv is based on the [Spack] package manager, 
therefore all the software in an uenv needs to be available in Spack.

If a software is not available in Spack, you need to create a Spack package for it,
following the [Spack packaging guide].

## Recipes

A [uenv recipe] is a collection of YAML files describing the uenv (compilers, libraries, applications, etc.).
An uenv can be built from a recipe using [`stackinator`], or using the [uenv build serivice].

Uenv recipes for the metatensor ecosystem are available under

```
CSCS-Alps/uenv/
```

### Updating a package version

To update a package version in an existing recipe, you need to do the following:

1. [Optional] Update the `spack-package` version/commit in the `config.yaml` file,
2. Update the version of the package in the `environment.yaml` file.

The first step is only required if the version of interest has been added to Spack
after the version/commit of `spack-packages` specified in the `config.yaml` file.

### Custom Spack packages

Custom Spack packages, overwriting the ones in Spack, can be added to

```
<PATH_TO_RECIPE>/repo/packages/
```

## Examples

### Build uenv with build service

Build an uenv from a recipe using the [uenv build serivice]:

```bash
uenv build <PATH_TO_RECIPE> 
```

### Build uenv with stackinator

[Install `stackinator`]. 
Clone the [Alps cluster configuration] repository.

Build an uenv from a recipe using `stackinator`:

```bash
stack-config --build <PATH_TO_BUILD_DIR> --recipe <PATH_TO_RECIPE> -S <PATH_TO_CLUSTER_CONFIG>
```

It is recommended to build under `/dev/shm/$USER/`.
The path to the cluster configuration should point to the correct cluster directory in the `alps-cluster-config` repository.

## Available recipes

### lammps-metatomic

Recipe for building an uenv with LAMMPS and the `metatomic` package.
The LAMMPS Spack package is a custom package, which uses [metatensor/LAMMPS],
since the `metatomic` package is not yet available in the official LAMMPS distribution.

The main changes are the following:
1. Changed `git` attribute to use `metatensor/LAMMPS`
2. Added `+metatomic` variant to enable `metatomic` (`lammps+metatomic`)
3. Added dependencies `libmetatensor-torch`, `libmetatomic-torch`, `py-torch` (when `+metatomic`)
4. Enabled `PKG_ML-METATOMIC` when `+metatomic`, and enforced usage of Spack-installed `metatensor` and `metatomic`
5. Removed `^[virtuals=mpi] cray-mpich`-specific handling, it's already accounted for in Alps' custom `cray-mpich` package 

The version of PyTorch provided by this uenv is non-distributed.

[uenv]: https://docs.cscs.ch/software/uenv/
[Spack]: https://spack.readthedocs.io/en/latest/
[Spack packaging guide]: https://spack.readthedocs.io/en/latest/packaging_guide_creation.html#
[`stackinator`]: https://eth-cscs.github.io/stackinator/
[uenv build serivice]: https://docs.cscs.ch/software/uenv/build_service/
[uenv recipe]: https://eth-cscs.github.io/stackinator/recipes/
[Install `stackinator`]: https://eth-cscs.github.io/stackinator/#getting-stackinator
[Alps cluster configuration]: https://github.com/eth-cscs/alps-cluster-config
