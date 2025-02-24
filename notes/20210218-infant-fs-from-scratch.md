# Building and running the FreeSurfer Infant pipeline from Scratch

## Overview

1) Build the container to compile FreeSurfer and NiftiReg
2) Compile FreeSurfer
3) Compile NiftyReg
4) Build the container to run the Infant pipeline
5) Find an Infant Model
6) Find a Subject
7) Run the pipeline

## Pre-requisites

- Git (tested with 2.17.1)
- Git annex (tested with 6.20180227)
- Docker (tested with version 19.03.12, build 48a66213fe)
- `wget` and `tar`
- A [FreeSurfer License file](https://surfer.nmr.mgh.harvard.edu/fswiki/License)
- An internet connection
- ~70Gb free hard drive space

## Setup

Define the following local variables
- `FS_PKG`: The directory for FreeSurfer's pre-compiled binaries
- `FS_CODE`: The directory for FreeSurfer's source code
- `FS_BIN`: The directory for FreeSurfer's complied binaries 
- `FS_DATAIN`: The directory to store FreeSurfer data (not the FreeSurfer subject dir!)
- `FS_SUB`: The FreeSurfer subject directory
- `FS_LICENSE`: The freesurfer license file
- `FS_DOCKER`: The directory for the fs-docker repo
- `FS_INFANT_MODEL:` The directory of the model for the infant stream (wip)
- `NR_CODE`: The directory for NiftiReg's source code
- `NR_BIN`: The directory for NiftyReg's compiled binaires

According to your local environment
```
export FS_PKG=/home/ubuntu/environment/baby2/pkg
export FS_CODE=/home/ubuntu/environment/baby2/freesurfer
export FS_BIN=/home/ubuntu/environment/baby2/bin
export FS_DATAIN=/home/ubuntu/environment/data/input
export FS_SUB=/home/ubuntu/environment/data/sub
export FS_LICENSE=/home/ubuntu/data/license.txt
export FS_DOCKER=/home/ubuntu/environment/baby2/fs-docker
export FS_INFANT_MODEL=/home/ubuntu/environment/baby/infant-model
export NR_CODE=/home/ubuntu/environment/baby2/niftyreg-git
export NR_BIN=/home/ubuntu/environment/baby2/niftyreg-bin
```

Define the build params for this specific build:
- `FS_GIT_REMOTE`: The git remote to use to fetch FreeSurfer's source code
- `FS_GIT_ID`:  The git commitID or branch name to use with FreeSurfer's git repo 
- `FS_GIT_ANNEX_REMOTE`: The remote of FreeSurfer's git annex repo
- `FS_DOCKER_GIT_REMOTE`: The remote of the fs-docker git repo
- `FS_DOCKER_GIT_ID`: The git commitID or branch name to use with the fs-docker git repo
- `NR_GIT_REMOTE`: The git remote to use to fetch NiftyReg's source code
- `NR_GIT_ID`: The git commitID or branch name to use with NiftyReg's git repo
- `FS_PKG_REMOTE`: The location of FreeSurfer's pre-compiled binaries

```
export FS_GIT_REMOTE=git@github.com:pwighton/freesurfer.git
export FS_GIT_ID=20210115-fs-baby
export FS_GIT_ANNEX_REMOTE=https://surfer.nmr.mgh.harvard.edu/pub/dist/freesurfer/repo/annex.git
export FS_DOCKER_GIT_REMOTE=git@github.com:pwighton/fs-docker.git
export FS_DOCKER_GIT_ID=20201206-baby
export NR_GIT_REMOTE=https://git.code.sf.net/p/niftyreg/git
export NR_GIT_ID=4e4525b84223c182b988afaa85e32ac027774c42
export FS_PKG_REMOTE=http://surfer.nmr.mgh.harvard.edu/pub/data/fspackages/prebuilt/centos7-packages.tar.gz
```

Make required directory paths:
```
mkdir -p $FS_PKG
mkdir -p $FS_CODE
mkdir -p $FS_BIN
mkdir -p $FS_DATAIN
mkdir -p $FS_SUB
mkdir -p $FS_DOCKER
mkdir -p $NR_CODE
mkdir -p $NR_BIN
```

## Download remaining pre-reqs:

Download FreeSurfer, git annex files and pre-compiled binaries
```
git clone $FS_GIT_REMOTE $FS_CODE 
cd $FS_CODE
git checkout $FS_GIT_ID
git remote add datasrc $FS_GIT_ANNEX_REMOTE
git fetch datasrc
git annex enableremote datasrc
git annex get .
wget -c $FS_PKG_REMOTE -O - | tar -xz -C ${FS_PKG} && mv ${FS_PKG}/packages/* ${FS_PKG} && rm -rf ${FS_PKG}/packages
```

Clone the `fs-docker` repo (not needed if not building containers)
```
git clone $FS_DOCKER_REMOTE $FS_DOCKER
cd $FS_DOCKER
git checkout $FS_DOCKER_GIT_ID
```

NiftyReg
```
git clone $NR_GIT_REMOTE $NR_CODE
cd $NR_CODE
git checkout $NR_GIT_ID
```

Get test data (wip)
```
rsync -r pwighton@door.nmr.mgh.harvard.edu:/autofs/space/vault_021/users/lzollei/DHCP-2nd-Processing/Data/sub-CC00656XX13 $FS_DATAIN
rm -rf $FS_SUB/sub-CC00656XX13_ses-217601/
mkdir -p $FS_SUB/sub-CC00656XX13_ses-217601/
cp \
  $FS_DATAIN/sub-CC00656XX13/ses-217601/sub-CC00656XX13_ses-217601_desc-restore_space-T2w_T1w.nii.gz \
  $FS_SUB/sub-CC00656XX13_ses-217601/mprage.nii.gz
```

Get infant model data and put in `$FS_INFANT_MODEL` (todo)
```
???
```

## Build the container used to compile FreeSurfer and NiftiReg.

The container `pwighton/fs-dev-build` is used to compile the FreeSurfer and NiftiReg binaries.  It can be built with the makefile targets `fs-build` and `fs-build-nc`.  The targets are the same, except `fs-build-nc` instructs docker to not use any caching.  Use `make fs-build` if you would like a cached build.
```
cd $FS_DOCKER
make fs-build-nc
```

The container defines mount points for:
- `/fs-code`: Location of FreeSurfer source code (input)
- `/fs-pkg`: Location of FreeSurfer pre-compiled binaries (input)
- `/fs-bin`: Location to store compiled binaries (output)

## Compile FreeSurfer

Compile FreeSurfer and store the result in `$FS_BIN` directory.

*pw: for some reason I can't go `docker run cmd1 && cmd2` (todo)*

```
docker run -it --rm \
  -v ${FS_BIN}:/fs-bin \
  -v ${FS_PKG}:/fs-pkg \
  -v ${FS_CODE}:/fs-code \
  -w /fs-code \
  -u ${UID}:${GID} \
  pwighton/fs-dev-build:latest \
    cmake \
      -DFS_PACKAGES_DIR="/fs-pkg" \
      -DCMAKE_INSTALL_PREFIX="/fs-bin" \
      -DBUILD_GUIS=OFF \
      -DMINIMAL=ON \
      -DINFANT_MODULE=ON \
      -DGFORTRAN_LIBRARIES="/usr/lib/gcc/x86_64-linux-gnu/5/libgfortran.so" \
      -DINSTALL_PYTHON_DEPENDENCIES=OFF \
      -DDISTRIBUTE_FSPYTHON=OFF \
      .
```

```
docker run -it --rm \
  -v ${FS_BIN}:/fs-bin \
  -v ${FS_PKG}:/fs-pkg \
  -v ${FS_CODE}:/fs-code \
  -w /fs-code \
  -u ${UID}:${GID} \
  pwighton/fs-dev-build:latest \
    make -j 4
```

```    
docker run -it --rm \
  -v ${FS_BIN}:/fs-bin \
  -v ${FS_PKG}:/fs-pkg \
  -v ${FS_CODE}:/fs-code \
  -w /fs-code \
  -u ${UID}:${GID} \
  pwighton/fs-dev-build:latest \
    make install
```
    
## Compile NiftyReg

Compile NiftyReg and store the result in `$NR_BIN` directory.

*pw: for some reason I can't go `docker run cmd1 && cmd2` (todo)*

```
docker run -it --rm \
  -v ${NR_CODE}:/nr-code \
  -v ${NR_BIN}:/nr-bin \
  -w /nr-bin \
  -u ${UID}:${GID} \
  pwighton/fs-dev-build:latest \
    cmake \
      -DCMAKE_INSTALL_PREFIX="/nr-bin" \
      -DCMAKE_BUILD_TYPE="Release" \
      -DUSE_SSE="OFF" \
      -DUSE_DOUBLE="ON" \
        /nr-code
```

```
docker run -it --rm \
  -v ${NR_CODE}:/nr-code \
  -v ${NR_BIN}:/nr-bin \
  -w /nr-bin \
  -u ${UID}:${GID} \
  pwighton/fs-dev-build:latest \
    make -j 4
```
 
```
docker run -it --rm \
  -v ${NR_CODE}:/nr-code \
  -v ${NR_BIN}:/nr-bin \
  -w /nr-bin \
  -u ${UID}:${GID} \
  pwighton/fs-dev-build:latest \
    make install
```

## Build the container to run the infant pipeline

The container `pwighton/fs-baby` is used to run FreeSurfer's infant pipeline.  It can be built with the makefile targets `fs-baby` and `fs-baby-nc`.  The targets are the same, except `fs-baby-nc` instructs docker to not use any caching.  Use `make fs-baby` if you would like a cached build.
```
cd $FS_DOCKER
make fs-baby-nc
```

The container consists of:
  - A neurodocker base container that installs FSL and python
  - Mount points for:
    - `/fs-bin`: FreeSurfer binaries (compiled above)
    - `/nr-bin`: NiftyReg binaries (compiled above) (pw: adding niftyreg support to neurodocker would simplify our build.. todo)
    - `/fs-pkg`: FreeSurfer pre-compiled packages (pw: this is only needed for one file.. todo)
    - `/fs-sub`: FreeSurfer's subject directory
    - `/fs-infant-model`: The infant model (todo define)
  - An entrypoint script, that:
    - Manages the FreeSurfer license (todo)
    - Manages the setup of the infant model (todo)

## Invoke the container and run the infant pipeline

```
docker run -it --rm \
  -v ${FS_BIN}:/fs-bin \
  -v ${FS_SUB}:/fs-sub \
  -v ${FS_PKG}:/fs-pkg \
  -v ${NR_BIN}:/nr-bin \
  -v ${FS_INFANT_MODEL}:/fs-infant-model \
  -v ${FS_LICENSE}:/data/license.txt \
  -e FS_LICENSE='/data/license.txt' \
  -u ${UID}:${GID} \
  pwighton/fs-baby:latest \
    infant_recon_all --s sub-CC00656XX13_ses-217601 --age 0
```

## Minify

todo