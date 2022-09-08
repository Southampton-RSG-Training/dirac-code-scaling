---
title: "Introduction to Job Scheduling"
slug: dirac-code-scaling-introduction-job-scheduling
teaching: 45
exercises: 30
questions:
- "What is a scheduler and why does a cluster need one?"
- "How do I launch a program to run on a compute node in the cluster?"
- "How do I capture the output of a program that is run on a node in the
  cluster?"
objectives:
- "Submit a simple script to the cluster using Slurm."
- "Monitor the execution of jobs using command line tools."
- "Describe the basic states through which a submitted job progresses to completion or failure."
- "Inspect the output and error files of your jobs."
- "Cancel a running job."
keypoints:
- "The scheduler handles how compute resources are shared between users."
- "A job is just a shell script."
- "Use `sbatch`, `squeue`, and `scancel` commandsto run, monitor, and cancel jobs respectively."
- "Request _slightly_ more resources than you will need."
---

## Job Scheduler

An HPC system might have thousands of nodes and thousands of users. How do we
decide who gets what and when? How do we ensure that a task is run with the
resources it needs? This job is handled by a special piece of software called
the _scheduler_. On an HPC system, the scheduler manages which jobs run where
and when.

The following illustration compares these tasks of a job scheduler to a waiter
in a restaurant. If you can relate to an instance where you had to wait for a
while in a queue to get in to a popular restaurant, then you may now understand
why sometimes your job do not start instantly as in your laptop.

![Compare a job scheduler to a waiter in a restaurant]({{ site.url }}{{ site.baseurl }}/fig/restaurant_queue_manager.svg){: width="650px"}

The scheduler used in this lesson is Slurm. Although
Slurm is not used everywhere, running jobs is quite similar
regardless of what software is being used. The exact syntax might change, but
the concepts remain the same.

## Running a Batch Job

### A Basic Script

The most basic use of the scheduler is to run a command non-interactively. Any
command (or series of commands) that you want to run on the cluster is called a
_job_, and the process of using a scheduler to run the job is called _batch job
submission_.

In this case, the job we want to run is a shell script -- essentially a
text file containing a list of UNIX commands to be executed in a sequential
manner. Our shell script will have three parts:

