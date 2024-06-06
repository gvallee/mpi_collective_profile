# mpi_collective_profile

A profiler to extract data from within MPI collectives.

# Overview

The repository is structure to offer support for several MPI collectives, one per sub-directory.
For example, the alltoallv  folder of the repository gathers all the code required to profile MPI alltoallv collective calls from your application. For each type of collective (e.g., alltoallv), a series of shared libraries (.so files) are created, which are a PMPI implementation of the collective. 
The PMPI code intercepts the associated MPI calls, gather some data and execute the actual MPI collective.
This approach allows us to provide a low-overhead solution that can be customized to your profiling needs. 

# Installation

## Requirements

It is adviced to compile the profiling tool with the same compilers than the ones
used to compile the application, including MPI.
To compile the source code, it is therefore required to have an MPI implementation properly installed on the system.

## Compilation

To compile the code, simply execute the following command:
```console
make
```

This will generate a series of shared libraries. For example, for alltoallv, the following shared libraries are created:
- `liballtoallv_counts.so`,
- `liballtoallv_exec_timings.so`,
- `liballtoallv_late_arrival.so`,
- `liballtoallv_backtrace.so`,
- `liballtoallv_location.so`,
- and `alltoallv/liballtoallv.so`.
For most cases, the first 4 libraries are all users need.

## Generated shared libraries

The profiling shared library can gather a lot of data and while it is possible to
gather all the data at once, it is not encouraged to do so. We advice to follow the
following there steps. For convenience, multiple shared libraries are being generated
with the adequate configuration for these steps.
For illustration, we will present some details related to alltoallv but the same approach is implemented for all supported collectives. The following capabilities are supported by our profiler:
- Gather the send/receive counts associated to the alltoallv calls: use the
`liballtoallv_counts.so` library. This generates files based on the following naming
scheme:
`send-counters.job<JOBID>.rank<RANK>.txt` and `recv-counters.job<JOBID>.rank<RANK>.txt`;
where:
    - `JOBID` is the job number when using a job scheduler (e.g., SLURM) or `0` when no
job scheduler is used or the value assigned to the `SLURM_JOB_ID` environment variable
users decide to set (it is strongly advised to set the `SLURM_JOB_ID` environment variable
to specify a unique identifier when not using a job scheduler).
    - `RANK` is the rank number on `MPI_COMM_WORLD` that is rank 0 on the communicator used
for the alltoallv operations. Note that this means that if the
application is executing alltoallv operations on different communicators, the tool
generates multiple counter files, one per communicator. If the application is only using
`MPI_COMM_WORLD`, the two output files are named `send-counters.job<JOBID>.rank0.txt` and
`recv-counters.job<JOBID>.rank0.txt`.
Using these two identifiers makes it easier to handle multiple traces from multiple
applications and/or platforms.
- Gather timings: use the `liballtoallv_exec_timings.so` and `liballtoallv_late_arrival.so` shared libraries. These generate
by default multiple files based on the following naming scheme:
 `<COLLECTIVE>_late_arrivals_timings.rank<RANK>_comm<COMMID>_job<JOBID>.md` and `<COLLECTIVE>_execution_times.rank<RANK>_comm<COMMID>_job<JOBID>.md`.
- Gather backtraces: use the `liballtoallv_backtrace.so` shared library. This generates
files `backtrace_rank<RANK>_call<ID>.md`, *one per alltoallv call*, all of them stored in a `backtraces`
directory. In other words, this generates one file per alltoallv call, where `<ID>` is the
alltoallv call number on the communicator (starting at 0).
- Gather location: use the `liballtoallv_location.so` shared library. This generates files
`location_rank<RANK>_call<ID>.md`, *one per alltoallv call*. In other words, this generates
one file per alltoallv call, where `<ID>` is the alltoallv call number on the communicator
(starting at 0).

## Execution

Before running the application to get traces, users have the option to customize the
tool behavior, mainly setting the place where the output files are stored (if not specified,
the current directory) by using the `MPI_COLLECTIVE_PROFILER_OUTPUT_DIR` environment variable.

Like any PMPI option, users need to use `LD_PRELOAD` while executing their application.

On a platform where `mpirun` is directly used, the command to start the application
and profile alltoallv collectives looks like:
```console
LD_PRELOAD=$HOME<path_to_repo>/src/alltoallv/liballtoallv.so mpirun --oversubscribe -np 3 app.exe
```

### Example with no job manager is used

On a platform where a job manager is used, such as Slurm, users need to update the
batch script used to submit an application run. For instance, with Open MPI and Slurm,
it would look like:
```console
mpirun -np $NPROC -x LD_PRELOAD=<path_to_repo>/src/alltoallv/liballtoallv_counts.so app.exe
```

When using a job scheduler, users are required to correctly set the LD_PRELOAD details
in their scripts or command line.

### Example with Slurm

Assuming Slurm is used to execute jobs on the target platform, the following is an example of
a Slurm batch script that runs the OSU microbenchmakrs and gathers all the profiling tracesi for alltoallv:

