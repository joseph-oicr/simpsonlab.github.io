---
layout: post
title: Nanopolish v0.2.0
author: jared
draft: false
comments: true
---

This post describes changes I have made to [nanopolish](https://github.com/jts/nanopolish), our HMM-based consensus caller for Oxford Nanopore data. This post can be thought of as a long changelog with background and rationale.

Background and History
----------------------

[Nick Loman](http://nickloman.github.io/), Josh Quick and I started working on nanpore assembly at a hackathon at the Newton Institute in Cambridge. Our initial goal was pretty simple; we wanted to see if we could run [DALIGNER](https://github.com/thegenemyers/DALIGNER) on nanopore data and devise a way to error correct the reads. After a lot of tinkering and "proper bioinformatics" as Nick put it (converting file formats) we were able to run [poa](http://sourceforge.net/projects/poamsa/) on the overlapping reads that DALIGNER found. Taking poa's consensus sequence as the error corrected read improved identity to around 92-93%. Nick was able to get [Celera Assembler](http://wgs-assembler.sourceforge.net/wiki/index.php?title=Main_Page) running on the corrected reads and our assembly became progressively better as Nick and Josh added more data.

Once Nick got a single-contig assembly out of Celera Assembler we turned our attention to improving the accuracy of the final assembly. The consensus sequence that Celera Assembler called off the corrected reads had accuracy of about 98%. We knew that by working with the base-called reads, rather than the raw signal data emitted by the nanopore, we were losing a lot of information. During the winter holidays I started to write code that would use the raw current signal to call a new consensus. My initial exploratory code was in Python as the [poretools](https://github.com/arq5x/poretools) package gave convenient access to the raw signal data encoded in ONT's FAST5 files. I wrote a quick hidden Markov model in Python to calculate the probability of observing a sequence of nanopore signals given an arbitrary sequence. I immediately realized my Python HMM would be far too slow to run on even a bacterial genome so I decided the core algorithms would have to be written in C or C++.

I asked on twitter the best way to call out to a C++ library from Python and received many helpful replies (h/t to Titus Brown, Michael Crusoe and others). I settled on using [ctypes](https://docs.python.org/2/library/ctypes.html) as this bridge between the Python frontend/poretools and the HMM in C++. I was surprised at how easy ctypes makes this - I had Python talking to a prototype C++ library in under an hour. This hybrid Python/C++ solution was just fast enough to make model development and testing possible. We spent the next month or so revising the probabilistic model of the data, developing algorithms to propose candidate consensus sequences and testing them on our E. coli data. Once the model settled we ran it on the single-contig assembly which took a few days running in parallel on Nick's server. We wrote up a [preprint](http://biorxiv.org/content/early/2015/03/11/015552) describing this work and posted it on BioRxiv.

Improving the design
--------------------

I was not satisfied with the Python/C++ hybrid design. I am sensitive to installation issues when releasing software as I have found that installing dependencies is a major source of problems for the user (although great projects like [homebrew-science](https://github.com/Homebrew/homebrew-science) are helping a lot here). I admire Heng Li's software where one usually just needs to run ```git clone``` and ```make``` to build the program. The initial version of nanopolish was far from this ideal as it depended on eight Python libraries that needed to be installed with ```pip```. When moving between my local development version and Nick's server I realized that installing these dependencies often failed. With this in mind I decided to rewrite the Python frontend in C++. To do this, I crucially needed a replacement for poretools which I used to access the raw data. Matei David in my group volunteered to help and wrote an excellent, intuitive C++ [library](https://github.com/mateidavid/fast5) for parsing ONT's FAST5 files. 

There were additional benefits to this rewrite. In the Python version I again used poa to compute an initial multiple alignment and used this to seed the HMM realignment. In the C++ version I discarded this step, removing another dependency, by calculating the initial alignment directly from the BAM file using [htslib](https://github.com/samtools/htslib). This simplification, along with the much faster parsing of the FAST5 files provided by Matei's library, reduced startup time from a few minutes to a few seconds. This has helped me iterate on the code much faster during development.

Improving HMM efficiency
------------------------

Despite writing the HMM in C++ the first version of nanopolish was very slow. After the Python to C++ rewrite I focused on improving run time. During development I use a lightweight header-only [profiler](https://github.com/jts/sga/blob/master/src/Util/Profiler.h) to keep track of where time is being spent in my program. As expected over 90% of the time was spent running the forward algorithm on the hidden Markov model. I used the amazing [perf](https://perf.wiki.kernel.org/index.php/Tutorial) kernel profiler to explore this further. ```perf``` indicated that most time was in the $$\log()$$ and $$\exp()$$ functions. The forward algorithm on HMMs requires summing log-transformed probabilities. The naive way, $$c = \log(\exp(a) + \exp(b))$$, requires two calls to $$\exp$$ and one call to $$\log$$ for every state/emission pair in the HMM. This is very slow and the subject of an entire section in the classic Biological Sequence Analysis text. I remembered reading Sean Eddy's [paper](http://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1002195) on accelerating HMMER3. In this paper Sean describes how the calculation can be improved by using the transformation $$c = a + \log(1 + \exp(b - a))$$ where $$a \geq b$$. On the surface this would only save a single call to $$\exp$$ but Sean goes further by using a table indexed by $$b - a$$ to cache $$\log(1 + \exp(b - a))$$. This method completely removes the $$\exp$$ and $$\log$$ calls in the inner loop of our HMM. After plugging Sean's implementation into nanopolish we immediately had an 8-fold improvement in speed. Thanks to Sean for allowing us to use this as public domain code.

Improving memory layout
-----------------------

The ```perf``` output also indicated that a lot of CPU time was wasted waiting on memory access. To improve cache usage and reduce the amount of data that is transferred over the memory bus, I reduced the precision of the floating point values from 64 bits to 32 bits. At the same time, I changed the memory layout of the nanopore event signals so that data accessed together was located in contiguous memory locations. This change simply interleaved two arrays, one storing event currents and one storing event durations, into a single array of structs. Finally, I pre-computed all of the scalings and transformations that need to be applied to the nanopore events (for example the current signal drifts over time and this needs to be corrected for) to reduce the work in the inner loop of the HMM. 

Summary
-------

This collection of changes reduced the average time spent in the forward algorithm of the HMM from $$3,000 {\mu}s$$ per call to $$278 \mu s$$ per call for 100 input events and a 100bp sequence, an improvment of over 10x. This did not require any changes at the algorithm level, only minor code optimizations. My next goal is to make algorithmic improvements, primarily avoiding testing unlikely candidate sequences in the HMM. Version 0.2.0 of nanopolish is [here](https://github.com/jts/nanopolish).
