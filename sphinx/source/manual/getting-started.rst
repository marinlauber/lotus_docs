.. manual-getting-started:

*****************
 Getting Started
*****************

* :ref:`required`
* :ref:`set-up`
* :ref:`hpc`
* :ref:`docker`

.. _required:

####################
 Required software
####################

**Linux/Unix and Developer Tools**

Lotus needs to compile a small fortran script file at the start of every simulation. This is way too difficult to do with Windows machines. (``cmake`` is an option, but we don't recommend it.)

Ubuntu comes preloaded with ``git`` and ``make``. OSX users should open the ``terminal`` app and type
::
    $ git

At the prompt, you can download the developer tools.

**Programming languages and libraries**

`gfortran <https://gcc.gnu.org/wiki/GFortranBinaries>`_ (v4.8+): The code is written in fortran, so you'll need a compiler. Check your version using
::
    $ gfortran --version

`MPICH <http://www.mpich.org/downloads/>`_ : To run in parallel on a multi-core machine, you'll need to install this library.

`python <https://www.python.org/>`_ : We recommend doing all of the data-analysis and visualization in python. This can also be used to run a series of simulations and to update these documents.


**Additional Software**

`convert <http://www.imagemagick.org/script/convert.php>`_ (v6.9+): Convert is a cross platform library to create and manipulate images. It is required for the code to generate images during run-time.

`paraview <http://www.paraview.org/>`_ : The output binary file can be read and rendered using paraview.

`atom <https://atom.io/>`_ : This is a great text editor with lots of add-ons for efficient and easy coding such as the `language-fortran` package.

.. _set-up:

#############
 Setting up
#############

**Add Profile**

Create a ``~/.profile`` file in the home directory and add the following lines
::
    $ export MGLHOME=$HOME'/Workspace/Lotus/solver'
    $ export PATH=$PATH:$MGLHOME/bin:$MGLHOME/oop/bin
    $ export MPI='true'
    $ export F90='gfortran'
    $ alias open='xdg-open'
    $ ulimit -s unlimited
        
Source the ``.profile`` file into the ``.bashrc`` file
::
	$ echo "source ~/.profile" >> ~/.bashrc

**Clone Libraries**

Now you're ready to get Lotus. Make a Workspace folder and go into it
::
    $ mkdir -p ~/Workspace/Lotus/
    $ cd ~/Workspace/Lotus/

Clone the solver repositories
::
    $ git clone https://github.com/weymouth/lotus_geom_lib solver
    $ cd solver
    $ git clone https://github.com/weymouth/lotus_oop oop

Now you should have both the geom and the fluid libraries at ``solver/geom_lib`` and ``solver/oop``, respectively.

.. warning::
    If your are doing some sort of developement, your are encourage to creat your own branch ``git checkout my_new_branch`` at this point, this ensures the master is always the latest working branch, and you can revert to it if you break your developement branch.

**Build Libraries**

This step is optional since running the Lotus will build the libraries automatically
::
    $ cd ~/Workspace/Lotus/solver/geom_lib
    $ make
    $ cd ../oop
    $ make

There should be no errors though possibily a few warnings.

.. _hpc:

#################
  HPC instalation
#################

Iridis 4
########

**Set-up Iridis 4**

First you will need and Iridis4 account. You can request on to isolution `here <https://sotonproduction.service-now.com/serviceportal?id=sc_cat_item&sys_id=427c697fdb9ae300f91c8c994b96192>`__. Once this is done, login to Iridis via ssh with an activated account when connected to the university network or VPN:
::
    $ ssh -x <username>@iridis4_a.soton.ac.uk

**Pull Lotus solver from github**

Once accessed to the login node you will be in a remote terminal. First, create a folder for Lotus directory:
::
    $ mkdir ~/Lotus

Now get in that directory and pull the latest version of the geom and fluid libraries from Github. Make sure you are added as a collaborator these repositories (ask Gabe) or you won't be able to access them.
::
    $ cd ~/Lotus
    $ git clone https://<github-username>:<github-password>@github.com/weymouth/lotus_geom_lib.git solver
    $ cd solver
    $ git clone https://<github-username>:<github-password>@github.com/weymouth/lotus_oop.git oop

The clone command pulls the 'master' branch by default. To access any other branch just use
::
    $ cd oop
    $ git checkout <branch-name>

And this will pull the branch, checkout into it, and track the remote branch.

**Configure your bash profile**

In order to run Lotus the bash profile needs to be amended and the linux ``.profile`` script sourced into your ``~/.bash_profile``. Create the linux ``.profile`` Lotus script (almost same as profile_ubuntu.txt for which only the first line needs to be modified):
::
    $ cd
    $ nano .profile

    $ export MGLHOME=$HOME'/Lotus/solver'
    $ export PATH=$PATH:$MGLHOME/bin:$MGLHOME/oop/bin
    $ export MPI='true'
    $ export F90='gfortran'
    $ alias open='xdg-open'
    $ ulimit -s unlimited
    $ alias trash='gvfs-trash'

Save this file with
::
   $ ctrl+o
   $ ctrl+x

(You can also scp your local ``.profile`` and just modify the first line as above). Edit the ``~/.bash_profile`` script to source the new ``.profile``
::
    $ nano .bash_profile

Add the following lines
::
    # Source your linux .profile
    if [ -f ~/.profile ]; then
        . ~/.profile
    fi

Save the file as before. To update the ``~/.bash_profile`` for the current log in session you need to source the profile in the command line.
::
    $ source ~/.bash_profile

You don't need to do this every time you log in to iridis. Now you should be good to compile and run Lotus.


**Load the required modules and compile lotus**

The module required to run Lotus are: R, gcc, openmpi. They can be loaded into your login node as:
::
    $ module purge
    $ module load binutils/2.26
    $ module load R/3.3.0
    $ module load openmpi/1.10.6/gcc
    $ module load gcc/6.1.0

In order to avoid problems with the openmpi library (mpi.mod), the FC (Fortran compiler) environment
variable needs to be exported as:
::
    $ export FC=gcc-6.1.0

Otherwise the openmpi Fortran compiler (mpif90) will complain regarding the GNU Fortran version.
Also note that the last module to be uploaded is gcc/6.1.0 to override the default gcc module loaded
with openmpi.

Now go to your ``~/Lotus/solver/geom_lib/`` and ``~/Lotus/solver/oop/`` folder and try to compile the code:
::
    $ cd ~/Lotus/solver/geom_lib/
    $ make clean
    $ make
    $ cd ~/Lotus/solver/oop/
    $ make clean
    $ make

If the mpi.mod library gives trouble is probably because the openmpi module has been loaded before
the gcc module. To solve it try:
::
    $ make clean
    $ module unload openmpi/1.10.6/gcc
    $ module load openmpi/1.10.6/gcc
    $ make

**Run your Lotus test simulation into the login nodes**

Before running, the openmpi module which compiles lotus.f90 needs to be changed to version 1.8.3 since version
1.10.6 can be problematic (see * link). The best way to do this is to purge and load the modules again:
::
    $ module purge
    $ module load binutils/2.26
    $ module load R/3.3.0
    $ module load openmpi/1.8.3/gcc
    $ module load gcc/6.1.0

Now you can give it a try in the login node (a very short simulation, just to see that it runs correctly).
To do so, add a ``/projects`` folder into the ``~/Lotus/`` folder previously created and scp a valid lotus.f90 file).
::
    $ mkdir ~/Lotus/projects/
    $ mkdir ~/Lotus/projects/project1/