```
#!/bin/bash
#SBATCH -p cluster
#SBATCH -N 32
#SBATCH -t 05:00:00
#SBATCH -e alltoallv-32nodes-1024pe-%j.err
#SBATCH -o alltoallv-32nodes-1024pe-%j.out

set -x

module purge
module load gcc/4.8.5 ompi/4.0.1

export A2A_PROFILING_OUTPUT_DIR=/shared/data/profiling/osu/alltoallv/traces1

COUNTSFLAGS="/path/to/profiler/code/alltoall_profiling/alltoallv/liballtoallv_counts.so"
MAPFLAGS="/path/to/profiler/code/alltoall_profiling/alltoallv/liballtoallv_location.so"
BACKTRACEFLAGS="/path/to/profiler/code/alltoall_profiling/alltoallv/liballtoallv_backtrace.so"
A2ATIMINGFLAGS="/path/to/profiler/code/alltoall_profiling/alltoallv/liballtoallv_exec_timings.so"
LATETIMINGFLAGS="/path/to/profiler/code/alltoall_profiling/alltoallv/liballtoallv_late_arrival.so"

MPIFLAGS="--mca pml ucx -x UCX_NET_DEVICES=mlx5_0:1 "
MPIFLAGS+="-x A2A_PROFILING_OUTPUT_DIR "
MPIFLAGS+="-x LD_LIBRARY_PATH "

mpirun -np 1024 -map-by ppr:32:node -bind-to core $MPIFLAGS -x LD_PRELOAD="$COUNTSFLAGS" /path/to/osu/install/osu-5.6.3/libexec/osu-micro-benchmarks/mpi/collective/osu_alltoallv -f
mpirun -np 1024 -map-by ppr:32:node -bind-to core $MPIFLAGS -x LD_PRELOAD="$MAPFLAGS" /path/to/osu/install/osu-5.6.3/libexec/osu-micro-benchmarks/mpi/collective/osu_alltoallv -f
mpirun -np 1024 -map-by ppr:32:node -bind-to core $MPIFLAGS -x LD_PRELOAD="$BACKTRACEFLAGS" /path/to/osu/install/osu-5.6.3/libexec/osu-micro-benchmarks/mpi/collective/osu_alltoallv -f
mpirun -np 1024 -map-by ppr:32:node -bind-to core $MPIFLAGS -x LD_PRELOAD="$A2ATIMINGFLAGS" /path/to/osu/install/osu-5.6.3/libexec/osu-micro-benchmarks/mpi/collective/osu_alltoallv -f
mpirun -np 1024 -map-by ppr:32:node -bind-to core $MPIFLAGS -x LD_PRELOAD="$LATETIMINGFLAGS" /path/to/osu/install/osu-5.6.3/libexec/osu-micro-benchmarks/mpi/collective/osu_alltoallv -f
```

## Generated data

In this paragraph, we use alltoallv to properly illustrate what data is generated but other MPI collectives have similar data:
When using the default shared libraries, the following files are generated which are details further below:
- files with a name starting with `send-counters` and `recv-counters`; files providing the alltoallv counts as defined by the MPI standard,
- files prefixed with `alltoallv_locations`, which stores data about the location of the ranks involved in alltoallv operations,
- files prefixed with `alltoallv_late_arrival`, which stores time data about ranks arrival into the alltoallv operations,
- files prefixed with `alltoallv_execution_times`, which stores the time each rank spent in the alltoallv operations,
- files prefixed with `alltoallv_backtrace`, which stores information about the context in which the application is invoking alltoallv.

In other to compress data and control the size of the generated dataset, the tool is able to use a compact notation to avoid duplication in lists. This notation is mainly applied to list of ranks. The format is a comma-separated list where consecutive numbers are saved as a range. For example, ranks `1, 3` means ranks 1 and 3; ranks `2-5` means ranks 2, 3, 4, and 5; and ranks `1, 3-5` means ranks 1, 3, 4, and 5.

### Send and receive count files

A `send-counters` and `recv-counters` files is generated per communicator used to perform an alltoallv operations. In other words, if alltoallv operations are executed on a single communicator, only two files are generated: `send-counters.job<JOBID>.rank<LEADRANK>.txt` and `recv-counters.job<JOBID>.rank<LEADRANK>.txt`, where `JOBID` is the job number when a job manager such as Slurm is used (equal to 0 when no job manager is used) and `LEADRANK` is the rank on `MPI_COMM_WORLD` that is rank 0 on the communicator used. `LEADRANK` is therefore used to differantiate data from different sub-communicators.

