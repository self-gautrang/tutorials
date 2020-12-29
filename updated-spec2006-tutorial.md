---
Authors:
  - Hoa Nguyen
---

# Tutorial: Run SPEC CPU 2006 Benchmarks in Full System Mode with gem5art

## Introduction
In this tutorial, we will demonstrate how to utilize [gem5art](https://github.com/darchr/gem5art) and [gem5-resources](https://gem5.googlesource.com/public/gem5-resources/) to run [SPEC CPU 2006 benchmarks](https://www.spec.org/cpu2006/) in gem5 full system mode. 
The scripts in this tutorial work with gem5art v1.3.0, gem5 20.1.0.2, and gem5-resources 20.1.0.2.

### SPEC CPU 2006 Benchmarks
**Note:** The SPEC CPU 2006 benchmark suite [has been retired](https://www.spec.org/cpu2006/).
Please refer to [this link](#TODO) if you are looking for the tutorial of running SPEC CPU 2017 Benchmarks with gem5 full system mode.

### gem5-resources
[gem5-resources](https://gem5.googlesource.com/public/gem5-resources/) is an actively maintained collections of gem5-related resources that are commonly used.
The resources include scripts, binaries and disk images for full system simulation of many commonly used benchmarks.
This tutorial will offer guidance in utilizing gem5-resources for full system simulation.


### gem5 Full System Mode
Different from gem5 SE mode (syscall emulation mode), the FS mode (full system mode) uses an actual Linux kernel binary instead of emulating the responsibilities of a typical modern OS such as managing page tables and taking care of system calls.
As a result, gem5 FS simulation would be more realistic compared to gem5 SE simulation, especially when the interactions between the workload and the OS are significant part of the simulation.

A typical gem5 full system simulation requires a compiled Linux kernel, a disk image containing compiled benchmarks, and gem5 system configurations.
gem5-resources typically provides all required all of the mentioned resources for every supported benchmark such that one could download the resources and run the experiment without much modification.
However, due to license issue, gem5-resources does not provide a disk image containing SPEC CPU 2006 benchmarks.
In this tutorial, we will provide a set of scripts that generates a disk image containing the benchmarks assuming the ISO file of the SPEC CPU 2006 benchmarks is available.

### Outline of the Experiment
TODO

### An Overview of Host System - gem5 Interactions
![**Figure 1.**]( ../images/spec_tutorial_figure1.png "")
A visual depict of how gem5 interacts with the host system.
gem5 is configured to do the following: booting the Linux kernel, running the benchmark, and copying the SPEC outputs to the host system.
However, since we are interested in getting the stats only for the benchmark, we will configure gem5 to exit after the kernel is booted, and then we reset the stats before running the benchmark.
We use KVM CPU model in gem5 for Linux booting process to quickly boot the system, and after the process is complete, we switch to the desired detailed CPU to run the benchmark.
Similarly, after the benchmark is complete, gem5 exits to host, which allows us to get the stats at that point.
After that, optionally, we switch the CPU back to KVM, which allows us to quickly write the SPEC output files to the host.

**Note:** gem5 will output the stats again when the gem5 run is complete.
Therefore, we will see two sets of stats in one file in stats.txt.
The stats of the benchmark is the the first part of stats.txt, while the second part of the file contains the stats of the benchmark AND the process of writing output files back to the host.
We are only interested in the first part of stats.txt.

## Setting up the Experiment
In this part, we have two concurrent tasks: setting up the resources and documenting the process using gem5art.
We will structure the [SPEC 2006 resources as laid out by gem5-resources](https://gem5.googlesource.com/public/gem5-resources/+/refs/heads/stable/src/spec-2006/).
The script `launch_spec2006_experiment.py` will contain the documentation about the artifacts we create and will also serve as Python script that launches the experiment.

### Acquiring gem5-resources and Setting up the Experiment Folder
First, we clone the gem5-resource repo and check out the stable branch upto the `cee972a1727abd80924dad73d9f3b5cf0f13012d` commit, which contains the resources working with gem5 20.1.0.2.
```sh
git clone https://gem5.googlesource.com/public/gem5-resources
cd gem5-resources
git checkout cee972a1727abd80924dad73d9f3b5cf0f13012d
```
We set the root folder of the experiment in the `src/spec-2006` folder of the cloned repo.
We will also set up a git structure for the root folder to keep track of the changes in the experiment.
In the `gem5-resources` folder,
```sh
cd src/spec-2006
git init
git remote add origin https://remote-address/spec-expriment.git
```
We document the root folder of the experiment in `launch_spec2006_experiment.py` as follows,
```sh
experiments_repo = Artifact.registerArtifact(
    command = '''
        git clone https://gem5.googlesource.com/public/gem5-resources
        cd gem5-resources
        git checkout cee972a1727abd80924dad73d9f3b5cf0f13012d
        cd src/spec-2006
        git init
        git remote add origin https://remote-address/spec-expriment.git
    ''',
    typ = 'git repo',
    name = 'spec2006 Experiment',
    path =  './',
    cwd = './',
    documentation = '''
        local repo to run spec 2006 experiments with gem5 full system mode;
        resources cloned from https://gem5.googlesource.com/public/gem5-resources upto commit cee972a1727abd80924dad73d9f3b5cf0f13012d of stable branch
    '''
)
```
We use .gitignore file to ingore changes of certain files and folders.
In this experiment, we will use this .gitignore file,
```
*.pyc
m5out
.vscode
results
gem5art-env
disk-image/packer
disk-image/packer_cache
disk-image/spec-2006/spec-2006-image/spec-2006
disk-image/spec-2006/CPU2006v1.0.1.iso
gem5
vmlinux-4.19.83
```
In the script above, we ignore files and folders that we use other gem5art Artifact objects to keep track of them, or the presence of those files and folders do not affect the experiment's results.
For example, `disk-image/packer` is the path to the packer binary which generates the disk image, and newer versions `packer` probably won't affect the content of the disk image.
Another example is that we use another gem5art Artifact object to keep track of `vmlinux-4.19.83`, so we put the name of the file in the .gitignore file.

**Note:** You probably notice that there are more than one way of keeping track of the files in the experiment folder: either the git structure of the experiment will keep track of a file, or we can create a separate gem5art Artifact object to keep track of that file.
The decision of letting the git structure or creating a new Artifact object leads to different outcomes.
The difference lies on the type of the Artifact object (specified by the `typ` parameter): for Artifact objects that has `typ` of `git repo`, gem5art won't upload the files in the git structure to gem5art's database, instead, it will only keep track of the hash of the HEAD commit of the git structure.
However, for Artifact's that do **not** have `typ` that is `git repo`, the file specfied in the `path` parameter will be uploaded to the database.

Essentially, we tend to keep small-size files (such as scripts and texts) in a git structure, and to keep large-size files (such as gem5 binaries and disk images) in Artifact's of type `gem5 binary` or `binary`.
Another important difference is that gem5art does **not** keep track of files in a git Artifact, while it does upload other types of Artifact to its database.