scp here you lotus.f90 file, eg:
::
    $ scp lotus.f90 <username>@iridis4_a.soton.ac.uk:~/Lotus/projects/project1

Run Lotus as usual, eg:
::
    $ runLotus 4 test

**Submit your Lotus simulation to the computational nodes**

Once the compilation and the execution of Lotus has succeed you are good to submit your Lotus
simulation into the computational nodes. To do so, a submission file is required.
Have a look at the subLotus file and familiarize with the submission commands.

Remember that the number of processors specified in the lotus.f90 (blocs) and at the runnining command
(subLotus last line) must be the same as the number of processors specified at the #PBS variable of the
subLotus file. Eg: in lotus.f90:

.. code:: fortran

    integer :: b(3) = (/8,4,2/)   ! Blocks. Product = number of processors.

subLotus:
::
    # PBS -l nodes=4:ppn=16
    # Remember that Iridis has 2 processors per nodes with 8 cores per processor = 16 processor cores per node

    runLotus 64 test

Now, scp subLotus to ``~/Lotus/projects/project1/`` and modify with your own preferences. Submit the job:
::
    $ qsub subLotus

.. note::
    The qsub command will not work if you have loaded 'module purge' into the command line or your ~/.bash_profile. To rectify this simply log out and log back in to iridis, making sure that you are not loading 'module purge' in your ~/.bash_profile.

Output data will be found in the folder specified in the subLotus file. Useful information about Iridis (job submissions, queues, forums, etc) can be found in:

    * https://hpc.soton.ac.uk/community/projects/iridis/wiki
    * https://hpc.soton.ac.uk/community/boards/1/topics/7832?r=7861#message-7861

**subLotus**

