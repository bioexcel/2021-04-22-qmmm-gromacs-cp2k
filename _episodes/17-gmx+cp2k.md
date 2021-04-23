---
title: "Practical: GROMACS + CP2K Part III"
teaching: 15
exercises: 60
questions:
- "How to build up QMMM system for protein from the PDB file?"
- "How non-standard CP2K input parameters could be specified?"
objectives:
- "Make protein QMMM system starting from the PDB structure"
- "Usage of non-standard CP2K input parameters"
- "Cacluation of the absorption spectra for your system"
keypoints:
- "QMMM system the protein could be built from the classical MD system directly"
- "Non-standard CP2K parameters could be specified directly in the generated CP2K input file"
- "*qmmm-qmmethod = INPUT* should be used for providing your own CP2K input file"
- "Advanced properties, like absorption spectra could be calculated using external input files"
---

[Slides](../slides/QMMM-Tutorial-EPCC.pdf)

## Preparing for the tutorial

Everything, which is written inside the ```gray box``` are a commands, that should be executed in the terminal window, string-by-string, each following with the ENTER button.  
Please note that `<...>` in the commands means, that everything, including `<>` symbols, must be replaced with your own specific information. Be careful!  

> ## Helpful utilities and commands
>
>Some exercises will require usage of `less` Linux tool for looking up into the content of files. In case you are not familiar with it, here is a short list of hotkeys, which could be used inside LESS editor:
>>* q – exit  
>>* / – search for a pattern which will take you to the next occurrence  
>>* n – for next match in forward  
>>* N (SHIFT+n) – for previous match in backward  
>>* g – go to the start of file  
>>* G (SHIFT+g) – go to the end of file  
>
>All exercises will require you to submit job for computing using `sbatch run.sh` command. To check status of your job following commands would be useful.  
>>* `squeue -u <your login name>` - checks status of all your jobs. Output will look like that:  
>>* `scancel <JOBID>` will remove the job, if you occasionally submitted it.
{: .challenge}
{% include figure.html url="" max-width="80%" file="/fig/05-gmx+cp2k/tutorial-squeue-img.png" alt="squeue output" %}  

## Setting up tutorial environment

Let’s start the tutorial with the following steps  
1. Execute commands in the terminal:  
`module load gromacs-cp2k`  
`cd /work/ta025/ta025/<your login name>`  
2. And go to the tutorial directory  
`cd tutorial`  

## Exercise 5: Setting up simple protein system starting from the PDB file

1) Go to egfp directory:  
`cd egfp`  
{% include figure.html url="" max-width="80%" file="/fig/17-gmx+cp2k/egfp.png" alt="4EUL EGFP" %}
In the directory located forcefield and **4eul.pdb** file with a strucutre of EGFP protein dowloaded from PDB databank.  
Note that file was modified, missing atoms have been added, first and last residues has been marked as N- and C- terminus, periodic box has been added 10 x 10 x 10 nm.  
In addition forcefield for non-standard Chromophore residue CRO66 has been generated with Antechamber.  
You can download it and inspect structure with VMD or PyMOL.  

2) Make topology for the system using the following command:  
`gmx_cp2k pdb2gmx -f 4eul.pdb`  
choose the following forcefield and water model:  
```
Select the Force Field:
From current directory:
1: AMBER03 : Neutral GFP
....
....
Select the Water Model:
 1: TIP3P     TIP 3-point, recommended
```  
Files **topol.top**, **conf.gro** and **posre.itp** should appear in the directory  

3) Solvate te system in the **conf.gro**  
`gmx_cp2k solvate -cp conf.gro -o conf.gro -p topol.top -shell 10`  

4) Now we need to make our system neutral by adding 6 Na+ ions  
To do that first generate tpr file:  
`gmx_cp2k grompp -f em.mdp -p topol.top -c conf -o egfp-genions.tpr -maxwarn 10`  
then use the following command to replace 6 random water molecules with Na+ ions  
`gmx_cp2k genion -s egfp-genions.tpr -p topol.top -o conf.gro -neutral`  
select group 13 of SOL molecules `Select a group: 13`  
after that manipulations your **conf.gro** and **topol.top** files will contain solvated and neutralized protein system.  

5) The next step would be minimization and short classical equilibration NVT trajectory.  
First generate and run energy minimization:
`gmx_cp2k grompp -f em.mdp -p topol.top -c conf -o egfp-em.tpr`  
`sbatch run-em.sh`  
Wait until simulation will be completed.  
Then perform 100 ps NVT simulation starting from the optimized structure to equilibrate our system:
`gmx_cp2k grompp -f md-mm-nvt.mdp -p topol.top -c conf.gro -t egfp-em.trr -o egfp-mm-nvt.tpr`  
`sbatch run-mm-nvt.sh`  
while simulation is running you could check **em.mdp** and **mm-nvt.mdp** files for the details of classical MD simulations  

