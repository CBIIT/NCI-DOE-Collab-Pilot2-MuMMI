# NCI-DOE-Collab-Pilot2-MuMMI

### Description
Multiscale Machine-learned Modeling Infrastructure (MuMMI) is a methodology that the Pilot 2 team has developed to study the interaction of active KRAS with the plasma membrane (and related phenomena) on very large temporal and spatial scales. To achieve this, MuMMI cleverly connects a macro model of the system to a micro model using machine-learning-based “dynamic importance sampling,” implementing the workflow on world-class supercomputing resources. Given the team’s impressive results, it is reasonable to ask whether MuMMI could be applied to a different biological system consisting of a new lipid bilayer into which a new type of protein is embedded.

MuMMI connects biological models of the membrane-protein system on two different scales:

1. A macro model based on the classical approximation theory for liquids in which 300 proteins (Modeled as single-particle beads) move around on a 1x1 &mu;m^2 cross section of a perfectly flat (2D) plasma membrane.

2. A micro model that uses Martini coarse-grained (CG) molecular dynamics (MD) simulations to model a 30nm x 30nm “patch” of the macro model that necessarily contains at least one RAS protein



### Software workflow


### Suite Components


1) Maestro Workflow Conductor

2) Flux

3) ddcMD

4) GridSim2D/Moose

5) DataBroker(DBR)

### Requirement for MuMMI