.. code:: bash

    #!/bin/bash -login
    #PBS -S /bin/bash

    # Job script to run mpi job in parallel. The line above loads your bash profile into the computational node.

    # Set resource requirements for job
    # 1. number of nodes and processors per nodes
    # 2. runtime expected
    # 3. job name
    # - these can be overridden on the qsub command line

    #PBS -l nodes=1:ppn=16
    #PBS -l walltime=00:10:00
    #PBS -N testing
    #PBS -m abe

    # Load required modules so that the commands at the computational node are found
    module purge
    module load binutils/2.26
    module load openmpi/1.8.3/gcc
    module load gcc/6.1.0
    module load R/3.3.0

    # Change to directory from which job was submitted
    cd $PBS_O_WORKDIR

    # Run Lotus (mpif90). Note that total np must be the same as stated before. You can also use your /scratch
    # directory if the simulation writes too much data for your /home directory.

    runLotus 16 test



Iridis 5
########

**Using Singularity, the container-based platform for HPC systems**

Because Docker requires root privileges to run on systems, it is not possible to use in clusters. Therefore, an alternative container-based platform is required to run it in HPC systems. This is `Singularity <https://singularity.lbl.gov/>`__ and here it is explained how to use Lotus in Iridis5 through this app. This is very useful so you can avoid the troubles of using different OpenMPI version for build/run. 

Note that it is not possible to run the current Lotus image through Singularity on Iridis4 since the image is built on top of Ubuntu 18.04, which uses a more recent Linux kernel than the latest available in Iridis4.

To run Lotus through Singularity on Iridis5 follow the next steps.

Connect to Iridis5 and create a folder for Lotus in your home directory (if you do not have it already):
::
    $ ssh username@iridis5_b.soton.ac.uk
    $ mkdir -p Lotus/bin

