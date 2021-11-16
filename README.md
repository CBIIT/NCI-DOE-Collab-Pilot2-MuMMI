# NCI-DOE-Collab-Pilot2-MuMMI

### Description ###

Multiscale Machine-learned Modeling Infrastructure (MuMMI) is a methodology that the Pilot 2 team has developed to study the interaction of active KRAS with the plasma membrane (and related phenomena) on very large temporal and spatial scales. To achieve this, MuMMI cleverly connects a macro model of the system to a micro model using machine-learning-based “dynamic importance sampling,” implementing the workflow on world-class supercomputing resources. Given the team’s impressive results, it is reasonable to ask whether MuMMI could be applied to a different biological system consisting of a new lipid bilayer into which a new type of protein is embedded.

MuMMI connects biological models of the membrane-protein system on two different scales:

1. A macro model based on the classical approximation theory for liquids in which 300 proteins (Modeled as single-particle beads) move around on a 1x1 &mu;m2 cross section of a perfectly flat (2D) plasma membrane.

2. A micro model that uses Martini coarse-grained (CG) molecular dynamics (MD) simulations to model a 30nm x 30nm “patch” of the macro model that necessarily contains at least one RAS protein


### Software workflow

MuMMI workflow manager (WM)is written in Python and uses a minimum of five nodes to run. The entire workflow is controlled via a configuration file with information on the machine requirement and frequencies of the tasks to be run. The workflow interfaces with Mastero that assits in query job status and to schedule new jobs when needed.

The workflow managers manages the state and execution of the framework, including:

1) **Generation of patches**: continuous polling of the running macro model simulation and generates patches from the incoming macro model snapshots (saved to disk) of the data resulting from the macro model simulation

    a) Several patches (spatial regions of scientific interest) are generated per snapshot, which are binary data files WM processes the snapshots as soon as            they are created
   
    b) At full scale, one snapshot is generated every 150 seconds and contains 300 patches
    
    c) Patches are stored as pickle objects in a tar archive file
    
2) **Selection of patches using ML**: patches are then passed to a pre-trained ML model (a deep neural network) that is used for online inference to evaluate patches for their configurational relevance, ranking all candidate patches correspondingly, using the top-ranked candidates to steer the target multiscale simulation toward CG simulations of scientific interest
  
    a) As new patches are generated, they are analyzed for their "importance" in real-time and are dynamically ranked in memory by this metric
    
    b) This queue of patches is truncated
    
    c) The importance of a patch cannot increase over time
    
3) **Tracking system-wide computational resources**: This is done indirectly by tracking the tasks currently running and their known allocation size, using Maestro to query thy status of running jobs (previously started by the WM) from which the available resources (with and without GPU) are calculated

4) **Management of CG simulations**: monitoring available resources, starting new simulation tasks when resources become available (or during the loading phase of the workflow), monitoring running tasks (both CG setup and simulations), and restarting any jobs that fail due to hardware issues or simulation instability, providing extensive checkpointing and restoring capabilities

    a) Patches are selected from the priority queue to start new CG setup jobs based on need and to match available resources
    
    b) Completed CG setup systems are selected to run as new CG simulations
    
    c) The WM allows staggering scheduling of new jobs to reduce the load on the underlying scheduler, which is useful when executing large simulations on              several thousands of nodes.
  
5) **Feedback to the macro model from the micro model**: updating the macro model parameters, periodically (every two hours) collecting the accumulated RAS-lipid radial distribution functions (RDFs) from each CG simulation via data provided by the in situ analysis, gathering these metrics through the filesystem reading the RDFs for each CG simulation, aggregating them through appropriate weighting, and converting to the free-energy functionals needed for the macro model.

    a) To be clear, the data collected for generating feedback is from the results of the in situ analysis
    
    b) The filesystem is used for this (as opposed to memory), posing scalability challenges
    
    c) Macro model periodically reads in improved RDFs accumulated by the workflow via CG simulations and calculates PMFs using the OZ and HNC equations
    
6) **Checkpointing and restarting**: WM monitors all running jobs for dead jobs due to node failures, file corruption, FS failures, etc.
    
    a) Failed jobs are automatically restarted at the last available checkpoint
    
    b) In case the control data corrupts, all status files are duplicated
    
    c) The WM uses several checkpoint files to save the current state of the simulation in a coordinated manner, which can be used to restore the simulation,            potentially with different configurations or even on a different machine


### Suite Components


