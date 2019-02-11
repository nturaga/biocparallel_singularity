Using `BiocParallel::BatchtoolsParam` with a containerized version of Bioconductor for parallel computation on a high performance computer cluster
================

Authors: Nitesh Turaga, Ludwig Geistlinger

Acknowledgements: Sean Davis, Martin Morgan, Levi Waldron

## Why use Bioconductor on Docker?

Containers have become very prominent in Bioinformatics in the last
3-4 years for many reasons, most importantly, **reproducibility** and
**isolation**. Bioconductor, being the largest platform for open
source R packages for genomics has adopted containerization of
packages along with the appropriate R version.

Reproducibility: The exact software installed on every container can
be reproduced with a Dockerfile, which can be easily distributed. In
terms of Bioconductor containers, the version of R and each
Bioconductor package installed can be reproduced easily.

Isolation: Software installed within a container will not affect
installations or configurations on your host machine, or on any other
containers. The version of R and Bioconductor installed on the
container can be totally independent of all other versions. This
allows easy usage of multiple versions of R and Bioconductor packages.

## Where to get Bioconductor Docker images?

Bioconductor has docker images hosted on Docker Hub, located at https://cloud.docker.com/u/bioconductor/repository/list.

The location on the website for background and instructions on how to
use them is located on the Bioconductor website
https://bioconductor.org/help/docker/.

The Dockerfile(s) for each of the Container repo:
https://github.com/Bioconductor/bioc_docker

It is possible to build on top of these default containers for a
customized Bioconductor container as e.g. described here:
https://github.com/waldronlab/bioconductor_devel.

Let’s assume in the following, we have a github repository named
bioconductor_devel that contains as a minimum requirement a Dockerfile
with the following instruction line

FROM bioconductor/devel_base2


## What is a Singularity container and how do I turn my docker image into a Singularity container?

Singularity Hub is an online registry for images. This means that you
can connect a GitHub repo containing a build specification file to
this website, and the image is going to build for you automatically,
and be available programmatically.

Singularity Hub needs authorization to access your github repo

Add a Singularity file to your github repo which corresponds to the
Dockerfile to tell shub how to build the image see
https://singularity.lbl.gov/docs-docker for instructions on how to
specify this file

The Singularity file does not need to be more than:

	Bootstrap: docker
	From: <username>/bioconductor_devel

Your singularity container is then automatically built under
https://www.singularity-hub.org/collections/


## How do I get singularity on a cluster?

https://singularity.lbl.gov/docs-installation

Most clusters come with an installation of Singularity (contact your
IT department in doubt).

Depending on whether your cluster installation / configuration
incorporates environment modules http://modules.sourceforge.net/, you
might need to load Singularity first via

	module load singularity

To inspect available modules, use

	module available


## How do I use a singularity image on a cluster?

Assuming your singularity container is stored on Singularity Hub (see
instructions above for transforming a docker container into a
singularity container), you can obtain and create an image of the
container on your local host via

	singularity run shub://<username>/bioconductor_devel

This creates an image named
<username>-bioconductor_devel-master-latest.simg and you can shell
into the image via

	singularity shell <username>-bioconductor_devel-master-latest.simg

It is also possible to directly append shell commands, such as
starting the containerized R via

	singularity shell <username>-bioconductor_devel-master-latest.simg R

As a side note:

	Setting

	alias singulaR="singularity shell <username>-bioconductor_devel-master-latest.simg R”

in your .bashrc let’s you eg build your R package via

	singulaR CMD build <mypackage>


## How can I use additional R/Bioconductor packages in my singularity image?

It is possible to add all required packages upon construction of the
container and then use the static image for all further computation.

However, in some cases you might want to dynamically install
additional packages in your containerized R/Bioconductor installation.

This can be achieved by mounting a host directory to the container
that contains your personal library of R/Bioconductor packages.

https://singularity.lbl.gov/docs-mount#user-defined-bind-points

In the default configuration, the directories $HOME, /tmp, /proc,
/sys, and /dev are among the system-defined bind points.

That means that you can for example inside your .Rprofile

	home <- Sys.getenv("HOME")
	mylib <- file.path(home, “mylib”)
	.libPaths(mylib)

Installation of additional packages can then be achieved from within your container via:

	BiocManager::install(“mypackage”)


## How to ensure that the singularity container can be invoked on the compute nodes?

It is a good idea to run a number of small tests before executing real
jobs and integrate with BiocParallel.

We assume in the following that the cluster runs slurm, but similar
instructions apply for other cluster environments.

1. Check first that you can list jobs registered for execution on the
   cluster via squeue (or alike for other cluster environments).

2. Create a basic test script that we name slurm-test.sh which
   executes a simple R command via the singularity-wrapped R on a
   compute node of the cluster

```
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


Note that expressions in angle brackets have to be replaced
accordingly.  And then submit this simple script to the cluster via

	sbatch slurm-test.sh



Make sure that an output file named as specified in the script is
created and that it contains as expected the R startup message and the
“Hello World” message.


## How do I invoke the singularity container through BiocParallel to execute jobs in parallel on a high performance computer cluster?

To use the containerized R/Bioconductor in conjunction with
BiocParallel, a recent R installation on the cluster is needed that,
as a minimum requirement, has also the packages BiocParallel and
batchtools installed.

This basic R installation is interactively used on the head node to
submit jobs via BiocParallel to the compute nodes. The jobs are then
executed using the containerized R/Bioconductor on the compute nodes.

Note that submission of jobs directly from within the singularity
container on the head node is not possible, as the container doesn’t
recognize the cluster environment and its specific configuration. Note
also that binding host paths to the container for that purpose is
likely problematic due to conflicting libraries.

BiocParallel provides functionality for computation on batch systems
with `BatchtoolsParam`.


A template file is needed that defines how jobs are executed on the
compute nodes.  For a SLURM cluster, a simple template file named
slurm-simple.tmpl would look like this:

```
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

Note that expressions in angle brackets have to be replaced
accordingly.

We use the template file to configure a BatchtoolsParam in the
cluster’s basic R installation running on the head node:

```
library(BiocParallel)
library(batchtools)
param <- BatchtoolsParam(workers = 2, cluster = "slurm" ,
								template="slurm-simple.tmpl")

and supply the param to bplapply for parallel execution of a simple function that returns the hostname of the compute nodes for testing purpose.

test_cluster <- function(n) system2(“hostname”)
bplapply(1:20, test_cluster, BPPARAM = param)
```

Session Info

```
sessionInfo()
```