In the ``~/Lotus/bin`` director, download and build the Lotus Docker image as a Singularity image (you need to be added as collaborator in the Docker Hub repository, send me an `email <mailto:b.fontgarcia@soton.ac.uk>`__:
::
    $ cd ~/Lotus/bin
    $ export SINGULARITY_DOCKER_USERNAME=docker_username
    $ export SINGULARITY_DOCKER_PASSWORD=docker_password
    $ module load singularity
    $ singularity build lotus.sif docker://bfont/lotus

Copy the ``runSingularity`` script into the ``~/Lotus/bin`` directory. It should now contain the ``runSingularity`` script and the ``lotus.sif`` Singularity image. The ``runSingularity`` script needs execution permissions, so set them with:
::
    $ chmod +x runSingularity

Add the following lines (environmental variables) in `~/.bash_profile`:
::
    $ export LOCAL_LOTUS_DIR=$HOME'/Lotus'
    $ export LOTUS_BIN=$LOCAL_LOTUS_DIR'/bin'
    $ export LOCAL_PROJ_DIR='/scratch/'$USER'/Lotus'
    $ export DOCKER_PROJ_DIR='/app/projects'
    $ export PATH=$PATH:$LOTUS_BIN

Source again the ``~/.bash_profile`` script typing ``. ~/.bash_profile`` in the terminal.

Create a project folder in ``~/Lotus`` such as ``test``. Include in ``~/Lotus/test`` the provided ``subLotus`` script and a ``lotus.f90`` file you would like to run.

Edit the ``subLotus`` submission script to meet your ``lotus.f90`` requirements (such as number of processes), walltime, etc. The line that calls the ``runSingularity`` script to run the Singularity image (which includes the Lotus solver) is the last line of the subimission script:
::
    $ runSingularity $SLURM_NTASKS test

Specify the folder where Lotus will place the output, in this case ``test``. The script ``runSingularity`` automatically places this folder in your ``scratch`` directory. Check the script for more info. You can specify a restarting folder as normally with:
::
    $ runSingularity $SLURM_NTASKS test2 test

Where the restarting folder has to be located in the ``/scratch/$USER/Lotus/test`` directory.

The job can be finally submitted from the ``~/Lotus/test`` directory as follows:
::
    $ cd ~/Lotus/test
    $ sbatch subLotus

and results will be placed in the ``/scratch/$USER/Lotus`` directory.

**runSingularity**

.. code:: bash

    #!/bin/bash

    # Create the /scratch/$USER/Lotus folder if it does not exist
    if [ ! -d "$LOCAL_PROJ_DIR" ]; then
    mkdir -p $LOCAL_PROJ_DIR
    echo "Directory $LOCAL_PROJ_DIR created."
    fi

    # Check that at least the number of processors and the working directory is specified
    if [ $# -le 1 ]; then
    echo "usage: runSingularity proc_num work_dir resume_dir_path"
    exit 1
    fi

    # If the directories specified lack "/" at the end, add it.
    if [ $# -eq 3 ]; then
    if [ "${3: -1}" != "/" ]; then
        set -- "${@:1:2}" "$3/" 
    fi
    elif [ $# -eq 2 ]; then
    if [ "${2: -1}" != "/" ]; then
        set -- "$1" "$2/"
    fi
    fi

    # Setting default project folders in /scratch/$USER/Lotus. Comment the following lines to specify other paths
    if [ $# -eq 3 ]; then
    set -- "$1" "$LOCAL_PROJ_DIR/$2" "$LOCAL_PROJ_DIR/$3"
    elif [ $# -eq 2 ]; then
    set -- "$1" "$LOCAL_PROJ_DIR/$2"
    fi

    # Put the `lotus` singularity image in a container and run it. 
    # The /scratch folder is already binded to the $DOCKER_PROJ_DIR through the env variable $SINGULARITY_BINDPATH,
    # so it will find the projects in /scratch/$USER/Lotus automatically.

    singularity run $LOTUS_BIN/lotus.sif $@

**subLotus**

.. code:: bash

    #!/bin/bash

    #SBATCH --ntasks=16					# Request number of processes (if >40, multiple nodes are used in Iridis5) 
    #SBATCH --job-name=test				# Job name
    #SBATCH --time=00:10:00		 		# Walltime
    #SBATCH --mail-type=ALL				# Mail notifications

    module load singularity				# Load the singularity mode

    runSingularity $SLURM_NTASKS test	# run the singularity container with the number of processes specified above

.. _docker:

####################
 Docker instalation
####################

**1.** `Install Docker <https://docs.docker.com/install/linux/docker-ce/ubuntu/>`__

**2.** `Manage Docker as a non-root user <https://docs.docker.com/install/linux/linux-postinstall/>`__

Note that the files created during the simulation will still have root privileges. These may need to be restored to user privileges for post-processing reasons as explained in the remarks in **11**.

**3.** Test Docker: 
::
    $ docker run hello-world


**4.** Pull the Docker image based on Ubuntu which contains the Lotus solver (you need to be added as collaborator in the Docker Hub repository, send me an `email <mailto:b.fontgarcia@soton.ac.uk>`__)
::
    $ docker login --username your_username
    $ docker pull bfont/lotus

**5.** Create a working directory and the required subdirectories
::
    $ mkdir -p $HOME/Workspace/docker/Lotus/bin
    $ mkdir $HOME/Workspace/docker/Lotus/projects

**6.** Add the following lines (environmental variables) in ``~/.profile``:
::
    $ export LOCAL_LOTUS_DIR=$HOME'/Workspace/docker/Lotus'
    $ export LOCAL_PROJ_DIR=$LOCAL_LOTUS_DIR'/projects'
    $ export DOCKER_PROJ_DIR='/app/projects'
    $ export PATH=$PATH:$LOCAL_LOTUS_DIR/bin

**7.** Source ``~/.profile`` in ``~/.bashrc`` adding the following line in ``~/.bashrc``:
::
    $ . ~/.profile

**8.** Copy from this directory the following scripts: ``bin/runDocker``, ``bin/runStat``, ``lotus.f90``, and ``stat.py``. Put them in the relevant directory which should look like this:
::
    $LOCAL_LOTUS_DIR/
    lotus.f90
    stat.py
    bin/
        runDocker
        runStat
    projects/


The script ``runDocker`` is a wrap around ``docker run``, which function is to create a container from the Docker image you have just pulled. Also, ``runDocker`` passes the arguments normally used with ``runLotus`` to the Docker container and launches ``runStat`` if the ``stat.py`` python is present. The script ``runStat`` takes care of actually running ``stat.py``, which finally provides the post-processing results. 

The ``Dockerfile`` script (no need to copy this one) is the "Docker Makefile" that have been used to create the ``bfont/lotus`` image. It is informative to have a look at it and see how the Lotus docker image is created. Ultimatelly, the image is created using the command: ``docker build -t bfont/lotus .```

**9.** Run a simulation using the ``lotus.f90`` script in the Lotus folder with the ``runDocker`` bash script:
::
    $ runDocker 4 test1

**10.** Run again restarting from ``test1``
::
    $ runDocker 4 test2 test1

**11.** Remarks:

- When using ``runDocker``, the specified project folder will always be located in ``$LOCAL_LOTUS_DIR/projects``.
- If you are restarting from another simulation, for example ``test1``,  this needs to be located in ``$LOCAL_LOTUS_DIR/projects``.
- Even after following step **2**, the project folders will always have root privileges. To be able to run the post-processing python script, you will need to introduce your root password to change that folder privilages after the simulation finishes (this change of privileges is called from the runDocker script using ``sudo chown -R $USER:$USER $LOCAL_PROJ_DIR/project_directory``). If you fail to do so (long simulation not paying attention at when it finishes), no worries - you can do it manually afterwards with

.. code:: bash

    $ sudo chown -R $USER:$USER $LOCAL_PROJ_DIR/project_directory
    $ cd $LOCAL_PROJ_DIR/project_directory
    $ cp $LOCAL_LOTUS_DIR/stat.py .
    $ runStat
