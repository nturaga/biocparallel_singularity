Parallel evaluation using `BiocParallel::BatchtoolsParam` in a singularity container on a high performance compute cluster
==================================================

`#Bioconductor`, `#singularity`, `#HPC`, `#BiocParallel`

Authors: Nitesh Turaga (@nturaga), Ludwig Geistlinger(@lgeistlinger)

Acknowledgements: Sean Davis(@seandavis12), Martin Morgan, Levi Waldron( @LeviWaldron1), and Bioconductor Team (@Bioconductor)

[TOC]

## Bioconductor on Docker

Containers have become very prominent in Bioinformatics in the last 3-4 years for many reasons, most importantly, **reproducibility** and **isolation**. Bioconductor, being the largest platform for open source R packages for genomics has adopted containerization of packages along with the appropriate R version.

* *Reproducibility*: The exact software installed on every container can be reproduced with a **Dockerfile**, and these can be easily distributed. In terms of Bioconductor containers, the version of R and each Bioconductor package installed can be reproduced easily.

* *Isolation*: Software installed within a container will not affect installations or configurations on your host machine, or on any other containers. The version of R and Bioconductor installed on the container can be totally independent of all other versions. This allows easy usage of multiple versions of R and Bioconductor packages.

## Where are these Bioconductor Docker images?

Bioconductor provides Docker images in a couple of ways currently,

