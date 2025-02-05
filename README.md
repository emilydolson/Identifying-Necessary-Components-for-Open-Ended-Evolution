# The MODES toolbox: Measurements of Open-ended Dynamics in Evolving Systems
[![DOI](https://zenodo.org/badge/151119818.svg)](https://zenodo.org/badge/latestdoi/151119818)

This repository contains code, analysis, and data for the paper 
"The MODES toolbox: Measurements of Open-ended Dynamics in Evolving Systems"
by [Emily Dolson](emilyldolson.com), [Anya Vostinar](https://vostinar.sites.grinnell.edu/), 
[Michael Wiser](https://msu.edu/~mwiser/) and [Charles Ofria](ofria.com). In this paper, we present 
a toolbox of measurements for quantifying hallmarks of open-ended evolution and test them in two systems:
NK Landscapes and Avida.

## Links

- Preliminary versions of the ideas in this paper are discussed in [this blog post](https://thewinnower.com/papers/2309-what-s-holding-artificial-life-back-from-open-ended-evolution) 
and [this paper presented at the second workshop on Open-ended evolution](http://www.tim-taylor.com/oee2/abstracts/vostinar-oee2-submission.pdf).

- Supplemental graphs are available here: [https://emilydolson.github.io/MODES-toolbox-paper/analysis/OEE.html](https://emilydolson.github.io/MODES-toolbox-paper/analysis/OEE.html)

- The version of Avida used for these experiments is here: [https://github.com/emilydolson/avida-empirical](https://github.com/emilydolson/avida-empirical)

- Some of the NK Landscape code and the MODES toolbox itself are inside the [Empirical library](https://github.com/devosoft/Empirical).
The precise version of Empirical used in this paper is here: [https://github.com/emilydolson/Empirical/tree/OEE_metrics_paper_submission](https://github.com/emilydolson/Empirical/tree/OEE_metrics_paper_submission).

## Contents of this repository

- **Paper**: This where all of the latex code for the paper itself lives
- **Data**: This directory contains two csv files, one containing the nk landscape data (`nk_data.csv`) and one containing the Avida data (`avida_data.csv`). It also contains a markdown file explaining the columns.
- **Analysis**: This directory contains an R-markdown file with all of the analysis code (and a little commentary), a version of the R-markdown file rendered to html (linked to above as supplmentary material), and a script for making flat violin plots (required for the rain cloud plots).
- **Figs**: This directory contains the figures generated by the code in `analysis` so that the code in `paper` can access them.
- **Code**: This directory contains data analysis scripts and code for the NK Landscape experiments

## Tutorial on using our C++ implementation of the MODES Toolbox

Currently, our C++ implementation of MODES lives in [Empirical](https://github.com/devosoft/Empirical), a library of tools for writing scientific software (particularly computational evolution). Although Empirical has support for building Artificial Life systems from scratch (see our [NK Landscape code](https://github.com/emilydolson/MODES-toolbox-paper/blob/master/code/source/nk_oee.h) for an example), you can also integrate MODES into your existing software (see our [integration of MODES into Avida](https://github.com/emilydolson/avida-empirical) for an example). Here are the steps:

### Adding MODES to an existing system

Two add MODES to an existing system, there are two things you need to set up: the systematics manger (which tracks the tree of parent-child relationships over evolution) and the MODES tracker (which tracks the MODES metrics themselves). Because this implementation of MODES only supports filtering by lineage persistence (owing to the difficulty of a generalizable technique for setting up a shadow run), the MODES tracker relies heavily on Empirical's systematics manager. If there is sufficient demand, we may implement a version in the future that can interface with an arbitrary systematics manager. However, we would urge caution with this, as subtle bugs in a systematics manager can wildly throw off the persistance filter (if anyone was wondering why it took us 3 years to write a paper where we used these metrics in Avida, that's why).

#### Step 1. Add a systematics manager

To add a systematics manager, you need to include the `Evolve/Systematics.h` header, create a systematics manager object, and then set things up to ensure that the systematics manager is notified whenever an organism is born or dies.

Systematics managers take two template arguments: the type of organisms that live in your world and the type of the taxonomic unit you want to track. For instance, you may want to keep a phylogeny of every individual (e.g. microbe A gave birth to microbe B), every genotype (e.g. genotype B arose via a mutation in genotype A), every phenotype, or something else. When you make a systematics manager, you tell it how to determine an individual's taxonomic unit by passing a function to the constructor:

```
#include "Evolve/Systematics.h"  // Include the systematics header

struct MyOrg {
  int id; // Assume each org is assigned a unique identifier at birth
  std::string genotype; // Assume the genotype is a string
};

emp::Systematics<MyOrg, int> individual_systematics([](const MyOrg & org){return org.id}); // A systematics manager that tracks each individual
emp::Systematics<MyOrg, std::string> genotype_systematics([](const MyOrg & org){return org.genotype}); // A systematics manager that tracks each genotype
emp::Systematics<MyOrg, int> phenotype_systematics([](const MyOrg & org){return org.genotype.size()}); // Let's pretend the length of the genotype is a relevant component of the phenotype.
```

The most challenging part is to correctly set up the systematics manager to track births and deaths.

TODO

### Using MODES with a system built in Empirical
(note: Empirical is still in beta, so the interface between MODES and the other Artificial Life building tools may change)

#### 1. Include the MODES code

It lives in a file called OEE.h (for open-ended evolution) in the `Evolve` directory in `source`. Empirical is header-only, so all you need to do to get started using it is include the relevant file:

```
#include Evolve/OEE.h  // make sure to tell your compiler how to find this file by compiling with -Ipath/to/Empirical/source
```

#### 2. Setup your system

To set-up an artificial life system with Empirical, the first thing we need to do is create a `World` object. This object keeps track of most aspects of an evolving population. It is templated off of the type of organism that lives in the world. For the rest of this example, let's pretend we're trying to evolve higher numbers in a population of integers.

```
#include "Evolve/World.h"

int main() {
  emp::World<int> my_int_world;
}
```

Once we've created our world, we need to give it a little more information. Most importantly, we should tell it how to calculate fitness and perform mutations. Fitness functions always take a reference to whatever type of organisms live in the world as input and return a double:

```
std::function<double(int &)> fit_fun = [](int & org){ return org; }; // We're trying to evolve high numbers so the org is its fitness
my_int_world.SetFitFun( fit_fun ); // Tell world to use the fitness function as its fitness function
```

Mutation functions take a reference to an organism and a random number generator as arguments and return a count of the number of mutations that occurred:

```
  std::function<size_t(int &, emp::Random &)> mut_fun =
    [](int & org, emp::Random & random) {
      int num_muts = 0;
      if (random.P(.05)) { // Increment the value of org 5% of the time
        org++;
        num_muts++;
      }
      return num_muts;
    };
  my_int_world.SetMutFun( mut_fun ); // Tell the world to use this mutation function
  my_int_world.SetAutoMutate(); // Tell the world to mutate automateically on reproduction (there are various more specialized controls you can use here if necessary)

```
Some other world setup things we might want to do include setting the population structure and whether generations overlap with each other or not (check out the SetPopStruct_* family of methods).

Now that the world is set up, we need to initialize the population:

```
for (int i = 0; i < 1000; i++) {
  pop.Inject(0); // Inject adds an individual without a parent. We'll start out with a bunch of 0s for simplicity
}
```

And last we need to set up a loop that runs evolution:

```
for (int gen = 0; gen < 1000; gen++) { // Run 1000 generations
    TournamentSelect(my_int_world, 2, 100); // Tournament selection takes a world, a tournament size, and a number of rounds of tournament selection to complete. It handles the resulting reproduction.
    my_int_world.Update(); // Tell the world that a time step has passed.
  }
```

#### 3: Add a systematics manager

Because phylogeny is critical to the persistent lineage filter calculations, the MODES tracker relies heavily on the systematics manager.Thus, we need to add one to the world:

```
  auto sys_ptr = pop.AddSystematics(calc_taxon);

```
TODO