* On the very first line, add `#!/usr/bin/env bash`. The `#!`
  (pronounced "hash-bang" or "shebang") tells the computer what program is
  meant to process the contents of this file. In this case, we are telling it
  that the commands that follow are written for the command-line shell (what
  we've been doing everything in so far).
* Anywhere below the first line, we'll add an `echo` command with a friendly
  greeting. When run, the shell script will print whatever comes after `echo`
  in the terminal.
  * `echo -n` will print everything that follows, _without_ ending
    the line by printing the new-line character.
* On the last line, we'll invoke the `hostname` command, which will print the
  name of the machine the script is run on.

Let's use `nano` to write this script.

```
[yourUsername@login7a [cosma7] ~]$ nano example-job.sh
```
{: .language-bash}
```
#!/usr/bin/env bash

echo -n "This script is running on "
hostname
```
{: .output}

You can then use <kbd>Ctrl-O</kbd> followed by <kbd>Enter</kbd> to save the file,
and <kbd>Ctrl-X</kbd> to exit the editor.

> ## Creating Our Test Job
>
> Run the script. Does it execute on the cluster or just our login node?
>
> > ## Solution
> >
> > ``` bash
> > [yourUsername@login7a [cosma7] ~]$ bash example-job.sh
> > ```
> > {: .language-bash}
> > ```
> > This script is running on login7a
> > ```
>{: .solution}
{: .challenge}

### Submitting the Job

This script ran on the login node, but we want to take advantage of
the compute nodes: we need the scheduler to queue up `example-job.sh`
to run on a compute node.

To submit this task to the scheduler, we use the
`sbatch` command.
This creates a _job_ which will run the _script_ when _dispatched_ to
a compute node which the queuing system has identified as being
available to perform the work.

```
[yourUsername@login7a [cosma7] ~]$ sbatch example-job.sh
```
{: .language-bash}

However, running the script in its current form may yield an error like the following:

~~~
sbatch: error: A valid account is required (use -A and a DiRAC project or Unix group)
sbatch: error: Batch job submission failed: Unspecified error
~~~
{: .output}

In this case, it's telling us that we need to specify more details for submitting it. As it turns out, as a minimum for COSMA we need to specify the following:

- *Our account*: this is the systems budgetary account to which we individually are assigned. You can find out the accounts you have access to by using the `sacctmgr list user yourUsername` command.
- *Partition*: HPC system resources are typically split by type (such as cpu, gpu, large memory, etc.), or some other classification, into partitions. The configuration of these varies from system to system, but essentially are different queues to which you may submit your job. You may only be authorised to use specific partitions/queues.
- *Walltime*: the maximum amount of real-world time your job will take to run, specified as `days-hours:minutes:seconds` (the `days` can be omitted).

Note that depending on the system, other minimal parameters may also be necessary such as specifying a desired quality of service, or the minimum number of required nodes. But we'll leave these for now.

We can specify them on the command line when submitting our job, like the following. Our job is very short running, so let's just give it a maximum wall time of 1 minute:

```
$ sbatch --account=yourAccount --partition=cosma7 --time=00:01:00 example-job.sh
```
{: .language-bash}

```
Submitted batch job 36855
```
{: .output}

And that's what we need to do to submit a job. Our work is done -- now the
scheduler takes over and tries to run the job for us.

### Monitoring our job

While the job is waiting to run, it goes into a list of jobs called the _queue_. 
To check on our job's status, we check the queue using the command `squeue`:

``` bash
[yourUsername@login7a [cosma7] ~]$ squeue -u yourUsername
```
{: .language-bash}

You may find it looks like this:

```
  JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
5791510 cosma7-pa example- yourUser PD       0:00      1 (Priority)
```
{: .language-bash}

We can see all the details of our job, including the partition, user, and also the state of the job (in the `ST` column). In this case, we can see it is in the PD or PENDING state. Typically, jobs go through the following states:

- `PD` - *pending*: sometimes our jobs might need to wait in a queue first before they can be allocated to a node to run
- `R` - *running*: job has an allocation and is currently running
- `CG` - *completing*: job is in the process of completing
- 

> ## Where's the Output?
>
> On the login node, this script printed output to the terminal -- but
> now, when `squeue` shows the job has finished,
> nothing was printed to the terminal.
>
> Cluster job output is typically redirected to a file in the directory you
> launched it from. on DiRAC, for example, the output file looks like `slurm-<job_number>.out`,
> with `<job_number>` representing the unique identifier for the job.
> Use `ls` to find and `cat` to read the file.
{: .callout}

## So what about other job states?


## Customising a Job

In the job we just ran we didn't specify any detailed requirements, such as the
number of cpu cores we need, amount of memory, or number of nodes to use. In a
real-world scenario, that's probably not what we want. Chances are, we will need more cores, more
memory, more time, among other special considerations.

So far, we've specified job requirements directly on the command line which is
quick and convenient, but somewhat limited and less clear when specifying many more parameters.
Plus, we may forget which parameters we used in previous runs, which may be
critical to reproducing a previous result. The good news is that we can amend
our submission script directly to include these parameters instead.

Comments in UNIX shell scripts (denoted by `#`) are typically ignored, but
there are exceptions. For instance the special `#!` comment at the beginning of
scripts specifies what program should be used to run it (you'll typically see
`#!/usr/bin/env bash)`. Schedulers like Slurm also
have a special comment used to denote special scheduler-specific options.
Though these comments differ from scheduler to scheduler,
Slurm's special comment is `#SBATCH`. Anything
following the `#SBATCH` comment is interpreted as an
instruction to the scheduler.

Let's illustrate this by example. First, we'll add the account, partition, and time
parameters to the script directly, then give the job itself a different name. By default,
a job's name is the name of the script, but the `--job-name` (or `-J` for short) option
can be used to change the name of a job. Amend the `example-job.sh` script to look like
the following:

```
#!/usr/bin/env bash
#SBATCH --account yourAccount
#SBATCH --partition cosma7
#SBATCH --time 00:01:00
#SBATCH --job-name hello-world

echo -n "This script is running on "
hostname
```
{: .output}

Submit the job and monitor its status:

```
[yourUsername@login7a [cosma7] ~]$ sbatch example-job.sh
[yourUsername@login7a [cosma7] ~]$ squeue -u yourUsername
```
{: .language-bash}

```
  JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
5791531 cosma7-pa hello-wo yourUser PD       0:00      1 (Priority)
```
{: .language-bash}

Fantastic, we've successfully changed the name of our job!


### Resource Requests

What about more important changes, such as the number of cores and memory for
our jobs? One thing that is absolutely critical when working on an HPC system
is specifying the resources required to run a job. This allows the scheduler to
find the right time and place to schedule our job. If you do not specify
requirements (such as the amount of time you need), you will likely be stuck
with your site's default resources, which is probably not what you want.

The following are several key resource requests:

- `--ntasks=<ntasks>` or `-n <ntasks>`: How many CPU cores does your job need, in total?

- `--mem=<megabytes>`: How much memory on a node does your job need in megabytes? You can also specify gigabytes using by adding a little “g” afterwards (example: `--mem=5g`)

- `--nodes=<nnodes>` or `-N <nnodes>`: How many separate machines does your job need to run on? Note that if you set `ntasks` to a number greater than what one machine can offer, Slurm will set this value automatically.

Note that just _requesting_ these resources does not make your job run faster,
nor does it necessarily mean that you will consume all of these resources. It
only means that these are made available to you. Your job may end up using less
memory, or less time, or fewer nodes than you have requested, and it will still
run.

It's best if your requests accurately reflect your job's requirements. We'll
talk more about how to make sure that you're using resources effectively in a
later episode of this lesson.

> ## Job environment variables
> 
> When Slurm runs a job, it sets a number of environment variables for the job. One of these will let us check what directory our job script was submitted from. The `SLURM_SUBMIT_DIR` variable is set to the directory from which our job was submitted. Using the `SLURM_SUBMIT_DIR` variable, modify your job so that it prints out the location from which the job was submitted.
> 
> > ## Solution
> >
> > ```
> > [yourUsername@login7a [cosma7] ~]$ nano example-job.sh
> > [yourUsername@login7a [cosma7] ~]$ cat example-job.sh
> > ```
> > {: .language-bash}
> > 
> > ``` bash
> > #!/usr/bin/env bash
> > #SBATCH --account yourAccount
> > #SBATCH --partition cosma7
> > #SBATCH --time 00:00:30
> > #SBATCH --job-name hello-world
> > 
> > echo "This job was launched in the following directory:"
> > echo ${SLURM_SUBMIT_DIR}
> > ```
> > {: .language-bash}
>{: .solution}
{: .challenge}

Resource requests are typically binding. If you exceed them, your job will be
killed. Let's use wall time as an example. We will request 1 minute of
wall time, and attempt to run a job for two minutes.

```
[yourUsername@login7a [cosma7] ~]$ cat example-job.sh
```
{: .language-bash}

```
#!/usr/bin/env bash
#SBATCH -J long_job
#SBATCH -t 00:01 # timeout in HH:MM

echo "This script is running on ... "
sleep 240 # time in seconds
hostname
```
{: .output}

Submit the job and wait for it to finish. Once it has finished, check the
log file.

```
[yourUsername@login7a [cosma7] ~]$ sbatch example-job.sh
[yourUsername@login7a [cosma7] ~]$ squeue -u yourUsername
```
{: .language-bash}

```
[yourUsername@login7a [cosma7] ~]$ cat slurm-38193.out
```
{: .language-bash}

```
====
Starting job 5791549 at Thu  8 Sep 16:07:02 BST 2022 for user yourUsername.
Running on nodes: m7231
====
This script is running on slurmstepd: error: *** JOB 5791549 ON m7231 CANCELLED AT 2022-09-08T16:08:28 DUE TO TIME LIMIT ***
```
{: .output}

Our job was killed for exceeding the amount of resources it requested. Although
this appears harsh, this is actually a feature. Strict adherence to resource
requests allows the scheduler to find the best possible place for your jobs.
Even more importantly, it ensures that another user cannot use more resources
than they've been given. If another user messes up and accidentally attempts to
use all of the cores or memory on a node, Slurm will either
restrain their job to the requested resources or kill the job outright. Other
jobs on the node will be unaffected. This means that one user cannot mess up
the experience of others, the only jobs affected by a mistake in scheduling
will be their own.

## Cancelling a Job

Sometimes we'll make a mistake and need to cancel a job. This can be done with
the `scancel` command. Let's submit a job and then cancel it using
its job number (remember to change the walltime so that it runs long enough for
you to cancel it before it is killed!).

```
[yourUsername@login7a [cosma7] ~]$ sbatch example-job.sh
[yourUsername@login7a [cosma7] ~]$ squeue -u yourUsername
```
{: .language-bash}

```
Submitted batch job 5791551

  JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
5791551 cosma7-pa hello-wo yourUser PD       0:00      1 (Priority)
```
{: .output}

Now cancel the job with its job number (printed in your terminal). A clean
return of your command prompt indicates that the request to cancel the job was
successful.

```
[yourUsername@login7a [cosma7] ~]$ scancel 5791551
# It might take a minute for the job to disappear from the queue...
[yourUsername@login7a [cosma7] ~]$ squeue -u yourUsername
```
{: .language-bash}

```
...(no output when there are no jobs to display)...
```
{: .output}

> ## Cancelling multiple jobs
> 
> We can also cancel all of our jobs at once using the -u option. This will delete all jobs for a specific user (in this case, yourself). Note that you can only delete your own jobs.
> 
> Try submitting multiple jobs and then cancelling them all.
> 
> > ## Solution
> > 
> > First, submit a trio of jobs:
> >
> > ```
> > [yourUsername@login7a [cosma7] ~]$ sbatch example-job.sh
> > [yourUsername@login7a [cosma7] ~]$ sbatch example-job.sh
> > [yourUsername@login7a [cosma7] ~]$ sbatch example-job.sh
> > ```
> > {: .language-bash}
> > 
> > Then, cancel them all:
> >
> > ```
> > [yourUsername@login7a [cosma7] ~]$ scancel -u yourUsername
> > ```
> > {: .language-bash}
>{: .solution}
{: .challenge}

## Other Types of Jobs

Up to this point, we've focused on running jobs in batch mode.
Slurm also provides the ability to start an interactive session.

There are very frequently tasks that need to be done interactively. Creating an
entire job script might be overkill, but the amount of resources required is
too much for a login node to handle. A good example of this might be building a
genome index for alignment with a tool like [HISAT2][hisat]. Fortunately, we
can run these types of tasks as a one-off with `srun`.

`srun` runs a single command on the cluster and then exits. Let’s demonstrate this by running the `hostname` command with `srun`. (We can cancel an `srun` job with `Ctrl-C`.) Note that we still need to specify the account, partition, and expected runtime as we would with any job, but in the case of `srun`, we can only specify these on the command line:

```
[yourUsername@login7a [cosma7] ~]$ srun --account=yourAccount --partition=cosma7 --time=00:01:00 hostname
```
{: .language-bash}

```
somenode.cosma7.network
```
{: .output}

### Interactive jobs

Sometimes, you will need a lot of resource for interactive use. Perhaps it’s our first time running an analysis or we are attempting to debug something that went wrong with a previous job. Fortunately, Slurm makes it easy to start an interactive job with `srun`:

```
[yourUsername@login7a [cosma7] ~]$ srun --account=yourAccount --partition=cosma7 --time=00:01:00 --pty bash
```
{: .language-bash}

You should be presented with a bash prompt. Note that the prompt will likely change to reflect your new location, in this case the compute node we are logged on. You can also verify this with `hostname`.

When you are done with the interactive job, type `exit` to quit your session.

{% include links.md %}

[fshs]: https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard
[hisat]: https://ccb.jhu.edu/software/hisat2/index.shtml
