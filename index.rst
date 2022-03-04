:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is not yet published.**

Introduction
============

The LSST Science Pipelines "stack" is a large software system composed of dozens of packages.
The build system, based on the Jenkins tool, tests this system, certifies releases, and creates and publishes binary artifacts.
This document describes the current state of the system, exposing some of its complexities.

Functions
=========

The build system is responsible for several functions, each of which is represented by a pipeline within Jenkins:

 * Manually-triggered integration testing ("stack-os-matrix")
 * Nightly and weekly automated releases ("nightly-release" and "weekly-release")
 * Nightly "clean build" tests, some with extended testing ("ci_hsc", "ci_imsim", and "lsst_distrib")
 * Release candidate and official release builds ("official-release")

The release processes, whether nightly, weekly, or official, also have multiple functions:

 * Build the software, ensuring it passes its unit tests.
 * Run limited integration tests (by building and testing the "lsst_ci" product)
 * Tag the GitHub repositories that make up the software to indicate the release content.
 * Publish source packages, an environment file, and a tag file to the eups.lsst.codes distribution server allowing others to use "eups distrib install" to install the release.
 * Generate and publish tarball binary packages for Linux and macOS to the eups.lsst.codes distribution server to speed up installations.
 * Generate and publish a Docker container containing the software to hub.docker.com (and potentially other container/artifact registries).
 * Execute characterization jobs that generate metrics about the quality of the Alert Production and Data Release Production components of the release that are pushed to the SQuaSH system ("ap_verify" and "verify_drp_metrics").
 * Trigger, via a GitHub Action, building and publication of the JupyterLab image for the Rubin Science Platform Notebook Aspect.

In addition, there is a separate system that maintains the "shared stack" installation of the Science Pipelines on the developer systems at NCSA.


Components
==========

Jenkins (ci.lsst.codes)
-----------------------

Control (the "jenkins-master" node) is hosted in the Elastic Computing Cloud (EC2) in Amazon Web Services (AWS); this serves the Jenkins UI at https://ci.lsst.codes.

