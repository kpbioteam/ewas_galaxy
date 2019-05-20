[![Build Status](https://travis-ci.org/kpbioteam/ewas_galaxy.svg?branch=master)](https://travis-ci.org/kpbioteam/ewas_galaxy)


# 🔬📚 Galaxy EWAS Tools Wrappers

This repository contains EWAS tools that can be installed and used inside the Galaxy. Tools are already included in the [!Galaxy Tool Shed](https://toolshed.g2.bx.psu.edu/view/kpbioteam/ewastools/53aaf097238c) and ready to use via [!Docker](https://galaxyproject.org/use/ewas-galaxy/).

Analysis pipline includes raw intensity data loading .idat preprocessing, optional normalisation of the data and quality control with additional sample check. It gives the user the opportunity to perform any of these preparation and data cleaning steps, including the highly recommended genetic variation annotation step resulting in single nucleotide polymorphism identification and removal by simply running the tools. Finally, the dataset generated through all of these steps can be used to hunt (find) differentially-methylated positions DMP and regions DMR with respect to a phenotype co-variate.
