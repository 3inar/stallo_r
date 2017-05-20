# So you want to use R on stallo

--- 

This is a quick and dirty guide for how I run my long R scripts on stallo. I am
assuming that either your job can somehow be split up or that it is ok to run the
one job for a super-long time but you don't want it doing so on your laptop. I'm
also assuming you are me, so all examples have my username (`einar`) and my paths
in them. You'll of course have to substitute your own.

I'm usually doing resampling-based computations where I'm just doing the same
thing many times. Splitting the job is simple then: instead of doing say B=5000
resamples in one job i can do 500 in ten jobs or even 50 in 500 jobs.

# The two things you need: R script and job script
Stallo uses a system called slurm to manage computations/jobs. The job you want to
run should be described in a **job script**. The job script contains a description 
of your job and instructions for how to start it. [Here is an example job script
based on my own life](https://gist.github.com/3inar/7c937be149b5fe8750f1777c231e52e6)

## The big questions
The `#SBATCH` comments in that script describe the job. Most importantly 
they **describe the resources you need**. For the most part this will be:

* **How many cpus do you need?** Unless you're using a fancy parallel library in R, this
is always 1. I would generally advise against using a parallel library because it 
adds another layer of complexity.
* **How much memory do you need?** I'll explain how to estimate this below
* **How long should the job run for?** Also explained below.
* **How many identical jobs do you want?** This is how I parallelize a repeat job:
simply say that I want to run this same job K times and do B/K resamples in each job.

## Estimating memory and job length
The cost of underestimating resource need is that your job will be killed without warning
once you exceed your estimates. If you find your job shutting down with completely
nonsensical error messages, it has probably been killed by the slurm watchdogs.
The cost of overestimating is less severe: you
risk ending up in a slower queue on stallo.

I **estimate compute time** very naively. Let's say I need to do 5000 resamplings:
I run two resamplings and time them (see [this link](http://stackoverflow.com/questions/6262203/measuring-function-execution-time-in-r) for instructions), and divide this by 2. This gives me a rough estimate, t, of how long one resampling takes.
If I'm doing 10 jobs of 500 resamplings, I know that this job should take 500*t.
Then I add some more time just to be safe. Say multiply by 1.5 or something. Note that **if you have a job that you expect to run for longer than 48h** you have to use the single node partition of the cluster and not the normal one. Do this by adding the following line to your job script:
```
#SBATCH --partition singlenode
```
This will allow your job to run for up to 28 days.

I **estimate memory used** at the same time. R has a function `gc()` that gives 
you some info about memory use. After having estimated time above, i simply run
`gc()` and get something that looks like this:

```
> gc()
         used (Mb) gc trigger  (Mb) max used  (Mb)
Ncells 225019 12.1     460000  24.6   350000  18.7
Vcells 413623  3.2   58398304 445.6 55918408 426.7
```

This output says something about how much memory you're currently using. 
I won't lie, I don't 100% understand this output. What I do is pick the largest 
Mb number (here 445.6), take this as the worst-case estimate, and add on some
extra memory just to be safe. I'd probably set somewhere between 600 and 1000 Mb.
The computers on stallo have 8000 available, I believe.

## Your R script
[This is an example R-script to go with the job script](https://gist.github.com/3inar/f705e2c568a03be9aff14e58f6810948). The job
runs on any available computer. It's important to save the output to somewhere you know
about. In the example file I write to somewhere in my home directory.
As I assume my job will run many times many different places I put a random 
string of text at the end of the filename so that the 
different jobs don't overwrite the results of one another.

# Running it
Having set these things up i run my job by submitting the job script to slurm:
```
	sbatch jobscript.sh
```
Having done this I can look at my jobs with the `squeue` program
```
	squeue -u einar
```

[Here are some convenient slurm commands](https://www.rc.fas.harvard.edu/resources/documentation/convenient-slurm-commands/)

Please note that whichever working directory you have when you submit the job
will be filled with log files, one per duplicate job.