6) Next step would be changing simulation from classical forcefiled to QMMM.  
First generate index.ndx file that would contain QMatoms group, marking QM atoms in our protein:
`gmx_cp2k make_ndx -f conf.gro`  
and do the following input  
```
> a 938-956
> name 18 QMatoms
> q
```  
Look into the **conf.gro** with VMD or PyMOL and make sure that atoms from 938 to 956 are the same as shown in spheres on the following figure:  
{% include figure.html url="" max-width="80%" file="/fig/17-gmx+cp2k/egfp-qm.png" alt="QM part" %}  

7) Now we are ready to generate QMMM simulation file:  
`gmx_cp2k grompp -f md-qmmm-nvt.mdp -p topol.top -c conf.gro -t egfp-mm-nvt.trr -n index.ndx -o egfp-qmmm-nvt.tpr`  
Here we are using classically equilibrated trajectory **egfp-mm-nvt.trr** as a starting point for QMMM simulation.  

8) Run QMMM simulation:  
`sbatch run-qmmm-nvt.sh`  
While simulation is running you could inspect **md-qmmm-nvt.mdp** and check that QM part in that case has charge -1.  
```
; CP2K QMMM parameters
qmmm-active              = true   ; Activate QMMM MdModule
qmmm-qmgroup             = QMatoms; Index group of QM atoms
qmmm-qmmethod            = PBE    ; Method to use
qmmm-qmcharge            = -1     ; Charge of QM system
qmmm-qmmultiplicity      = 1      ; Multiplicity of QM system
```  

9) At the end of the simulation you can download trajectory file **egfp-qmmm-nvt.trr** and render it using your favorite software (e.g. VMD, PyMOL).  
Also you could check temperature as a function of time with the following command:  
`gmx_cp2k energy -f egfp-qmmm-nvt.edr`  
and choose `16  Temperature`  
File **energy.xvg** will contain data about Temperature (K) against simulation time (ps).  
{% include figure.html url="" max-width="80%" file="/fig/17-gmx+cp2k/egfp-qmmm-temperature.png" alt="EGFP temperature" %}  

## Exercise 6: Using non-standard parameters in CP2K input

1) Stay in the same egfp directory  

2) Copy **egfp-qmmm-nvt.inp** and **egfp-qmmm-nvt.pdb** files:  
`cp egfp-qmmm-nvt.inp egfp-qmmm-spec.inp`  
`cp egfp-qmmm-nvt.pdb egfp-qmmm-spec.pdb`  

3) Modify **egfp-qmmm-spec.inp** file you have just copied with `vim egfp-qmmm-spec.inp` or any other editor.  
Insert between `&END DFT` and `&QMMM` lines an additional `&PROPERTIES` section.  
Final result should look like that:  
```
  &END DFT
  &PROPERTIES
    &TDDFPT
       NSTATES      5
       MAX_ITER    10
       CONVERGENCE [eV] 1.0e-3
    &END TDDFPT
  &END PROPERTIES
  &QMMM
```  
This will order CP2K to also calculate 5 excited states at each MD step with TDDFT.  

4) Generate Gromacs-CP2K simulation file:  
`gmx_cp2k grompp -f md-qmmm-spec.mdp -p topol.top -c conf.gro -t egfp-qmmm-nvt.trr -n index.ndx -o egfp-qmmm-spec.tpr`  

5) Run simulation:  
`sbatch run-qmmm-spec.sh`  

6)While it is running inspect content of **md-qmmm-spec.mdp** file, the following lines will order GROMACS to use external CP2K input file:  
```
; CP2K QMMM parameters
qmmm-active              = true   ; Activate QMMM MdModule
qmmm-qmgroup             = QMatoms; Index group of QM atoms
qmmm-qmmethod            = INPUT  ; Method to use
qmmm-qminputfile         = egfp-qmmm-spec.inp ; external input file
```  

7) After job is finished, we need to gather information about excitation energies over the calculated trajectory:  
`grep " TDDFPT|" egfp-qmmm-spec.out | awk '{ print $3 " " $7 }' > excitations`  
The **excitations** file should appear in the directory, it will consist out of two columns. 
First column is an excitation energy (in eV) and second is an oscillator strength (in a.u.) for each excitation computed by CP2K. 
Final absorption spectra could be convolved by representing each excitation with gaussian function and sum up over all of them.  

8) Convolve the spectra using provided Python script:  
`module load cray-python`  
`./conv.py excitations 0.1 2 5`  
File **spec.xvg** should appear in the directory. You can open it in Grace or copy data into any other software (i.e. Excel).  
As an example, convolved spectra with 0.1 eV half-width gaussians over 100fs (100 steps) trajectory:  
{% include figure.html url="" max-width="80%" file="/fig/17-gmx+cp2k/egfp-spec-100fs.png" alt="EGFP Spectra 100 fs" %}  

9) Spectra collected over 3 ps (3000 MD steps) will look like that:  
{% include figure.html url="" max-width="80%" file="/fig/17-gmx+cp2k/egfp-spec-3ps.png" alt="EGFP Spectra 3 ps" %}  

{% include links.md %}
