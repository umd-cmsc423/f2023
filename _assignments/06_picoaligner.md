---
type: assignment
date: 2023-12-05T0:00:00+5:00
title: "Assignment #5: Picomap"
due_event: 
    type: due
    date: 2023-12-12T11:59:00+5:00
    description: 'Assignment #5 due'
---

# CMSC 423 Project 5 (the picomapper) : Overview

**Note**: The sample data (and project skeleton) for project 5 are available [here](https://github.com/umd-cmsc423/F23_A5_sample).

This project is a synthesis of the previous projects that you've worked on this semester.  You will put together your capability for exact matching (as performed via either the suffix array or the FM-index) with your ability to perform fitting alignment to build a small proof-of-concept read aligner to align sequencing reads against target genomes.  In addition to the ability to look up exact strings and perform alignment, the main novel part of this project is to implement a seeding and filtering heuristic to decide _where_ you should score reads for alignment.  Specifically, when your seed is highly-specific, and there is only one (or a couple) of potential mapping locations for a read, you can test an alignment at all such locations.  However, if your seed appears in many locations in the genome, then you will generally want to restrict the set of potential mapping locations further by looking up more seeds from the read and filtering out the set of candidate locations that explain all of them.

For this project, you you will implement 2 programs (one to build an index and one to align reads), but the grading will be based only on the alignment program.  For the indexer, you are free to use _either_ the suffix array or the FM-index.  

The index will be used for efficient seed lookup in the genome, to determine candidate locations where you will perform full read alignment.  Given the index and a set of reads, you will perform seed search for each read, looking up exact matches shared between the read and the reference.  Once you have determined a set of candidate locations where the read _may_ align, you will use your fitting alignment function to determine the optimal scoring alignment at each location, and you will report the location, score, and CIGAR string for the optimal alignment for each read.  In the rare event that there are multiple locations that yield an equally optimal alignment for the read, you will report all such locations.

**NOTE** : The timeout for all tests for the gradescope server will be 30 minutes

## Overall structure

You will submit your assignment as a tarball named `CMSC423_F23_A5.tar.gz`.  When this tarball is expanded, it should create a **single** folder named `CMSC423_F23_A5`.  This folder must be created in the directory where the decompression (i.e. `tar xzvf`) is done, and must not be nested inside any other folders. The details of how you structure your "source tree" are up to you, but the following **must** hold (to enable proper automated testing of your programs).

 * There should be a script at the top-level of `CMSC423_F23_A5` called `build.sh`.  This should do whatever is necessary to create 2 executables at the top level called `picoindex` and `picomap`.  If you're comfortable with Makefiles, this can just call `make`, or it could simply run the commands necessary to compile your programs and copy them to the top-level directory.  You can assume this script is run in a `bash` shell.
 
 * There **must** be a **README.md** *file in the top level directory.  This README.md file should contain the following information.
     
     - What language have you written your solution in?
     - What did you find to be the hardest part of this assignment?
     - What resources did you consult in working on this assignment (view this as a form of citation; you shouldn't _copy_ code directly from anywhere in your assignment, but if you consulted other sources please list them here).

**Turnin** : The assignment turnin will be handled using Gradescope.  

## Picoindex

The `picoindex` command should take 2 command line arguments:

### Input 

* `reference_genome` - the path to an input FASTA file containing the reference genome that should be indexed.
* `output_file` - the path to the location of an output file where your index should be written.

### Output 

The only output of `picoindex` should be your serialized index written at the location provided by `output_file`.  Since we are providing the flexibility to use either a suffix array or FM-index for this project, we are not making any specific assumptions about the information contained in your index or the layout of that information.  There is no graded component of the project that relies on inspecting your index.  However, as with projects 2 and 3, you should be sure to serialize your index in a **binary** format, as writing text-based output and having to deserialize a text-based index will be slow, memory intensive, and may result in your program running out of memory or time during testing.  **Finally,** since the `picomap` command will be given only the location of your index and not the location of the original reference genome, you should be sure that your index contains a serialized version of the full reference string.

## Picomap 

## Input 

Your program `picomap` should take 5 arguments, given in this order:

* `index_file` - the path to the index file generated by the `picoindex` command described above.
* `read_file` - a path to a FASTA format file containing a list of sequencing reads, each of which should be aligned against the genome.
* `mismatch_penalty` - this is a positive integer `m`, such that the score of a mismatch in an alignment will be `-m`.
* `gap_penalty` - this is a positive integer `g`, such that the score of each gap character inserted into the alignment will be `-g`.
* `output_file` - the path to the file where your output will be written.

### Output format

The output written by your program will consist of a set of `OutputRecord`s.  Each read in the input file will yield exactly one `OutputRecord` in the output, though an `OutputRecord` may encode more than 1 equally optimal alignment. Each `OutputRecord` has the following format:

```
read_name   num_alignments (k)
reference_start_pos_1   score   CIGAR_1
reference_start_pos_2   score   CIGAR_2
...
reference_start_pos_k   score   CIGAR_K
```

The first line of the `OutputRecord` contains 2 tokens **separated by a tab**.  The first token is simply the name of the input read to which the `OutputRecord` corresponds, and the second token specifies the number of optimal alignments that were discovered for the read (this number will almost always be 1). Let the number of optimal alignments for the record be `k`, then this initial line is followed by `k` `AlignmentRecords`. Each `AlignmentRecord` is simply a **tab separated** line with 3 tokens.  The first token specifies the (0-based) index in the reference where the alignment of the read begins, the second token records the score of the alignment, and the third token records the extended CIGAR string (just as computed in project 4).

Just as with project 4, your **CIGAR** string must be (1) consistent with your reported score (i.e. `-m` x mismatches + (`-g` x (insertions + deletions)) = score) and (2) walking the strings X and Y (starting at the recorded start position) and applying the appropriate **CIGAR** operations in order must produce matching strings.

An example of an `OutputRecord` is :

```
p-0 1
157368  -7  12=1I1X1=2X1=1X1=
```

This was computed with `m` = 1 and `g` = 3, so the score of a mismatch is -1, the score of a gap is -3, and the score of a match is 0.  For details about the meaning of the **CIGAR** string and how it should be formed, please refer to the specification for [the pairwise alignment project](https://umd-cmsc423.github.io/f2023/assignments/05_alignment).

### Finding potential matching loci with seeding

The process of actually aligning the read to the target genome in this project consists of 2 steps.  First, you want to determine a list of **candidate** positions where your read may occur within the target genome.  This is typically done by enumerating the location of "seeds" within the read (substrings of the read that are exactly shared with the reference sequence).  Anywhere a seed occurs is a potential candidate location for alignment.

#### Seed lookup

The strategy in this part of the project is simply to develop an efficient heuristic to search for seeds. Specifically, imagine that you are using the suffix array for your index. One obvious strategy is simply to start searching for an exact match between the prefix of the read and your reference (note that the longest matching prefix is either (a) the positions where the read occurs if the whole read appears exactly in your reference or (b) the last non-empty suffix array interval if only some prefix of the read appears exactly in your reference).  This strategy may work very well for most reads, but if there is a sequencing error or variant near the start of the read, the longest matching prefix may be very short --- in which case aligning the read at all such positions will be expensive --- or even may not correspond to the location from which the read actually derives (if the prefix *containing* the sequencing error is itself present in the reference, it will return candidate locations that do not necessarily match the entire read well).  I have explained this problem with respect to searching the prefix of the read using the suffix array, but an exactly analogous issue occurs if you search for a maximal matching suffix of the read using the FM-index.

Therefore, your seeding strategy needs to be a bit more intelligent.  There are a few different approaches one might take here.  

One such approach is to repeatedly search for the longest matching prefix (or longest matching suffix) for the unmatched portion of the read.  For example, imagine you are searching for the longest matching prefix of a read of length 150 nucleotides, and you find a maximal prefix match of length 10.  Now consider the suffix of the read starting from position 11 and extending until the end.  Perform the same longest-matching prefix search on this suffix to find where its longest matching prefixes occur.  This process can be repeated iteratively until all longest matching prefixes are greedily matched.

Another approach is to consider a set of fixed starting points "tiling" the read.  Again, imagine you have a read `r` of length 150.  Now consider the suffixes `r[0:150]`, `r[25:150]`, `r[50:150]`, `r[75:150]`, `r[100:150]`, `r[125:150]`.  This represents a tiling of the read at intervals of every 25 nucleotides.  Now, you can search for the maximum matching prefix of each of these substrings to find the set of possible locations where the read occurs. Even if the read has errors or variation, one or more of these substrings should still have a long and quite specific match (that occurs in only a few positions).

A third approach similar to the one above is to consider a tiling of the read, but rather than consider the full suffix at each position, to consider only some fixed length substring.  For example, given a read `r` of length 150, consider the strings: `r[0:25]`, `r[25:50]`, `r[50:75]`, `r[75:100]`, `r[100:120]`, `r[125:150]`.  As with above, you can look these up using your index to determine where the read might occur.  These substrings are non-overlapping, but that is also not a requirement, you could extract overlapping substrings from the original read and use them as seeds to look up in your reference.

There are other potential strategies for seed lookup.  Too many to list exhaustively, but I encourage you to start with the strategy that seems to make the most sense to you, and also to explore different strategies to see how efficiently they work in this context.

#### Candidate filtering

Given the type of seeds you use and the locations where those seeds match, it will often be obvious what the candidate locations for the sequencing read should be.  However, one of the main points of looking at multiple seeds is that you can use the agreement among these seeds to filter out locations that are unlikely to be good candidates for an alignment of the entire read.  Consider a read `r` of length 150, where you have looked up the following seeds `r[0:50]`, `r[50:100]`, `r[100:150]`.  Further let these seeds have the following set of mapping locations:

```
r[0:50] : [156325, 295376] 
r[50:100] : [156375, 893421]
r[100:150] : [156425]
```

In this case, the first 2 seeds have multiple mapping locations, and the third seed appears only once in the reference.  Moreover, there is a single location for the read that is supported by all 3 seeds.  If the read started at position `156325`, then the second seed (starting at position 50 on the read) would occur at location `156375` and the third seed (starting at position 100 on the read) would appear at position `156425`.  Thus, here we can conclude that `156325` is a much better candidate for aligning the read than either `295376` or `893371` (`893421` - `50`).

Of course, this is just an example.  It could happen that some of the seeds have *no* valid mapping location (they don't exactly appear anywhere in the reference), or that a set of seeds point to a very similar but not identical location.  For example, what if seeds 1, 2 and 3 occurred at positions `156325`, `156372`, and `156422` respectively.  There is no single position that supported with respect to all three of these seeds, but this is the set of locations we might expect to see if the read actually aligned at position `156325` and there was a small (3 nucleotide) deletion before the second and third seed mapped.  **Thus**, your candidate filtering procedure should look for seeds that support a very similar position on the reference, but they may not always support the same exact location on the reference.  **The maximum distance between the starting positions implied for a read from a set of consistent seeds is bounded above by the largest insertion or deletion in the reference**.  For the purpose of this project, **you may assume that no insertion or deletion is of length >15**.


### Using fitting alignment to solve for the final alignment 

Finally, given your set of one or more candidate locations for each read, you should use your fitting alignment to determine the optimal alignment of the read at this candidate position to the reference.  The purpose of using fitting alignment here rather than global alignment is that, as stated above, insertions or deletions in the reference may occur, and if the insertion or deletion occurs before the first mapped seed, then you may not be able to infer the exact starting location of the read.  Therefore, for each _distinct candidate position_ (each position implied by a set of seeds that do not **overlap** on the underlying genome), you should extract the region of the reference starting upstream of the candidate position by the maximum indel size (i.e. 15 nucleotides before the candidate position) through the position of the reference 15 nucleotides past the implied ending position.  For example, imagine that our seeds suggest that the read starts at position `256515` in the genome, and our read is of length `150`, then extract the substring of the genome from positions `256500` through `256680` and perform a fitting alignment of the read into this substring of the reference.  This will simultaneously give you the precise starting location and the optimal alignment score that this starting location supports.

**Note / tip** : For many of the reads, there will be no gap in the optimal alignment -- if you can infer that the alignment is gapless (e.g. the first and last seed correspond to exactly the same starting position in the reference), than you don't need to extract the flanking region around the candidate reference location before performing alignment.  Likewise, **a large number of reads may map exactly to the reference** (i.e. with no mismatches or insertions or deletions).  In this case, there is obviously no necessity to perform alignment.  For a read of length `M` that matches the reference exactly, the **CIGAR** string will simply be `M=`, and this (and the optimal score) can be determined without explicitly computing any alignment.  If the read maps exactly to the reference with no gaps or mismatches, **your code should avoid solving any alignment problems**, this will speed up your program tremendously on what is likely to be a rather common case.

### Grading

As with project 4, your alignments will be graded on their consistency, correctness, and optimality.  For an `OutputRecord` to be deemed correct, it must align the read at the correct starting location, applying the **CIGAR** string to the read and the reference at the given position must yield identical strings, the reported score must match the operations in the **CIGAR** string, and the reported score must be the _optimal_ alignment score for this read achievable to any substring of the reference.  **However**, just as with project 4, the alignment problems will be evaluated independently, so you can obtain partial credit for having correct alignments for some of the reads even if you don't have correct alignments for all of them.
