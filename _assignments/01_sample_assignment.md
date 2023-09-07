---
type: assignment
date: 2023-08-28T4:00:00+4:30
title: 'Assignment #0 - Using the command line and parsing input'
#pdf: /static_files/assignments/asg.pdf
#attachment: /static_files/assignments/asg.zip
#solutions: /static_files/assignments/asg_solutions.pdf
#published: true
due_event: 
    type: due
    date: 2023-09-12T4:00:00+4:30
    description: 'Assignment #0 due'
---

**Due: Tues September 12, 2023**  
**Posted: August 31, 2023**  
**Last updated: August 29, 2023**  

**Note**: A "skeleton" structure for the project (using Java as the underlying language) has been created [here](https://github.com/umd-cmsc423/F23_A0_sample). It shows how to setup the relevant files, and how e.g. one would use `build.sh` to invoke `javac` to compile the source into a `.class` file and how one would create a script named `fasta_stats` to run the class file.

You will implement a program for parsing a FASTA format file, computing some basic statistics about the records it contains, and printing these statistics to `stdout` in `JSON` format. 

As you develop your proejct, I **highly, highly** recommend that you use [`git`](https://git-scm.com/book/en/v1/Getting-Started) for developing your code. If you use a service such as GitHub for hosting your code, 
you should develop your code in a **private** repository.

This program must be written in a _compiled_, statically-typed language (e.g. not Python). However, there is substantially flexibility in which particular language you want to use. The Gradescope image is outfitted with compilers for C, C++, Go, Java and Rust. We will also consider reasonable requests to add other compiled languages.

# Overview

This assignemnt has one component:

A program that parses valid (possibly multi-line) FASTA files, computes certain simple statistics, and writes them to a JSON format output.

## Overall structure

You will submit your assignment as a tarball named `CMSC423_F23_A0.tar.gz`.  When this tarball is expanded, it should create a folder named `CMSC423_F23_A0`.  The details of how you structure your "source tree" are up to you, but the following **must** hold (to enable proper automated testing of your programs).

 * There should be a script at the top-level of `CMSC423_F23_A0` called `build.sh`.  This should do whatever is necessary to create a single executables at the top level called `fasta_stats`.  If you're comfortable with Makefiles, this can just call `make`, or it could simply run the commands necessary to compile your programs and copy them to the top-level directory.  You can assume this script is run in a `bash` shell. The generated `fasta_stats` file must be an _executable_ file that can be invoked directly as `./fasta_stats input_file`.  This means that if you are using a language like `java`, that requries launching a program with a runtime, then `fasta_stats` will likely be a wrapper script that properly invokes the corresponding main java entry point.  For a language like `C`, `C++`, `Rust`, or `Go`, then `fasta_stats` would simply be the compiled binary.
 
 * There should be a `README.md` file in the top level directory.  This README file should contain the following information.
     
     - What language have you written your solution in?
     - What resources did you consult in working on this assignment (view this as a form of citation; you shouldn't _copy_ code directly from anywhere in your assignment, but if you consulted other sources please list them here).

**Turnin** : The assignment turnin will be handled using Gradescope.  

## The FASTA stats program

You will write a program that reads and parses a `FASTA` format file and generates statistics about the records in the file. Your program will take a single command line parameter (the path to the input file).  The output will be written to `stdout`. This also means that you should not write any other messages to `stdout` — if you need to write diagnostic messgages from your program, you should write them to `stderr`. Your program should be called `fasta_stats`.

### Input 

The input consists of 1 argument:

* `input_file` - the path to a `FASTA` format file containing the genome records that you should parse.

Each records in the `FASTA` format consists of 2 parts, a _header_ line (which starts with the `>` character, followed by a record identifier). The first string after `>` in each record header will provide a _unique_ identifier for this record. After this record identifier, there may be zero or more other strings providing extra information. You need not worry about them for the purpose of this project. After the header comes the _sequence_ of the record. The sequence consists of a sequence of one or more lines. Each line consists of a sequence of characters (in this case they will all be `A`, `C`, `G` or `T`). The _sequence_ for a record may span more than one line, in which case the total sequence associated with this record is **the concatenation of all sequence lines, removing any occurrences of the `\n` character**.  There is no _a priori_ limit on the length of lines, or the length of the sequence string they encode.  This means that your parser should implement the appropriate logic when reading a record (i.e. scanning and concatenating the sequence for a record until it encounters the end of the file or the next header).  The `FASTA` file itself, then, just consists of a sequence of such records.

### Output 

#### Statistics

Your program should write to `stdout` a JSON format object (string) with the following information (using the following keys, each associated with the proper value):

* `min_len` - The length of the shortest _sequence_ observed for any record (this is the total sequence length, which may comprise multiple input lines if this is a multi-line record).
* `max_len` - The length of the longest _sequence_ observed for any record (this is the total sequence length, which may comprise multiple input lines if this is a multi-line record).
* `mean_len` - The average length of sequences observed for the records in the input file.
* `tot_len` - The total length of sequences observed for the records in the input file.
* `num_records` - The total number of input records observed in the input file.
* `count_a` - The total number of occurrences of the nucleotide `A` appearing in this file.
* `count_c` - The total number of occurrences of the nucleotide `C` appearing in this file.
* `count_g` - The total number of occurrences of the nucleotide `G` appearing in this file.
* `count_t` - The total number of occurrences of the nucleotide `T` appearing in this file.

This information should be written as a properly-formatted `JSON` object to `stdout`.
