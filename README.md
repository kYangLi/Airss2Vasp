# airss4vasp (a4v)

## Basic Info

`Author` liyang@cmt.tsinghua

`Start Date` 2019.9.21

`Last Update` 2020.12.4

`Version` 1.1.0

## Description

[`AIRSS`](https://www.mtg.msm.cam.ac.uk/Codes/AIRSS)(Ab Initio Random Structure Searching) is a fantastic, efficient, and easy to parallel structure searching package, but with pool supporting for `VASP`. `airss4vasp`(a4v), based on `PBS` ,`Slurm`,or `NSCC(Tianhe)` job management system, is an interface program design for the better communication between AIRSS and VASP.

Visit [https://www.bilibili.com/video/BV12v411r7i9](https://www.bilibili.com/video/BV12v411r7i9) to watch the tutorial video!

## Installtion

Before the `make`, please check the `Makefile` to modify some setting for libs. There also are some tips for the installtion inside the `Makefile`.

```bash
vim Makefile # Makefile.GNU & Makefile.INTEL are also ready to use
```

To compile and install `a4v`, excute the following command in the main folder.

```bash
make all
```

The last step that MUST be done is adding the `bin` folder of `a4v` to the env `PATH`. Read the tail output of `make all` for more details.

```bash
echo "export PATH=<a4v>/<bin>/<path>/:${PATH}" >> ~/.bashrc
```

## Input File
  
To enable this script, you need perpare the following files:

- `a4v.input` (Optional)
- `<seedname>.cell`
- `<seedname>.INCAR-[1-9]`
- `<seedname>.KPOINTS-[1-9]` (Optional)
- `<seedname>.POTCAR`

### `a4v.input`

Here is the list of the parameters in `a4v.input` you can modify:

Parameter Name | Type | Descripution 
:-|:-|:-
SEED_NAME         | Char  | AIRSS seed name.
TASK_NAME         | Char  | A4V task name.
INTEL_MODULE      | Char  | Intel Module load command, if you are using Intel lib for VASP.
VASP_PROG         | Char  | Path of VASP executive program.
NODES_NUM         | Int   | Total Nodes number used in task.
CORES_PER_NODE    | Int   | The number of cores of each nodes in your machine(or you want to use).
CORES_PER_COACH   | Int   | The number of cores used by each coach.
STR_NUM           | Int   | AIRSS random structure number. 
IS_2D_MATERIAL    | T/F   | Whether the structure is 2D system, only effective when generate KPOINTS using `genkp`.
KP_SEP_LIST       | List  | Kpoints sepration list, only effective when generate KPOINTS using `genkp`.
SYMM_PREC         | Float | Symmetry precise used in `cellsym`.
SYS_TYPE          | Char  | Job system type, choice one from [pbs, nscc, slurm, direct]. The `nscc` is job manage system for TianHe, and the `direct` means run vasp on current node directly.
PBS_WALLTIME      | Int   | PBS walltime in hours, only effective when using PBS job system.
JOB_QUEUE         | Char  | PBS queue, only effective when using PBS job system. Use 'unset-queue' to comment it out.
VASP_WALLTIME     | Int   | Walltime for a single VASP relazation(one INCAR step) in seconds.
KEEP_CALC_DETAILS | T/F   | Whether keep all VASP calculation details or not.

Here is a example of `a4v.input`:
```bash
SEED_NAME         = Si
TASK_NAME         = a4v-Si2
INTEL_MODULE      = source /intel/cl2020/linux/bin/compilervars.sh intel64 
VASP_PROG         = /bin/vasp_ncl
NODES_NUM         = 4
CORES_PER_NODE    = 40
CORES_PER_COACH   = 20
STR_NUM           = 100
IS_2D_MATERIAL    = F
KP_SEP_LIST       = 0.2,0.1
SYMM_PREC         = 0.05
SYS_TYPE          = pbs
PBS_WALLTIME      = 96
JOB_QUEUE         = cmt
VASP_WALLTIME     = 3600
KEEP_CALC_DETAILS = F

```

### `<seedname>.cell`

This is the key file for the whole structure searching task. Please first learn how to use `AIRSS` before using `a4v`. You could reference the AIRSS manual writen by me in ./doc/AIRSS-0.9.1_manual/*/main.pdf.

There is one thing need to be explained more specifically.

During the generation of the new random structures in `AIRSS`, basically, there are two steps for the movement of a single atom: `random shift` and `push`. The step `push` is applied to make sure two atoms are not connected to close.

In the `<seedname>.cell`, there are two atomic tags called `NOMOVE` and `FIX`. The first one designed for disable the `PUSH` step, while the second for disbale the `PUSH` **and** fix the atom during the `CASTEP` relaxzation.

Since now we are using `VASP` in `a4v`, `FIX` and `NOMOVE` actually have the same effect. If you mean to fix an atom during relaxztion, use the tag `SD-*` (where the `*` can replace with `X`, `Y`, `Z`, `XY`, `YZ`, `ZX`, `XYZ`). This tag will enable the `Selective dynamics` mode in `POSCAR` (in `X`, `XY`, `XYZ`... directions).

The original keyword `#CLUSTER` and `#SPIN=` is not supported in `a4v`, for a4v is designed only for crystal structure searching maybe with a fix collinear magnetic order (`VASP` is not that 
effective when calculating the cluster, and the random spin number generated by `AIRSS` sometimes can cause the non-converge during the magnetic calculation). 

Here is an example of the `Si2.cell` file.

```bash
%BLOCK LATTICE_CART
 0.0    2.75    2.75
 2.75   0.0     2.75
 2.75   2.75    0.0
#FIX
%ENDBLOCK LATTICE_CART

%BLOCK POSITIONS_FRAC
Si 0.0 0.0 0.0 # Si1 % NUM=1 POSAMP=0 NOMOVE SD-XYZ
Si 0.0 0.0 0.0 # Si2 % NUM=1
%ENDBLOCK POSITIONS_FRAC

#MINSEP=2.0
```

### `<seedname>.INCAR-[1-9]`

The `INCAR` is a input file for VASP relaxzation. The quantity of `INCAR` decided how many times will the structure be relaxed.

E.g. If there are `Si.INCAR-1`, `Si.INCAR-2`, `Si.INCAR-3` in the calculation file, then the same `Si` structure will first be relaxed using `INCAR-1`, then `INCAR-2`, and at last `INCAR-3`. You can also indicate the KPOINTS for each INCAR with name `Si.KPOINTS-[1-9]`

### `<seedname>.POTCAR`

The order of the elements in `POTCAR` must agree with that in `<seedname>.cell`.

## Start the Search

### Submit task

Afte the input files getting ready, type the following command to submit the job.

```bash
a4v
```

### COACH

The `COACH` is a parallel unit among the whole task. Each `COACH` will run independently. They will pick up the structure from the `POSCAR-POOL`, push the result to the `RES-POOL`. The POSCAR that already be calculated will be marked in the `TRAIN.record` file.

The total `COACH` number is simply calculated as **nodes_num(cores_per_node/cores_per_coach)**, where cores_per_node/cores_per_coach = n or 1/n. 

### Process check

During the calculation, after enter the main calulation folder(the folder that has `a4v.input`), you can use the command below to check the current processing.

```bash
a4v-prg
```

### Result output

After or during the searching process, you can enter the `RES-POOL` folder and use `a4v-res` to check the result. You may need to learn how to use the `cryan` in `AIRSS` first.

```bash
a4v-res -r -u 0.01 -t 5
```

### Kill the job

```bash
./_KILLJOB.sh
```

### Clean the foder to initial

```bash
./_CLEAN.sh
```
### Show job excution (in `direct` mode only)

```bash
./_SHOWJOB.sh
```