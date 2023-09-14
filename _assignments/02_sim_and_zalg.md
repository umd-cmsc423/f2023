---
type: assignment
date: 2023-09-14T4:00:00+4:30
title: "Assignment #1: Simulation and the Z-algorithm"
#pdf: /static_files/assignments/asg.pdf
#attachment: /static_files/assignments/asg.zip
#solutions: /static_files/assignments/asg_solutions.pdf
#published: true
due_event: 
    type: due
    date: 2023-09-28T4:00:00+4:30
    description: 'Assignment #1 due'
---

**Due: Thurs September 28, 2023**  
**Posted: Thurs September 14, 2023**  
**Last updated: Thurs September 14, 2023**  

# Overview

This assignemnt has two components:

(a) A basic sequencing simulator, to help understand the nature of what is "required" to reconstruct a genome.

(b) An implementation of exact string search using the Z-algorithm.

You will implement a simple "perfect" sequence simulator in part (a), that samples reads of a fixed length from along the genome, sampled at _random_ positions.  For simplicity, we will assume that there are no finite sampling effects.  This means you don't have to simulate e.g. starting with multiple clones of the genome, fragmentation, etc. — rather, you can just repeatedly choose random valid positions along the genome, and extract the substring of the appropriate length.  The more interesting / challenging part of component (a) is just computing the statistics for each simulated sampling run.

In part (b) you will implement the Z-algorithm we discussed in class. For a given reference genome and a small sample of (error-free) sequencing reads, you will return the read position as computed by the Z-algorithm, as well as the length of the longest prefix match between the read and genome (that is not a full occurrence of the read).

## Overall structure

