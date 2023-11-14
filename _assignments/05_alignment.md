---
type: assignment
date: 2023-11-14
title: "Project 5: Pairwise Alignment"
due: 2023-12-02 11:59:00
published: true
---


# CMSC 423 Project 5 (pairwise alignment) : Overview

This assignment deals with pairwise alignment of strings.  We covered many different variants of the alignment problem in class.  
For this project, we will be concerned only with 2; **global** alignment and **fitting** alignment.
Throughout this document (and in the project), we will follow the convention that the string X is considered the _query_ while Y is considered the _reference_.

Unlike projects 1-3, you will implement only 1 program for this assignemnt.  However, it will have 2 distinct modes, and these will be graded separately.

In the first part of the project, you will implement global alignment, with _linear_ gap costs (the simplest) between pairs of strings.  You will compute both the score of the optimal alignment between the strings as well as the edits that yield this optimal score. You will write down the optimal alignment in terms of an **extended CIGAR string** which is explained below.

In the second part of the project, you will implement a fitting alignment, with _linear_ gap costs (the simplest) between pairs of strings.  This will compute everything computed by your global alignment, except that your program must also determine where on the reference (Y string) the optimal alignment _starts_ and where on the reference (Y string) the optimal alignent _ends_.  In the fitting alignment, gaps that occur in Y before X are free and gaps that occur in Y after X are free.  Rather than explicitly represent these in the CIGAR string, you will instead record in your output the positions in Y that are aligned to X.

**NOTE** : The timeout for all tests for the gradescope server will be 30 minutes

## Overall structure