1) **Maestro Workflow Conductor**: It is a Python-based workflow manager used in MuMMI that is used to run the macro model on partitions of the nodes, run inference on lipid patches in order to determine their importance, instantiate the CG setup jobs, spawn and track theCG simulations on the important patches, and run the in situ analysis. It interfaces with Flux in the backend.[Link](https://github.com/LLNL/maestrowf)

2) **Flux**:is the resource manager used for MuMMI that allows the workflow manager to break up the allocated nodes in custom, optimized ways. It is designed to be configured and run directly by the user inside of allocated jobs after they are optimally placed on the nodes by the scheduler. Flux assigns the jobs picked out in Maestro to the backend scheduler. MuMMI uses a Maestro plugin for Flux to allow WM’s interface to remain virtually independent of the ongoing development within Flux and to allow the option to switch schedulers in the future. [Link](https://flux-framework.github.io/)

3) **ddcMD**: It is LLNL’s own GPU-accelerated MD software that utilizes the Martini force field and it is faster than competitors such as AMBER, GROMACS, etc. ddcMD is used in by MuMMI in two ways: (1) a CPU-only version of it is used to integrate protein equations of motion in the macro model and (2) a customized GPU
version of it is used for the micro model CG simulations utilizing the Martini force field.[Link](https://github/com/LLNL/ddcMD)

4) **GridSim2D/Moose**: This is the finite element software implementing the equations of motion for the lipids within the dynamic density functional theory framework that is the larger part of the macro model, the other part of which is implemented using a CPU-only version of ddcMD to simulate the protein beads on the lipid membrane, which interact through potentials of mean force. [Link](??)

5) **DataBroker(DBR)**: was investigated for improving data management and I/O operations and allowing fast data storage and retrieval with database-level fault tolerance.

6) **DynIm** is the dynamic importance sampling software that seems to interface with the MuMMI workflow manager (for running inference). [Link](https://github.com/CBIIT/NCI-DOE-Collab-Pilot2-DynIm)
 
7) **MemSurfer** is an analysis tool that is not used within the MuMMI workflow, is an efficient and versatile tool to compute and analyze membrane surfaces found in a wide variety of large-scale molecular simulations. [Link](https://github.com/CBIIT/NCI-DOE-Collab-Pilot2-MemSurfer)

### Requirement for MuMMI

1) Initial macro model parameters (from CG training simulations)
  a) Radial distribution functions (RDFs) are taken from analysis of the Martini MD CG force field parameters and converted to free-energy functionals that are      needed for the macro model
  b) Also needed from the CG simulations: lipid self-diffusion coefficients to get the mobility parameters for the macro model; potentials of mean force; direct    correlation function; self-diffusion coefficients; protein diffusivity; initial protein conformations (this requires 30 CG MD simulations of standard patch      size)
2) Ensure the CG (and macro) simulations are stable
3) Pre-training of model for encoding lipid configurations
4) Protein density on membrane
5) Initial library of protein conformations to sample from during CG simulations
6) Working Martini parameter set, structures of the proteins and lipids
7) Other physical parameters such as CG setup pull-protein-to-membrane speed, cut-off radii, etc.
8) Optimization of analysis routines so that using 3 CPU cores for each simulation it can keep up with the frequency of incoming frames from ddcMD
9) Crystal structure of active proteins in the lipid membrane context
10) Required experimental measurements
11) Biologically relevant membrane compositions and test of its stability in both models
12) Preparation of the lipid bilayer, e.g., lipid spacing in each leaflet
13) Modeled and optimized (minimized + equilibrated) protein
14) CG beads version of the protein structure calculated using martinize.py
15) CG-modeled/parametrized protein with all sanity checks
16) "Extensive sets of CG simulations were carried out in order to validate the behavior of mixed lipid systems with and without RAS, as well as to provide input     parameters for the macro model" resulting in "preliminary CG simulation data" or "CG MD Martini parameterization simulations" or "training data"
    
    These result in these parameters to the macro model:
      a) diffusion coefficients for the different lipids
      b) diffusion coefficients for RAS in the two different orientational states
      c) lipid-lipid correlation functions
      d) potentials for lipid-RAS and RAS-RAS interactions
      e) state change rates for RAS
17) HMM analysis to determine orientational states of the protein
      a) They found RAS is generally in two metastable states in the macro model and three states in the micro model
18) HPO and data augmentation (rotations) on the VAE model to work for the data for the particular biological system
19 Use MemSurfer to perform basic analysis of membrane simulations (e.g., local areal densities) in preparation for creating a macro model from CG MD data

Reference:
Refer to this [article](https://www.researchsquare.com/article/rs-50842/v1)
