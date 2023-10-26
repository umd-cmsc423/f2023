---
type: assignment
date: 2023-10-26T4:00:00+4:30
title: "Assignment #3: FM-index construction and search"
#pdf: /static_files/assignments/asg.pdf
#attachment: /static_files/assignments/asg.zip
#solutions: /static_files/assignments/asg_solutions.pdf
#published: true
due_event: 
    type: due
    date: 2023-11-09T4:00:00+4:30
    description: 'Assignment #3 due'
---

**Due: Thurs Oct 26, 2023**  
**Posted: Thurs Oct 26, 2023**  
**Last updated: Thurs Oct 26, 2023**  

# Overview

This assignment deals with the construction and querying of the FM index.  As we saw in class, having a useful FM index relies upon having (at least sampled) suffix array entries to recall the actual positions of the matching patterns. Thus, this project assumes that you have a working suffix array construction implementation.

If you completed project 2 and implemented your own suffix array construction routines, I strongly encourage you to use those.  However, if you didn't complete project 2 successfully and are therefore in need of an efficient suffix array construction implementation, you can find a starter project in Java [here](https://github.com/umd-cmsc423/project3_java_sa_construction) and in C++ [here](https://github.com/umd-cmsc423/project3_cpp_sa_construction).

Much like project 2, your project will consist of 3 executables `buildfm`, `inspectfm`, and `queryfm` which are described in more detail below.  Though you will implement 3 programs, you can think of the project as being broken into two parts.

In the first part of the project, you will implement a program to read a reference sequence from a `FASTA` file, to construct the suffix array and FM index on this sequence, and then to write the suffix array and FM index to file in a **binary format**.  You will also implement a program to read the saved file from disk, produce a textual representation of the FM index, and write that representation to an output file.

In the second part of the project, you will implement a program to read your serialized index from file, as well as to read an input `FASTA` file containing many queries and an execution mode.  Your program will then produce an output file with the query results in a well-specified output format.

**NOTE** : The timeout for all tests for the gradescope server will be 30 minutes

## Overall structure