You will submit your assignment as a tarball named `CMSC423_F23_A4.tar.gz`.  When this tarball is expanded, it should create a **single** folder named `CMSC423_F23_A4`.  This folder must be created in the directory where the decompression (i.e. `tar xzvf`) is done, and must not be nested inside any other folders. The details of how you structure your "source tree" are up to you, but the following **must** hold (to enable proper automated testing of your programs).

 * There should be a script at the top-level of `CMSC423_F23_A4` called `build.sh`.  This should do whatever is necessary to create 1 executable at the top level called `saligner`.  If you're comfortable with Makefiles, this can just call `make`, or it could simply run the commands necessary to compile your programs and copy them to the top-level directory.  You can assume this script is run in a `bash` shell.
 
 * There should be a `README.md` file in the top level directory.  This README file should contain the following information.
     
     - What language have you written your solution in?
     - What did you find to be the hardest part of this assignment?
     - What resources did you consult in working on this assignment (view this as a form of citation; you shouldn't _copy_ code directly from anywhere in your assignment, but if you consulted other sources please list them here).

**Turnin** : The assignment turnin will be handled using Gradescope. The Gradescope submission and autograder for this project should be visible on the course Gradescope website. 

## Input 

Since there is only one program being written in this project, the input will be unfiorm over all invocations of the program.  Therefore, we describe the input here.  Your program `saligner` should take 5 arguments, given in this order:

* `input_file` - the path to an input file of string pairs in the format specified below.
* `method` - the type of alignments to be computed, this must be either the string `global` or the string `fitting`.
* `mismatch_penalty` - this is a positive integer `m`, such that the score of a mismatch in an alignment will be `-m`.
* `gap_penalty` - this is a positive integer `g`, such that the score of each gap character inserted into the alignment will be `-g`.
* `output_file` - the path to the file where your output will be written.

### Input pair format

The `input_file` provided to your program will have a collection of `ProblemRecord`s, and your program should align the pair of strings in each `ProblemRecord` and produce output in the specified format below. Each `ProblemRecord` consists of _exactly_ 3 lines in the input file and is of the format:

```
problem_name
X
Y
```

where `problem_name` is simply an arbitrary (unique) string that identifies this alignment problem and `X` and `Y` are strings over the DNA alphabet (`A`,`C`,`G`,`T`).  For example, here is a `ProblemRecord`:

```
p-0
GGTGGAGATACGGGCTCCAA
CGTGGAGGTACGGGCA
```

The `problem_name` is `p-0`, `X` is `GGTGGAGATACGGGCTCCAA` and `Y` is `CGTGGAGGTACGGGCA`.

The input file will simply consist of a collection of such records, written back to back in the file.  So, for example, if there are 100 `ProblemRecord`s in the input file, there will be 300 lines.

### Output format

The output written by your program will consist of a set of `OutputRecord`s.  The `OutputRecord`s **must appear in the output file in the same order as the `ProblemRecord`s in the input file**.  Each `ProblemRecord` in the input will yield exactly one `OutputRecord` in the output. Each `OutputRecord` is exactly 4 lines long, and it has the following format:


```
problem_name
X
Y
score y_start y_end  CIGAR
```

The first 3 lines of the `OutputRecord` are simply an identical copy of the `ProblemRecord`.  All of the information you compute is included in the 4th line of the `OutputRecord`, which is a **tab-separated** line.  This line contains 4 pieces of information.

* `score` - this is the optimal score as computed by your dynamic program
* `y_start` - this the location on the reference string Y where the alignment begins. **For global alignments, this is always 0** and **for fitting alignments it may be greater than 0**.
* `y_end` - this is the location on the reference string Y where the alignemnt ends. For global alignments, this is always the length of Y, for fitting alignments it may be less than the length of Y.
* `CIGAR` - this is the **extended CIGAR** string that specifies the edits that must be made to transform the refernce sequence Y into the query sequence X.  The format of this string is described below.

_Crucially_, your **CIGAR** string must be (1) consistent with your reported score (i.e. `-m` x mismatches + (`-g` x (insertions + deletions)) = score) and (2) walking the strings X and Y and applying the appropriate **CIGAR** operations in order must produce matching strings.
An example of an `OutputRecord` is :

```
p-0
GGTGGAGATACGGGCTCCAA
CGTGGAGGTACGGGCA
-14	0	16	1X6=1X6=3I1=1I1=
```

This was computed with `m` = 1 and `g` = 3, so the score of a mismatch is -1, the score of a gap is -3, and the score of a match is 0.  This result was computed in `global` mode.  This output tells us that the best scoring alignment between X and Y is -14. Because this is a global alignment, the alignment starts at position 0 of Y and ends at position 16 of Y (Y's length).  Finally, the **CIGAR** string is `1X6=1X6=3I1=1I1=`.  This explains how `Y` can be transformed into `X`.  We explain the **CIGAR** string format below using this example.


#### The CIGAR string

We will consider our working example that matches the string `X=GGTGGAGATACGGGCTCCAA` to the string `Y=CGTGGAGGTACGGGCA` at a score of -14 with the CIGAR string `1X6=1X6=3I1=1I1=`.  The CIGAR string is a list of pairs of `(count, operation)` where `count` is an integer >= 1 and `operation` explains what operation follows in the alignment.  The CIGAR string here consists of the following list of pairs:

`(1,X), (6,=), (1,X), (6,=), (3,I), (1,=), (1,I), (1,=)`

The CIGAR string itself is simply these numbers and characters concatenated together.  The potential operations that can be contained in a CIGAR string are:

* `=` - in the alignment the corresponding nucleotides in X and Y match
* `X` - in the alignment the corresponding nucleotides in X and Y mismatch
* `D` - in the alignment the next nucleotide is **deleted from the reference** (i.e. deleted from Y, which is equivalent to having this nucleotide inserted into X)
* `I` - in the alignment the next nucleotide is **interted into the reference** (i.e. inserted into Y, which is equivalent to having this nucleotide deleted from X)

So, let's visualize this with our strings:

```
GGTGGAGATACGGGCTCCAA
X======X======III=I=
CGTGGAGGTACGGG---C-A
```

Here, we have visually represented `I` with the gap character `-` in the `Y` string.  Likewise, a `D` would correspond to placing the gap character `-` in the `X` string.  Under this alignment, we can see that all characters that line up with a `=` match, all character that line up with a `X` mismatch and would match if we copied over the corresponding characeter from `X` to `Y`, and if we were to replace each `-` with the corresponding characters in `X`, then the string `Y` would be transformed into `X`.

In your dynamic program, you should obtain the CIGAR string by building it from right-to-left, as you perform the backtracing in your dynamic programming table.

#### An example fitting alignment

Until this point in the specification, we've been focused on global alignment,
where all of `X` must align against all of `Y`.  If the mode `fitting` is
passed to your program, then you should comptue a `fitting` alignment,
where gaps are allowed "for free" before the start of `X` and after the
end of `X` in the alignment.  In this case, the starting and ending
positions in `Y` need not be 0 and |Y| respectively.  Below is an example
of an `OutputRecord` showing the fitting alignment between `X` and `Y`
under the same scoring function we discussed above.

```
p-3
CACCCTAGTGGAGAAGACCG
TCCACCCTAGTGGATATCCAGAAGACCCCCTTGTT
-7	2	21	12=1I1X1=2X1=1X1=
```

This tells us that the alignment between `X` and `Y` starts at index 0 in `X` but index 2 in `Y` and ends at the end of `X` and at position 21 in `Y`.  Visualizing the CIGAR string as above we have:


```
  CACCCTAGTGGAGAAGACCG
  ============IX=XX=X=
TCCACCCTAGTGGA-TATCCAGAAGACCCCCTTGTT
```

Here, you can see that we have "slid" `X` optimally along `Y` to find the best alignment that is global with respect to `X` and local with respect to `Y`.  The score of this alignemnt is -7 as it has 1 gap and 4 mismatches.  Given this is a fitting alignment, there is no score deduction for the unmatched bases of `Y` before `X` or the unmatched bases of `Y` after `X`.

### Grading

The `global` and `fitting` alignment functionality will be graded separately, so it will be possible to obtain partial credit if you implement the global alignment properly but don't finish the fitting alignment.  However, if you have the `global` alignment functionality working, you need only modify the base case of the recurrence and the traceback starting position to obtain the fitting alignment.  **Critically**, to obtain credit for a proper alignment you _must_ be able to write out the CIGAR string — not just compute the score of the best alignment. **However**, the alignment problems will be evaluated independently, so you can obtain partial credit for having correct alignments for some of the problems even if you don't have correct alignments for all of them.