You will submit your assignment as a tarball named `CMSC423_F23_A1.tar.gz`.  When this tarball is expanded, it should create a folder named `CMSC423_F23_A1`.  The details of how you structure your "source tree" are up to you, but the following **must** hold (to enable proper automated testing of your programs).

 * There should be a script at the top-level of `CMSC423_F23_A1` called `build.sh`.  This should do whatever is necessary to create 2 executables at the top level (one called `randsim` and one called `zmatch`).  If you're comfortable with Makefiles, this can just call `make`, or it could simply run the commands necessary to compile your programs and copy them to the top-level directory.  You can assume this script is run in a `bash` shell.
 
 * There should be a README.md file in the top level directory.  This README file should contain the following information.
     
     - What language have you written your solution in?
     - What did you find to be the hardest part of this assignment?
     - What resources did you consult in working on this assignment (view this as a form of citation; you shouldn't _copy_ code directly from anywhere in your assignment, but if you consulted other sources please list them here).

**Turnin** : The assignment turnin will be handled using Gradescope.  

## Part (a), a basic sequencing simulator

In this part of the assignment, you will write a program that reads in a "genome" (in `FASTA` format) and generates random sequencing "reads" from it.  In addition to the input genome, your program will take as input 4 parameters, (1) the length of reads to simulate and (2) the sequencing "depth", (3) the overlap required between adjacent reads to form an island, and (4) the "output_stem" that will prefix the output files you write.  Your program will then randomly sample reads of the specified length until the requested depth is achieved.  In addition to outputting the reads, it will also output some basic statistics about what was sequenced, and how. **Note**: To check that your implementation is performing reasonably, you may want to compare your statistics under different runs to the Lander-Waterman statistics we covered in class.  Are you getting _roughly_ the expected number of islands over many samples given the parameters?

Your program for this part should be called `randsim`.

### Input 

The input consists of 5 arguments, given in this order:

* `read_len` - the length of the simulated reads for you to generate.
* `target_depth` - the target depth of sequencing.  In other words, you should generate the smallest number (m) of reads for which (`m` * `read_len`) / `genome_length` >= `target_depth`.
* `theta` - the $$\theta$$ value we discussed in the Lander-Waterman statistics (the fraction of the read length that a pair of reads must overlap to constitute an island). This input is only necessary for including the number of islands in the output statistics and has no bearing on how reads are simulated.
* `genome` - the path to a FASTA format file containing the genome from which you will simulate reads.
* `output_stem` - the program will write two output files, one called `output_stem.fa` and the other called `output_stem.stats`. To be clear, this argument is an arbitrary string that will be the file name prefix (not the literal string `output_stem`).

### Output 

#### Simulated Reads

Your program should output the simulated reads in "FASTA" format.  For each read, the header should be of the format `>read_identifier:start_position:length`, where the `read_identifier` is just a unique serial identifier for the read (numbered starting at 0), the `start_position` is the leftmost position on the underlying genome that is covered by the read, and the `length` is just the length of the read (same as the input parameter).  These reads should be written to a file called `output_stem.fa`.

#### Statistics

Your program should output a JSON format file with the following information (using the following keys, each associated with the proper value):

* `num_reads` - number of reads generated by the simulation run
* `bases_covered` - total number of distinct nucleotides of the input covered by at least 1 read
* `avg_depth` - the average coverage (this is just `num_reads` * `read_length` / `genome_length`, and should closely match the input parameter of requested depth)
* `var_depth` - the variance of the per-nucleotide depth across the genome; this is just $$s^2 = \frac{\sum_{i=1}^{n} (x_i - \bar{x})^2}{n-1}$$, where $$x_i$$ is the number of reads covering nucleotide $$i$$ in the genome, and $$\bar{x}$$ is just the `avg_depth`.
* `num_islands` - this is the number of distinct contiguous substrings that are covered some set of reads overlapping by at least $$\theta$$ fraction of their length.  In other words, if every nucleotide of the genome is covered by at least one read, such that a read overlaps by $$\theta \ell$$ with the next simulated read, then there is one island.  If there is a single uncovered gap (regardless of it's length), or a region where a subsequent pair of reads overlap by $$< \theta$$, there are 2 islands.  If there are 2 uncovered gaps, there are 3 islands, etc.

This should be written to a file called `output_stem.stats`.

## Part (b), query with Z-algorithm

In the second part of the assignment, you will implement the Z-algorithm we discussed in class.  Your program will take as input two files; the first file containing a reference genome (a _long_ string or "text") in `FASTA` format, and the second file containing a sequence of short query strings (sequencing reads) also in `FASTA` format. For each query sequence in the second file, your program will apply the Z-algorithm to determine where the read occurs in the genome, and it will also examine the resulting Z-array to find the length of the longest prefix shared between the query and the genome **that is not a full occurrence of the query** (that is, the largest value in the Z-array that is less than the length of the query).

To "test" that your implementation finds the right position, you can make use of the simulator you built in part (a).  Specifically, you can use the simulator to simulate a small number of reads from a known genome, ensure that you find the right positions for these reads in your `zmatch` program.  

Your program for this part should be called `zmatch`.

#### Input

* `genome` - the path to an input file in `FASTA` format containing a single long reference string (the genome).
* `reads` - the path to an input file in `FASTA` format containing a collection of records (much shorter strings than the genome, each one is a query).

#### Output

* `output JSON file (written to stdout)` - Your program should write a single `JSON` object to stdout.  This `JSON` object should have two entries, both of which will have values that are **arrays** of numbers.  Each of these arrays will contain _precisely_ `m` elements, where `m` is the number of records in your `reads` file. In other words, each of the two arrays will have one entry per record.  The first array should be called `"occs"`, and it should list the offset (0-based) in the reference string (genome) where each query occurred.  Note, the records in the `reads` file should be processed in order, and the offsets in the `"occs"` array are expected to correspond to the reads in the order in which they appear in the input file.  The second record in your `JSON` output should be called `"longest_prefs"`, and it should also be an array of length `m`.  Each entry in this array should record the length of the _longest prefix of each query that doesn't constitute a full occurrence_. In other words, what is the length of the longest matching prefix of each query that is strictly less than the full query length.  When you compute the Z-array for each matching problem, this will simply be the largest value in the Z-array with length less than `|q|` (the length of the query `q`). As with `"occs"`, the entries in the `"longest_prefs"` array are expected to correspond to the reads in the order in which they appear in the input file.