**Firstly, Bioconductor docker images** hosted on Docker Hub, located at https://cloud.docker.com/u/bioconductor/repository/list. The website has information and instructions on how to use them https://bioconductor.org/help/docker/. The Dockerfile(s) for each of these containers is located at https://github.com/Bioconductor/bioc_docker/tree/master/out. These are provided directly by the Bioconductor team and are categorized into,

 - **core** (contains core Bioconductor packages),

 - **base** (`BiocManager` comes with it, you have to install all other Bioconductor packages yourself),

 - and a couple of other flavors, **protmetcore** (proteomics and metabolomics) and  **metabolomics** (additional metabolomics packages).

   It is possible to build on top of these default containers for a customized Bioconductor container. A very good example is given at this [link](https://github.com/waldronlab/bioconductor_devel) which provides Bioconductor [release](https://github.com/waldronlab/bioconductor_release) and [devel](https://github.com/waldronlab/bioconductor_devel) images with the complete set of software needed to build and install over 1600 packages in Bioconductor. This is a very good resource for developers/users who are looking to get an image without having to worry about system requirements. It is also very extendable! (visit @lwaldron and @waldronlab on GitHub for updates on these repositories)

**Secondly, [Bioconda](https://bioconda.github.io/)** provides the [**BioContainers**](https://biocontainers.pro/registry/#/). Bioconda provides a container for each individual package in Bioconductor under the BioContainer umbrella in https://quay.io.

 - They also provide additional "Multi package" container, where you can combine multiple bioconda packages into one container. You may choose a Bioconductor package combination of your desire, and build a container with them. (https://biocontainers.pro/registry/#/multipackage)

**Lastly**, another resource, if users want to use both an RStudio server and Jupyter notebooks in the Docker image can be done through a devel version [bioconductor_binder](https://github.com/nturaga/bioconductor_binder)  (or [release](https://github.com/nturaga/bioconductor_binder/tree/R-3.5.2)) which is based on the rocker/binder docker images, but is trimmed down a little. The image allows you to enter the container through a Jupyter notebook, but has all the requirements for Rstudio as well. 

### Extend a Bioconductor Docker image for my purposes

In a new directory (call it "my_docker_image"), open a file called `Dockerfile` with contents to extend your image,

```dockerfile
FROM bioconductor/devel_base2

RUN R -e "BiocManager::install(c(<my_list_of_packages))"
```


## What is a Singularity container and how do I turn my docker image into a Singularity container?

Singularity Hub is an online registry for images. This means that you can connect a GitHub repo containing a build specification file to this website, and the image is going to build for you automatically, and be available programmatically (similar to Docker hub).

Remember, Singularity Hub needs authorization to access your github repository, where the Dockerfile is located.

Add a Singularity file to your github repo which corresponds to the `Dockerfile` to tell `shub` how to build the image see https://singularity.lbl.gov/docs-docker for instructions on how to specify this file.

The Singularity file does not need to be more than a simple:

	Bootstrap: docker
	From: <repo>/<image_name>

Your singularity container is then automatically built under https://www.singularity-hub.org/collections/. For an example of how this is done, take a look at https://github.com/waldronlab/bioconductor_devel, where the `Singularity` file is defined as

```dockerfile
Bootstrap: docker
From: waldronlab/bioconductor_devel
```


## How do I get singularity on a cluster?

https://singularity.lbl.gov/docs-installation. Most clusters come with an installation of Singularity (contact your IT department in doubt).

Depending on whether your cluster installation / configuration incorporates environment modules http://modules.sourceforge.net/, you might need to load Singularity first,

	module load singularity

To inspect available modules, use

	module available


## How do I use a singularity image on a cluster?

Assuming your singularity container is stored on Singularity Hub (see instructions above for transforming a docker container into a singularity container), you can obtain and create an image of the
container on your local host via

	singularity run shub://<username>/bioconductor_devel

This creates an image named `<username>-bioconductor_devel-master-latest.simg` and you can shell
into the image via

	singularity shell <username>-bioconductor_devel-master-latest.simg

It is also possible to directly append shell commands, such as starting the containerized R via

	singularity shell <username>-bioconductor_devel-master-latest.simg R

As a side note:

	Setting
	
	alias singulaR="singularity shell <username>-bioconductor_devel-master-latest.simg R”

in your .bashrc let’s you eg build your R package via

```shell
singulaR CMD build <mypackage>
```


## How can I use additional R/Bioconductor packages in my singularity image?

It is possible to add all required packages upon construction of the container and then use the static image for all further computation. However, in some cases you might want to dynamically install additional packages in your containerized R/Bioconductor installation.

This can be achieved by mounting a host directory to the container that contains your personal library of R/Bioconductor packages.

https://singularity.lbl.gov/docs-mount#user-defined-bind-points

In the default configuration, the directories `$HOME`,` /tmp`,` /proc`, `/sys`, and `/dev` are among the system-defined bind points.

That means that you can for example inside your .Rprofile

```R
home <- Sys.getenv("HOME")
mylib <- file.path(home, “mylib”)
.libPaths(mylib)
```

Installation of additional packages can then be achieved from within your container via:

```R
BiocManager::install(“mypackage”)
```


## How to ensure that the singularity container can be invoked on the compute nodes?

It is a good idea to run a number of small tests before executing real jobs and integrate with BiocParallel.

We assume in the following that the cluster runs slurm, but similar instructions apply for other cluster environments.

1. Check first that you can list jobs registered for execution on the cluster via squeue (or alike for other cluster environments).
2. Create a basic test script that we name slurm-test.sh which executes a simple R command via the singularity-wrapped R on a compute node of the cluster

```bash
#!/bin/bash

#SBATCH -p <queue>
#SBATCH --job-name=<jobname>
#SBATCH --output=<outfile>
#SBATCH --error=<errfile>
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1

# load singularity
module load singularity

# execute a simple R command
singularity shell <path-to-image>/<username>-bioconductor_devel-master-latest.simg R -e "cat('Hello World')"
```

Note that expressions in angle brackets have to be replaced accordingly.  And then submit this simple script to the cluster via

```shell
sbatch slurm-test.sh
```

Make sure that an output file named as specified in the script is created and that it contains as expected the R startup message and the "Hello World” message.


## How do I invoke the singularity container through BiocParallel to execute jobs in parallel on a high performance computer cluster?

To use the containerized R/Bioconductor in conjunction with `BiocParallel`, a recent R installation on the cluster is needed that, as a minimum requirement, has also the packages BiocParallel and batchtools installed.

This basic R installation is interactively used on the head node to submit jobs via BiocParallel to the compute nodes. The jobs are then executed using the containerized R/Bioconductor on the compute nodes.

Note that submission of jobs directly from within the singularity container on the head node is not possible, as the container doesn’t recognize the cluster environment and its specific configuration. Note also that binding host paths to the container for that purpose is likely problematic due to conflicting libraries.

BiocParallel provides functionality for computation on batch systems with `BatchtoolsParam`.


A template file is needed that defines how jobs are executed on the compute nodes.  For a SLURM cluster, a simple template file named slurm-simple.tmpl would look like this:

```bash
#!/bin/bash

<%
# relative paths are not handled well by Slurm
log.file = fs::path_expand(log.file)
-%>

#SBATCH -p <queue>
#SBATCH --job-name=<%= job.name %>
#SBATCH --output=<%= log.file %>
#SBATCH --error=<%= log.file %>

module load singularity

singularity shell <path-to-image>/<username>-bioconductor_devel-master-latest.simg R -e 'batchtools::doJobCollection("<%= uri %>")'
```

Note that expressions in angle brackets have to be replaced accordingly.

We use the template file to configure a `BatchtoolsParam` in the cluster’s basic R installation running on the head node:

```R
library(BiocParallel)
library(batchtools)
param <- BatchtoolsParam(
    workers = 2, 
    cluster = "slurm",
	template="slurm-simple.tmpl"
)

## and supply the param to bplapply for parallel execution
## of a simple function that returns the hostname of the
## compute nodes for testing purpose.

test_cluster <- function(n) system2(“hostname”)
bplapply(1:20, test_cluster, BPPARAM = param)
```

Session Info

```R
sessionInfo()
```