Workers are "agent-ldfc" pods running on the NCSA k8s-devel Kubernetes (K8s) cluster.
They make up a StatefulSet called "agent-ldfc".
(These pods may have been configured using Terraform somewhere but almost certainly have had many manual changes not reflected in the original Terraform.)
Each pod is composed of a docker-in-docker container (https://github.com/lsst-sqre/docker-dind/), a docker-gc container that cleans up old docker containers and images (https://github.com/lsst-sqre/docker-docker-gc), and the main swarm container (https://github.com/lsst-sqre/docker-jenkins-swarm-client) that actually communicates with the Jenkins central control and executes pipelines.
The pods use disk space at /project/jenkins/prod, with each subdirectory allocated as a separate PV in K8s.
This space is located on GPFS at NCSA, but it is mounted via NFS from lsst-nfs.ncsa.illinois.edu in order to protect the filesystem from potential problems caused by the use of root in the Jenkins containers.

Workers are also installed on macOS machines located in the Tucson AURA headquarters machine room.
These machines are named mac1-6.lsst.cloud.
The Jenkins UI is used to start and configure workers on these machines, launching agents via ssh to the "square" account using credentials stored in the "sqre-osx" Jenkins secret.

There is a final "snowflake" worker used for release builds that also runs in EC2.

eups.lsst.codes
---------------

The primary publication location for the eups distribution server is a Simple Storage Service (S3) bucket at AWS.
Publication occurs in certain pipelines via the "aws s3 cp" command using credentials stored in the "aws-cmirror-push" Jenkins secret.
The data in that bucket is replicated via code in https://github.com/lsst-sqre/terraform-scipipe-publish/tree/master/s3sync to a Persistent Disk filesystem attached to a deployment in Google Kubernetes Engine (GKE) at Google Cloud Platform (GCP).
The GKE deployment runs a vanilla Apache container as specified in https://github.com/lsst-sqre/terraform-scipipe-publish/blob/master/tf/modules/pkgroot/pkgroot-deploy.tf along with an nginx ingress in order to support HTTPS.
The result is https://eups.lsst.codes.

hub.docker.com
--------------

The primary publication location for Docker containers is DockerHub at hub.docker.com.
Secondary publication to Google Artifact Registry at GCP is in development.
Several containers are published to DockerHub using credentials stored in the "dockerhub-sqreadmin" Jenkins secret.

GitHub
------

Release pipelines tag GitHub repositories to clearly designate what versions of the source code were incorporated into a release build.
These tagging operations use lsst-sqre/codekit as well as credentials in the "github-api-token-sqreadmin" Jenkins secret.

GitHub Actions
--------------

GitHub Actions (GHA) workflows in each repository are used to perform simple "lint"-style syntax checking and in certain cases more extensive tests.
Because each Science Pipelines package typically depends on many others, and because they frequently change together as well as separately, it is not considered feasible to have per-repository GHA workflows build and test each package.

LSST-the-Docs
-------------

Certain pipelines publish documentation via the LSST-the-Docs system.
Credentials for this are in the "ltd-mason-aws" and "ltd-keeper" Jenkins secrets.

Slack
-----

Most pipelines publish notifications of start, success, and failure to Slack channels.
Credentials for this are in the "ghslacker" Jenkins secret.

SQuaSH
------

Release pipelines measure certain metrics based on applying the Science Pipelines code to known data.
These metrics are pushed to a metrics dashboard system known as SQuaSH using the lsst/verify framework.
This framework takes credentials for an API endpoint which are stored in the "squash-api-user" Jenkins secret.

conda-forge
-----------

The third-party dependencies (Python and C++) of the Science Pipelines are, to the extent possible, installed in a conda environment via the rubin-env metapackage from the conda-forge channel.
conda-forge is used because it has strong policies around maintaining consistency and interoperability of the packages it publishes.

CernVM-FS
---------

CernVM-FS is a globally-distributed, locally-cached read-only shared POSIX filesystem.
CC-IN2P3 takes tagged weekly and official release source packages in the eups distribution server and rebuilds them into a binary "stack" installation in CernVM-FS, including a base rubin-env conda environment and an extended one with additional convenience packages.
Singularity container images are also produced and stored in this system.
Other artifacts could be similarly published.

As a shared filesystem, it is easy to ensure that developer systems and batch poroduction worker systems share the same view of the software to be executed.
This makes CernVM-FS an attractive software distribution mechanism for user-level applications that do not need the OS-level package and isolation that containers provide.
Note that while it is not a container registry per se, as mentioned, container images can still be usefully disseminated via CernVM-FS.

lsst-sqre/ci-scripts
--------------------

This repo contains four scripts:

* ``create_xlinkdocs.sh`` runs the doxygen build for the entire stack, resulting in doxygen.lsst.codes.
  It is invoked by ``lsstswBuild.sh``.
* ``jenkins_wrapper.sh`` translates from Jenkins-specified environment variables to script arguments for ``lsstswBuild.sh``.
  It executes ``deploy`` from ``lsstsw`` to prepare the build tree and environment.
* ``lsstswBuild.sh`` invokes ``envconfig`` from ``lsstsw`` to initialize the conda environment and then invokes ``rebuild`` to actually perform the build.
  If successful, it runs the doxygen build using ``create_xlinkdocs.sh``.
* ``run_verify_drp_metrics.sh`` sets up the code in ``faro`` and a dataset and then runs a dataset-dependent script to generate metrics by analyzing the results of running pipeline algorithms on that dataset.
  This is triggered by the "verify_drp_metrics" post-release job in Jenkins.

lsst/lsstsw
-----------

This repo contains code that was originally intended to handle the process of publishing source and binary tarball packages to the eups distribution server.
It has since expanded to be a more general-purpose multi-package build tool for the Science Pipelines.
Information on it is available in https://developer.lsst.io/stack/lsstsw.html

The primary scripts here are:

* ``deploy``, which installs needed code including conda, the rubin-env environment, and the ``lsst_build`` tool.
* ``rebuild``, which uses ``lsst_build`` to prepare eups package sources and then build them.
* ``publish``, which takes an existing eups installation and creates distribution server packages, tag files, and environment listings in a separate directory.
  This "distribution server" directory is ready to be mirrored to the real Web-hosted distribution server.

Some configuration information for the scripts is contained in ``etc/settings.cfg.sh``.
The ``etc/manifest.remap`` file must contain the names of all packages that use Git LFS, as they cannot be packaged normally by eups.
``etc/exclusions.txt`` is likely vestigial.

The ``lsst/versiondb`` repo is used to maintain records of the versions of packages that have had builds attempted.
See the README file in ``lsst/lsst_build`` for more information.

lsst/lsst_build
---------------

This repo is used by ``lsst/lsstsw``.
It contains Python code to rapidly clone all of the packages needed to build a Science Pipelines product, given the git repository configuration in ``lsst/repos``, check out appropriate git refs in each clone, and then invoke ``eupspkg`` to build them if needed.

lsst/lsst
---------

This repo contains the ``newinstall.sh`` and ``lsstinstall`` scripts that create the appropriate environment for using ``eups distrib`` to install Science Pipelines packages, either from source or from binary tarballs.
They install conda, the rubin-env environment, and configure an eups "stack" location, and they create a script that can be sourced to activate this environment in a shell.

eups, eupspkg, and eups distrib
-------------------------------

eups is the package manager used by the Science Pipelines.
It enables flexible combinations of versions of packages, including under-development versions.
Some information about it is available at https://developer.lsst.io/stack/eups-tutorial.html

eupspkg is the tool within eups that builds source and binary packages.
It has extensive documentation in a docstring within https://github.com/RobertLuptonTheGood/eups/blob/master/python/eups/distrib/eupspkg.py
Note that there are two kinds of source packages: "git" and "package".
"git" packages merely refer to a particular repo and so use much less space on the distribution server but somewhat more space on the installing client.
"package" packages include a complete copy of the source code, so they use much more space on the distribution server but less space on the client.

eups distrib is an independent module within eups that handles interactions with a distribution server that provides source and/or binary packages.
There are several types, but we currently use only the eupspkg variety, as specified in https://eups.lsst.codes/stack/src/config.txt
Note that the binary tarball servers also have similar configuration files, such as https://eups.lsst.codes/stack/osx/10.9/conda-system/miniconda3-py38_4.9.2-2.0.0/config.txt

sconsUtils
----------

sconsUtils is the library of code used with the scons build tool that customizes it for Science Pipelines use.
It standardizes handling of C++ and Python code as well as documentation, tests, and eups packaging information.
In addition to package dependencies from eups table files, it also uses special ``ups/*.cfg`` files to track dependency information, particularly for C++.
(However dependency information for C++-accessible shared libraries in the rubin-env conda environment is obtained from ``sconsUtils/configs``, not from ``ups`` directories.)

Docker Containers
=================

Several containers are published via the build system.

newinstall
----------

The "newinstall" container contains the conda environment used for the Science Pipelines.
Since this environment changes much less frequently than the Science Pipelines code, it saves time and space to have it as a base container.
This container is built by the "sqre/infra/build-newinstall" job, which is triggered on updates to the "lsst/lsst" GitHub repository or manually whenever desired.
Typically it would be triggered when a new build becomes available of the rubin-env conda environment that might fix a (temporary) problem in a previous container build.

Note that the build-newinstall job builds the version of the rubin-env environment that is specified in etc/scipipe/build-matrix.yaml, not the default in newinstall itself.
The container is pushed with a tag containing that version, as well as a "latest" tag that is typically enabled.

centos
------

The "centos" container contains the LSST Science Pipelines code in "minimized" form.
The lsst-sqre/docker-tarballs Dockerfile is used to install a "stack" from binary tarballs and then to strip out debugging symbols, test code, documentation in HTML and XML form, and C++ source code.
The "shebangtron" script that fixes "#!" lines in Python scripts is also executed.

sciplat-lab
-----------

Jenkins used to build the sciplat-lab containers used by the Rubin Science Platform directly, but it now merely triggers a certain GitHub Action using the "github-api-token-sqreadmin" credentials.


Jenkins Pipelines
=================

Most of these pipelines use complex Groovy scripts to describe their stages and steps.
One technique used frequently is to place the main activity of the stage within a "run()" function, write a dynamic Dockerfile, build a Docker container from it, and then execute the "run()" function within that Docker container.
This provides isolation at the cost of some complexity.

Much of the common pipeline code is found in the large library "pipeline/lib/util.groovy".


Bootstrap
---------

sqre/seeds/dm-jobs
^^^^^^^^^^^^^^^^^^
Most pipelines are written in Groovy and have two components: a "job" component that defines parameters for the pipeline and its triggers, and a "pipeline" component that defines the stages and steps to be executed.

The "seeds" pipeline installs all of the "job" components in the Jenkins configuration, allowing it to be defined by code rather than manual manipulation of the GUI.
It must be rerun any time a "job" component is modified.
It does not need to be rerun when a "pipeline" component is modified, as those are dynamically loaded from the "main" branch of lsst-dm/jenkins-dm-jobs as each pipeline begins execution.

Science Pipelines builds
------------------------

These build pipelines do not publish artifacts, but the extended integration test run by some of them do publish metrics.

stack-os-matrix
^^^^^^^^^^^^^^^

The primary build used by developers.
Runs on Linux and macOS.
To enable these jobs to run as rapidly as possible, they reuse state from previous builds, including the rubin-env environment.
However, this state grows with time so it does get cleaned up periodically.

The stack-os-matrix pipeline, via several layers of library code in pipeline/lib/util.groovy, invokes two layers of scripts in lsstsqre/ci-scripts (jenkinsWrapper.sh and lsstswBuild.sh) which in turn invoke the (somewhat documented in pipelines.lsst.io) lsst/lsstsw build tool which in turn uses the (relatively undocumented) lsst/lsst_build tool to invoke eupspkg on each repository which, for LSST Science Pipelines packages, invokes scons and the sconsUtils library to actually do the build and test of each package.

scipipe/lsst_distrib
^^^^^^^^^^^^^^^^^^^^

Clean build of the main branch of the Science Pipelines and lsst_ci integration tests.
The latter is primarily "pipelines_check", a minimal "aliveness" test; it also forces building and testing of several "obs_*" packages,
Since this build installs rubin-env from scratch, it ensures that we are prepared for any dependency updates.

scipipe/ci_hsc
^^^^^^^^^^^^^^

Clean build of the ci_hsc integration tests.
Note that Science Pipelines packages that are not used by ci_hsc are not built.
For now, "ci_hsc" runs both "ci_hsc_gen2" and "ci_hsc_gen3" tests, although Gen2 will soon be removed.

scipipe/ci_imsim
^^^^^^^^^^^^^^^^

Clean build of the ci_imsim integration tests.
Note that Science Pipelines packages that are not used by ci_imsim are not built.


Container builds
----------------

sqre/infra/build-newinstall
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Builds the newinstall container as described above.

sqre/infra/build-sciplatlab
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Triggers the GHA to build the RSP container as described above.

Administrative tasks
--------------------

sqre/infra/jenkins-node-cleanup
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Runs periodically (every 10 minutes) to check the amount of free space in each worker's workspace.
If this falls below the configured threshold (100 GiB default), the contents of the workspace directory will be removed unless a job is actively using it.
If the "FORCE_CLEANUP" parameter is specified, all workers' workspaces will be cleaned unless they have active jobs.
If the "FORCE_NODE" parameter is specified and "FORCE_CLEANUP" is not, only that node will be cleaned if it does not have an active job.

sqre/infra/clean-locks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Manually triggered when an interrupted build leaves eups lock files behind.
In most cases nowadays, eups locking should be disabled, meaning that this job should be unnecessary.

Release builds
--------------

Also publish doxygen output to doxygen.lsst.codes.

release/nightly-release
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Nightly build (d_YYYY_MM_DD)

release/weekly-release
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Weekly build (w_YYYY_WW)

release/official-release
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Official release build (vNN)

Release build components
------------------------

release/run-rebuild
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Runs a complete build, unit tests, and default integration tests on the canonical platform (Linux).

release/run-publish
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Publishes source packages, the release tag, and an environment file to the eups distribution server.

release/tarball
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Builds binary tarballs from the source packages, copies them into a local "distribution server" directory, tests that binary installs work correctly, including running a minimal check, and publishes the distribution server directory to the cloud distribution server.

docker/build-stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Builds the Science Pipelines Linux container from the binary tarballs, editing the result as described earlier.


Triggered post-release jobs
---------------------------

sqre/infra/documenteer
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Builds and publishes an edition of the pipelines.lsst.io website based on the centos Science Pipelines container.

scipipe/ap_verify
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Runs ap_verify code from the centos Science Pipelines container on test datasets, publishing metrics to SQuaSH.

sqre/verify_drp_metrics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Runs faro code from the centos Science Pipelines container on test datasets, publishing metrics to SQuaSH.

Qserv builds
------------

These three pipelines will very soon be obsolete; they were used to build the Qserv distributed database software package.

 * dax/qserv_distrib
 * dax/release/rebuild_publish_qserv-dev
 * dax/docker/build-dev

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