The content of the count files is predictable and organized as follow:
- `# Raw counters` indicates a new set of counts and is always followed by an empty line.
- `Number of ranks:` indicates how many ranks were involved in the alltoallv operations.
- `Datatype size:` indicates the size of the datatype used during the operation. Note that at the moment, the size is saved only in the context of the lead rank (as previously defined); alltoallv communications involving different datatype sizes is currently not supported.
- `Alltoallv calls:` indicates how many alltoallv calls *in total* (not specifically for the current set of counts) are captured in the file.
- `Count:` indicates how many alltoallv calls have the counts reported below. This line gives the total number of all calls as well as the list of all the calls using our compact notation.
- And finally the raw counts which are delimited by `BEGINNING DATA` and `END DATA`. Each line of the raw counts represents the count for ranks. Please refer to the MPI standard to fully understand the semantic of counts. `Rank(s) 0, 2: 1 2 3 4` means that ranks 0 and 2 have the following counts: 1 for rank 0, 2 for rank 1, 3 for rank 2 and 4 for rank 3.

### Time file: alltoallv_late_arrival* and alltoallv_execution_times* files

The first line is the version of the data format. This is used for internal purposes to ensure that the post-mortem analysis tool supports that format.

Then the file has a series of timing data per call. Each call data starts with `# Call` with the number of the call following by the ordered list of timing data per rank.

All timings are in seconds.

The late arrival timings are gathered by artificially adding a `MPI_Barrier` operation on the communicator right before starting the `MPI_Alltoallv` operations, using PMPI. The late arrival time is the time spent in the barrier. The longer the time, the earlier the rank arrived; the shorter the time, the later the rank arrived. The reported times cannot therefore be extrapolated to any real application execution time since this is done for every single alltoallv operation. However, it gives a general idea of the imbalance between ranks when initiating a new alltoallv operation,, especially since most systems do not have a fine-grain clock synchronization capability. As a result, this method is a compromise between accuracy and complexity of the implementation. We performed a study about the accuracy of the measurements between consecutive application profiles and we discovered some degree of variability, which are difficult to address without introducing complex mechanisms. So raw timings should not be used to draw any conclusions, only trends should be used. Hardware support would be a useful feature to improve measurement accuracy and report data that could be useful to analyse in the context of the application's execution without profiling.

It is possible to have a rough validation of the late arrival timing mechanism by using the `COLLECTIVE_PROFILER_INJECT_DELAY` environment variable, which triggers the injection of 1 second delay on rank 0.
The assumptions are:
- the profiler is fully functional,
- you understand the behavior of the alltoallv application you will be using with this feature, otherwise, it is not possible
to analyse the results since it would not be possible to know what to expect.
For example, by using a tightly coupled alltoallv code, using 8 ranks and setting the environment variables, the following late arrival result is generated:
```
FORMAT_VERSION: 9

# Call 0
0.000011
1.000095
1.000084
1.000102
1.000086
1.000097
1.000083
1.000105
```

This result is expected: all ranks except rank 0 are spending roughly 1 second in the barrier that is included to calculate late arrivals. In other words, rank 0 arrives roughly 1 second after all other ranks.

### Location files

The first line is the version of the data format. This is used for internal purposes to ensure that the post-mortem analysis tool supports that format.

Then the files has a series of entries, one per unique location where a location is the rank on the communicator and the host name. An example of such a location is:
```
Hostnames:
    Rank 0: node1
    Rank 1: node2
    Rank 2: node3
```
In order to control the size of the dataset, the metadata for each unique location includes: the communicator identifier (`Communicator ID:`), the list of calls having the unique location (`Calls:`), the ranks on MPI_COMM_WORLD (`COMM_WORLD_ rank:`) and PIDs (`PIDs`).

### Trace files

The first line is the version of the data format. This is used for internal purposes to ensure that the post-mortem analysis tool supports that format.

After the format version, a line prefixed with `stack trace for` indicates the binary associated to the trace. In most cases, only one binary will be reported.

Then the files has a series of entries, one per unique backtrace where a backtrace the data returned by the backtrace system call. An example of such a backtrace is:
```
/home/user1/collective_profiler/src/alltoallv/liballtoallv_backtrace.so(_mpi_alltoallv+0xd4) [0x147c58511fa8]
/home/user1/collective_profiler/src/alltoallv/liballtoallv_backtrace.so(MPI_Alltoallv+0x7d) [0x147c5851240c]
./wrf.exe_i202h270vx2() [0x32fec53]
./wrf.exe_i202h270vx2() [0x866604]
./wrf.exe_i202h270vx2() [0x1a5fd30]
./wrf.exe_i202h270vx2() [0x148ad35]
./wrf.exe_i202h270vx2() [0x5776ba]
./wrf.exe_i202h270vx2() [0x41b031]
./wrf.exe_i202h270vx2() [0x41afe6]
./wrf.exe_i202h270vx2() [0x41af62]
/lib64/libc.so.6(__libc_start_main+0xf3) [0x147c552d57b3]
./wrf.exe_i202h270vx2() [0x41ae69]
```

Finally each unique trace is accociated to 1 or more context(s) (`# Context` followed by the context number, i.e., the number in which it has been detected). A context is composed of a communicator (`Communicator`), the rank on the communicator (`Communicator rank`) which in most cases is `0` because the lead on the communicator, the rank on MPI_COMM_WORLD (`COMM_WORLD rank`), and finally the list of alltoallv calls having the backtrace using the compact notation previously presented.