You will submit your assignment as a tarball named `CMSC423_F23_A3.tar.gz`.  When this tarball is expanded, it should create a **single** folder named `CMSC423_F23_A3`.  This folder must be created in the directory where the decompression (i.e. `tar xzvf`) is done, and must not be nested inside any other folders. The details of how you structure your "source tree" are up to you, but the following **must** hold (to enable proper automated testing of your programs).

 * There should be a script at the top-level of `CMSC423_F23_A3` called `build.sh`.  This should do whatever is necessary to create 3 executables at the top level (one called `buildfm` and one called `inspectfm` and one called `queryfm`).  If you're comfortable with Makefiles, this can just call `make`, or it could simply run the commands necessary to compile your programs and copy them to the top-level directory.  You can assume this script is run in a `bash` shell.
 
 * There should be a README.md file in the top level directory.  This README file should contain the following information.
     
     - What language have you written your solution in?
     - What did you find to be the hardest part of this assignment?
     - What resources did you consult in working on this assignment (view this as a form of citation; you shouldn't _copy_ code directly from anywhere in your assignment, but if you consulted other sources please list them here).

**Turnin** : As usual, the assignment turnin will be handled using Gradescope.  

## Part (a), constructing and inspecting the 

In this part of the assignment, you will write two programs.  The first program will be called `buildfm`; it will read in a "genome" (in `FASTA`) format, build the suffix array and FM index on this reference, and write the suffix array and FM index to a binary file.  The second program will be called `inspectfm`; it will read in the binary file written by `buildfm`, and then it will compute a textual representation of the FM index and write out a file containing this representation. 

### `buildfm`

### `buildfm`: Input 

The input consists of 2 arguments, given in this order:

* `reference` - the path to a FASTA format file containing the reference of which you will build the suffix array.
* `output` - the program will write a single binary output file to a file with this name, that contains a serialized version of the suffix array and the FM index.

### `buildfm`: Output

Your program will output a file with the name given by the `output` argument above.  This must be a binary file holding everything necessary to perform query using your FM index. Specifically, it should include an encoding of the suffix array, the BWT of the string, the first column of the FM index and the "tally" table used to perform `occ` queries during the backward search procedure.

**Note:** The specific binary encoding is up to you. You _are_ allowed to use an external serialization library for this component, but the serialization must be to a binary (not text) format.  In C++ you could use something like [cereal](https://uscilab.github.io/cereal/) or [bitsery](https://github.com/fraillt/bitsery); in Rust you could use soemthing like [rkyv](https://github.com/rkyv/rkyv) or [serde](https://github.com/serde-rs/serde) with [bincode](https://github.com/bincode-org/bincode).  For reasons mentioned previously (and for the sake of not running into performance issues in future projects), I'd not recommend using Python for this project.

### `inspectfm`
### `inspectfm`: Input 

The input consists of 3 arguments, given in this order:

* `index` - the path to the binary file containing your serialized FM index array (as written by `buildfm` above).
* `sample_rate` - the rate at which "tally table" entries will be sampled in your output (see below).
* `output` - the program will write output to this file containing a textual representation of your FM index.

### `inspectfm`: Output

Your program will output a file with the name given by the `output` argument above.  The output will consist of the following information: 

 * `first column` : A **tab-separated** list of the counts of `$`,`A`,`C`,`G`,`T` in the text (in this order).

 * `BTW(T)` : The Burrows–Wheeler transform of the original text.

 * `tally spot check A` : This is a textual representation of certain values within the tally array for the `A` character.  Specifically, this line should encode a list of **tab-separated** integers that consists of the following entries of the tally array for the character `A` : [`sample_rate`*0, `sample_rate`*1, `sample_rate`*2, ..., `sample_rate`*floor(text-length / `sample_rate`)]. 

 * `tally spot check C` : This is a textual representation of certain values within the tally array for the `C` character.  Specifically, this line should encode a list of **tab-separated** integers that consists of the following entries of the tally array for the character `C` : [`sample_rate`*0, `sample_rate`*1, `sample_rate`*2, ..., `sample_rate`*floor(text-length / `sample_rate`)]. 

 * `tally spot check G` : This is a textual representation of certain values within the tally array for the `G` character.  Specifically, this line should encode a list of **tab-separated** integers that consists of the following entries of the tally array for the character `G` : [`sample_rate`*0, `sample_rate`*1, `sample_rate`*2, ..., `sample_rate`*floor(text-length / `sample_rate`)]. 

 * `tally spot check T` : This is a textual representation of certain values within the tally array for the `T` character.  Specifically, this line should encode a list of **tab-separated** integers that consists of the following entries of the tally array for the character `T` : [`sample_rate`*0, `sample_rate`*1, `sample_rate`*2, ..., `sample_rate`*floor(text-length / `sample_rate`)]. 

## Part (b), querying the FM index

In the second part of the assignment, you will implement a program for querying patterns in the text using your FM index. Your program for this part should be called `queryfm`.  This program will take as input 4 arguments, your serialized
FM index, a `FASTA` file containing queries, a `query mode` parameter, and an `output` file name.  It will perform query in the FM index and report the results in the `output` file specified in the format specified below.

#### `queryfm`: Input

* `index` - the path to the binary file containing your serialized suffix array (as written by `buildsa` above).
* `queries` - the path to an input file in `FASTA` format containing a set of records. **Unlike project 1** you will need to care about both the _name_ and _sequence_ of these FASTA records, as you will report the output using the name that appears for a record.  Note, query sequences can span more than one line (headers will always be on one line).
* `query mode` - this argument should be one of two strings; either `complete` or `partial`. If the string is `complete` you should perform your queries using the backward search algorithm and **only** report hits for a pattern if it matches, in its entirety, in the text.  If the string is `partial` you should perform backward search and return the length and positions of the **longest matching suffix** of the query pattern in the text.
* `output` - the name to use for the resulting output.

#### `queryfm`: Output

* `output` - the output file of your program. This file should contain the results of your queries in the following format.  Each line should contain a **tab-separated** list containing the following information:

    * `query_name`  `match_len` `k` `hit_1` `hit_2` ... `hit_k`

Here, the `query_name` is simply the header of the corresponding FASTA entry (the string after the `>` --- **not including the `>`** on the header line).  The value `match_len` is the length of the match.  **If your program was run with query_mode of `complete`** then `match_len` should be either the full length of the query string (if there were hits) or 0 (if there were no hits).  **If your program was run with query_mode of `partial`** then `match_len` should be the length of the longest mached suffix of the pattern. The value `k` is the number of occurrences of the query string (or the longest matched suffix the query string) in the underlying text on which the FM index is built.  Finally, `hit_1` through `hit_k` are the **positions** in the original text (0-indexed) where the query string (or the longest matched suffix of the query string) occurs.  If you are running in `complete` query mode and a query string does not occur in the text, or if you are running in `partial` mode and the last character in the query does not appear in the text, then you should report `k` = 0, and there will be no `hit_1`, ... etc. entries for that query.
