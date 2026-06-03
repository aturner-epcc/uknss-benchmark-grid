# UK NNSS Grid benchmark

This repository contains information on the GRID benchmark for the UK NNSS
procurement. 

> [!IMPORTANT]
> Please do not contact the benchmark or code maintainers directly with any questions. All questions must be submitted via the procurement response mechanism.

## Benchmark Overview
Benchmark_Grid benchmarks three discretisations of the Dirac matrix. 
The sparse Dirac matrix benchmark assigns a 4D array to each MPI rank/GPU, referred to as the local lattice size or local volume.
It is ran for different problem sizes (e.g. 8<sup>4</sup>, 12<sup>4</sup>, 16<sup>4</sup>, 24<sup>4</sup>, 32<sup>4</sup>, 48<sup>4</sup>).
Since the local volumes are fixed, increasing the number of MPI ranks corresponds to a weak scaling of the benchmark.

## Software

Git Repository: [https://github.com/aportelli/grid-benchmark](https://github.com/aportelli/grid-benchmark)

> [!CAUTION]
> All results submitted should be based on the following repository commits:
>- grid-benchmark repository: [c7457a8](https://github.com/aportelli/grid-benchmark/commit/c7457a85b6a0d9d1578838af11477cb41b1a5764)
>- Grid repository: [e2d607f](https://github.com/paboyle/Grid/commit/e2d607f6c708362050ec26cbe89fc971a1a879c5)

The benchmark software is licensed under GPLv2, with a list of
contributors available at https://github.com/aportelli/grid-benchmark/graphs/contributors.
The benchmark uses the underpinning Grid C++ 17 library for lattice QCD applications.

> [!NOTE]
> The grid-benchmark repository contains two benchmarks: Benchmark_Grid and Benchmark_IO. Only Benchmark_Grid is subject of discussion here. 


## Building the benchmark

Compiling the code involves the following steps:
1. Configure the build environment.

   Create a suitable `grid-config.json` configuration under `grid_benchmark/systems/<your-system-name>`. Additional files (e.g. to configure environment variables can be put in a `files` sub-directory).
   
   Example build configurations are provided for:

   - [Tursa (EPCC, Scotland)](https://epcced.github.io/dirac-docs/tursa-user-guide/hardware/): CUDA 11.4, GCC 9.3.0, OpenMPI 4.1.1, UCX 1.12.0 + NVIDIA A100 GPU, NVLink, Infiniband interconnect (GPU and CPU configurations)
   - [Daint (CSCS, Switzerland)](https://docs.cscs.ch/clusters/daint/): CUDA 12.4, GCC 14.2, HPE Cray MPICH 8.1.32 + NVIDIA GH200 CPU+GPU, NVLink, Slingshot 11 interconnect (GPU configurations)
   - [LUMI-G (CSC, Finland)](https://docs.lumi-supercomputer.eu/hardware/lumig/): ROCm 6.0.3, AMD clang 17.0.1, HPE Cray MPICH 8.1.23 (custom) + AMD MI250X GPU, Infinity fabric, Slingshot 11 interconnect (GPU configuration)

   Multiple configurations can be defined for each system. Since benchmark results are requested for both CPU and GPU runs, suitable configurations for these should be defined (the names `cpu` and `gpu`, respectively, are used below).

   > [!CAUTION]
   > Please ensure the correct Grid commit is specified in this file.

2. Boostrap the build environment directory `<env-dir>`:
   ```
   cd grid-benchmark
   ./bootstrap-env.sh <env-dir> <system>
   ```

3. Build GRID for both CPU and GPU configurations
   ```
   ./build-grid.sh <env-dir> cpu <njobs>
   ./build-grid.sh <env-dir> gpu <njobs>
   ```

4. Build GRID benchmark for both CPU and GPU configurations
   ```
   ./build-benchmark.sh <env-dir> cpu <njobs>
   ./build-benchmark.sh <env-dir> gpu <njobs>
   ```
   The required benchmark executables will then be located at
   ```
   # CPU benchmark
   <env-dir>/build/Grid-benchmarks/cpu/Benchmark_Grid

   # GPU benchmark
   <env-dir>/build/Grid-benchmarks/gpu/Benchmark_Grid
   ```

Detailed build instructions can be found in the benchmark source code
repository at:
[https://github.com/aportelli/grid-benchmark/blob/main/Readme.md](https://github.com/aportelli/grid-benchmark/blob/main/Readme.md)

Further reference builds are available from [https://github.com/paboyle/Grid/tree/develop/systems](https://github.com/paboyle/Grid/tree/develop/systems).


### Pre-approved code modifications
`Benchmark_Grid` has been written with the intention that no modifications to the source code
are required.
However, source code modifications might be required in few cases. Below is a list
of permitted modifications:

- Only modify the source code to resolve unavoidable compilation or runtime errors. The
  [Grid systems directory](https://github.com/paboyle/Grid/tree/develop/systems) has many examples
  of configuration and run options known to work on a variety of systems in case of e.g. linking
  errors or runtime issues.
- For compilation on systems with only ROCm 7.x and greater available, it is permitted to use
  the workaround described below as a substitute for code modification. Workarounds
  of this nature are permitted if unresolvable compilation errors otherwise occur.
- For NVIDIA GPUs, CUDA versions 11.x or 12.x are recommended. If only CUDA 13 or more recent
  is available, we provide a workaround below.
- For AMD GPUs, ROCm version 6.x is recommended since Grid is incompatible with ROCm
  version 7.x without minor code modifications. If only ROCm 7.x is available, we provide
  a workaround below.

**CUDA 13+ workaround**

Create a new header file with the following contents:
 
```c++
#pragma once
#include <cuda/std/functional>
namespace cub
{
    using Sum = cuda::std::plus<>;
}
```
 
Add `-include /path/to/header/file` in the `CXXFLAGS` definition for the system in
`grid-config.json`.

**ROCm 7.x/hipBLAS 3.x workaround**

Both the current develop branch of Grid and the selected Grid benchmarking commit explicitly use
the hipBLAS 2.x types hipblasComplex and hipblasDoubleComplex. As of hipBLAS 3.x, which
is the version of hipBLAS included with ROCm 7.x, these types have been deprecated in favour of
hipComplex and hipDoubleComplex. This will cause a compilation failure of the form
error: use of undeclared identifier 'hipblasComplex'.

This can be worked around by adding

```
-DhipblasComplex=hipComplex -DhipblasDoubleComplex=hipDoubleComplex
```

to the `CXXFLAGS` argument passed to the `configure` command for Grid. This can be automated
using a custom preset for the automatic deployment scripts for Grid and grid-benchmark as
documented in the [grid-benchmark README](https://github.com/aportelli/grid-benchmark/).



## Running the benchmark

The benchmark descriptions below specify mandatory command line flags.
The output of the `Benchmark_Grid` code is controlled via the `--json-out <filename>` option. 
All details to be reported are to be taken from this output file.


### Benchmark execution

The following conditions on options and runtime configuration must be adhered
to for both the baseline and optimised build:

- The runtime performance is affected by the MPI rank distribution. MPI ranks
  are specified with the option `--mpi X.Y.Z.T` flag. For almost all configurations, the reporting guidelines specify exactly what values to use for X, Y, Z and T. Only for the largest experiments, benchmarkers may pick the configuration that maximises the use of the system in line with the buidelines below.
  
  To obtain results representative of realistic workloads, the following
  algorithm **must** be used for setting the MPI decomposition:
    1. Allocate ranks to T until it reaches 4, e.g. `--mpi 1.1.1.4`.
    2. Allocate ranks to Z until it reaches 4, e.g. `--mpi 1.1.4.4`.
    3. Allocate ranks to Y until it reaches 4, e.g. `--mpi 1.4.4.4`.
    4. Allocate ranks to X until it reaches 4, e.g. `--mpi 4.4.4.4`.
    5. If further ranks are required, continue to allocate evenly in powers of 2.
- The maximum local volume size **must** be set to 48<sup>4</sup> using the `--max-L 48` 
  option to `Benchmark_Grid`.
- Some test configurations may be disabled using the following options to
  `Benchmark_Grid` in order to speed up benchmark execution:
  ```
  --no-benchmark-flops-fp64
  --no-benchmark-flops-sp4-2as
  --no-benchmark-flops-sp4-f
  --no-benchmark-flops-su4
  ```
- The `Benchmark_Grid` software **must** be run with `--json-out <filename>`, which will write the results of the benchmark to a JSON file.
- The `--accelerator_threads` parameter, while not documented in the help string to `Benchmark_Grid`, may be set to multiply the number of threads used on a device and may improve performance on some systems.
- The `Benchmark_Grid` software should be run with no additional flags beyond those described above.

Besides the mandatory flags, Grid has many command-line interface flags that
control its runtime behaviour. Identifying the optimal flags, as with the 
compilation options, is system-dependent and requires experimentation. A list
of Grid flags is given by passing `--help` to `Benchmark_Grid`, and a full
list is provided for both Grid and grid-benchmark in the 
[grid-benchmark README](https://github.com/aportelli/grid-benchmark/).

For the acceptance tests, all benchmark configurations are to be submitted via scheduler scripts.
The scripts may contain arbitrary system and benchmark settings (e.g. additional environment
variables or command line instructions), but have to be run on the vanilla system without any 
benchmark-specific reconfiguration. Notably, rebooting the acceptance system for a particular 
benchmark with particular settings is not permitted.

There are example job submission scripts and launch wrapper scripts in the
[grid-benchmark systems directory](https://github.com/aportelli/grid-benchmark/tree/main/systems).
There are also run scripts for specific systems that may be closer to the target
architecture in the [Grid systems directory](https://github.com/paboyle/Grid/tree/develop/systems).

## Results

### Correctness results 

The correctness check for this package ensures that a Conjugate Gradient solve using the Dirac matrix
matches a known analytic expression. The Conjugate Gradient solver relies on repeated applications
of the Dirac matrix and will therefore produce solutions in disagreement with the analytic result if
the Dirac matrix is incorrectly implemented.

The `benchmark_grid` code automatically performs this correctness check. If the check fails, you
will see a message similar to:

```
Failed to validate free Wilson propagator:
||(result - ref)||/||(result + ref)|| >= 1e-8
```

### Performance results

Performance results can be extracted using the [validate.py](./validate.py) script.
This script ensures all the required data is present in the file and that the MPI
decomposition matches the rules described above.

For example:

```
> ./validate.py
validate.py: test output correctness and extract performance for the UK-NNSS Grid benchmark.
Usage: validate.py <result_json_file>

> ./validate.py benchmark-grid-32-48l.2783375/result.json 

# Grid benchmark validation

               Nodes : 32
           MPI Ranks : 128
   MPI Decomposition : 2.4.4.4

   24^4 DWF4 Performance : 5154 Gflops/s/node
   32^4 DWF4 Performance : 8912 Gflops/s/node
   48^4 DWF4 Performance : 13876 Gflops/s/node

   Validation: PASSED

```

The evaluation sheets ask explicitly for the values reported in the lines ```32^4 DWF4 Performance``` and ```48^4 DWF4 Performance```. All other values serve as sanity check only. 

### Reference data

#### Isambard-AI (GH200)
[IsambardAI](https://docs.isambard.ac.uk/specs/#system-specifications-isambard-ai-phase-2) nodes have 4x NVIDIA GH200 per node and 4x 200 Gbps Slingshot 11 interfaces per node. 

In all cases, 1 MPI process per GPU was used and 72 CPU OpenMP threads per MPI process. Command-line options used were:
`
--no-benchmark-flops-fp64 --no-benchmark-flops-sp4-2as --no-benchmark-flops-sp4-f
  --no-benchmark-flops-su4 --max-L 48
`

| # Nodes | Local Vol. | MPI decomposition | # GPU | Perf. (Gflops/s/node) |
|--:|--:|---|--:|--:|
| 1 |   | `--mpi 1.1.1.4` | 4 | |
|   | 24<sup>4</sup> | | | 22410  |
|   | 32<sup>4</sup> | | | 25656  |
|   | 48<sup>4</sup> | | | 28014  |
| 8 |   | `--mpi 1.2.4.4` | 32 | |
|   | 24<sup>4</sup> | | | 9814  |
|   | 32<sup>4</sup> | | | 13999  |
|   | 48<sup>4</sup> | | | 20757  |
| 16 |  | `--mpi 1.4.4.4` | 64 | |
|   | 24<sup>4</sup> | | | 7633  |
|   | 32<sup>4</sup> | | | 10194  |
|   | 48<sup>4</sup> | | | 16731  |
| 32 |  | `--mpi 2.4.4.4` | 128 | |
|   | 24<sup>4</sup> | | | 5154  |
|   | 32<sup>4</sup> | | | 8912  |
|   | 48<sup>4</sup> | | | 13876  |
| 128 | | `--mpi 4.4.4.8` | 512 | |
|   | 24<sup>4</sup> | | | 3469 |
|   | 32<sup>4</sup> | | | 7293 |
|   | 48<sup>4</sup> | | | 11771 |
| 512 | | `--mpi 4.8.8.8` | 2048 | |
|   | 24<sup>4</sup> | | |  2258 |
|   | 32<sup>4</sup> | | |  6453 |
|   | 48<sup>4</sup> | | |  11383 |

#### Hunter (MI300A)
[Hunter](https://kb.hlrs.de/platforms/index.php/HPE_Hunter_Hardware_and_Architecture) nodes have 4x AMD Mi300A per node and Slingshot interconnect.

Command-line options used were:
`
--no-benchmark-flops-fp64 --no-benchmark-flops-sp4-2as --no-benchmark-flops-sp4-f
  --no-benchmark-flops-su4 --max-L 48
`

| # Nodes | Local Vol. | MPI decomposition | # GPU | Perf. (Gflops/s/node) |
|--:|--:|---|--:|--:|
| 1 |   | `--mpi 1.1.1.1` | 1 | |
|   | 32<sup>4</sup> | | | 5598 |
|   | 48<sup>4</sup> | | | - |
| 1 |   | `--mpi 1.1.1.4` | 4 | |
|   | 32<sup>4</sup> | | | 16767  |
|   | 48<sup>4</sup> | | | 16434  |
| 8 |   | `--mpi 1.2.4.4` | 32 | |
|   | 32<sup>4</sup> | | | 13584  |
|   | 48<sup>4</sup> | | | -  |
| 16 |  | `--mpi 1.4.4.4` | 64 | |
|   | 32<sup>4</sup> | | | 10614  |
|   | 48<sup>4</sup> | | | 15017  |
| 4 |   | `--mpi 1.2.2.1` | CPU | |
|   | 48<sup>4</sup> | | | 15042  |

## License

This benchmark description and any associated files are released under the
MIT license.

## Changelog

The following changes to this document have been made since initial release:

| <div style="width:90px">Date</div> | Change |
|-----------:|--------|
| 2026-05-14 | Update to the Grid repository commit ID to be used |
| 2026-04-29 | Updates to Hunter reference data and additional clarifications on Benchmark_Grid parameters |


