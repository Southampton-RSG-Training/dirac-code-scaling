---
title: "Scalability Profiling"
slug: dirac-code-scaling-scalability-profiling
teaching: 0
exercises: 0
questions:
- "How scalable is a particular piece of code?"
- "How can I generate evidence for a code's scalability?"
- "What does good and bad scalability look like?"
objectives:
- "Explain how Amdahl's Law can help us understand the scalability of code."
- "Use Amdahl's Law to predict the theoretical maximum speedup on some example code when using multiple processors."
- "Understand strong and weak scaling graphs for some example code."
- "Describe the graphing characteristics of good and bad scalability."
keypoints:
- "We can use Amdahl's Law to understand the expected speedup of a parallelised program against multiple cores."
- "It's often difficult to estimate the proportion of serial code in our programs, but a reformulation of Amdahl's Law can give us this based on multiple runs against a different number of cores."
- "Run timings for serial code can vary due to a number of factors such as overall system load and accessing shared resources such as bulk storage."
- "The Message Passing Interface (MPI) standard is a common way to parallelise code and is available on many platforms and HPC systems, including DiRAC."
- "When calculating a *strong scaling* profile, the additional benefit of adding cores decreases as the number of cores increases."
- "The limitation of strong scaling is the fixed problem size, and we can increase the problem size with the core count to obtain a *weak scaling* profile."
---

Let's look at how we can determine the scalability characteristics for some example code.

## Calculating π as an Example

In the following example we have a simple piece of code that calculates π. When run on our local system we get the following results:

| Cores (n) | Run Time (s) | Result      | Error         | Speedup |
|-----------|--------------|-------------|---------------|---------|
| 1         | 3.99667      | 3.141592655 | 0.00000003182 | -       |
| 2         | 2.064242     | 3.141592655 | 0.00000003182 | 1.94    |
| 4         | 1.075068     | 3.141592655 | 0.00000003182 | 3.72    |
| 8         | 0.687097     | 3.141592655 | 0.00000003182 | 5.82    |
| 16        | 0.349366     | 3.141592655 | 0.00000003182 | 11.44   |

> ## How is our program parallelised?
>
> Our π program uses the popular [Message Passing Interface (MPI)](https://en.wikipedia.org/wiki/Message_Passing_Interface) standard to enable communication between each of the parallel portions of code, running on separate CPU cores. It's been around since the mid-1990's and is widely available on many operating systems. MPI also is designed to make use of multiple cores on a variety of platforms, from a multi-core laptop to large-scale HPC resources such as those available on DiRAC. There are many available [tutorials](https://hpc-tutorials.llnl.gov/mpi/) on MPI.
{: .callout}

By using MPI, we can reduce the run time of our code by using more cores, without affecting the results. The speedup shown in the table above was calculated using,

> *Speedup = T<sub>1</sub> / T<sub>n</sub>*

 The speedup efficiency, which measures how efficiently the additional resources are being used, is,

> *Efficiency<sub>n</sub> = Speedup<sub>n</sub> / n*,

which could be as high as 1, but probably will never be in practice.

> ## What Type of Scaling?
>
> Looking at the table above, is this an example of determining strong or weak scaling?
>
> > ## Solution
> >
> > This is an example of strong scaling, as we are keeping the data sample the same but increasing the number of cores used.
>{: .solution}
>
{: .challenge}

When we plot the run time against the number of cores, we see the following graph:

![Time vs Cores for an implementation of Pi]({{ site.url }}{{ site.baseurl }}/fig/scalability-pi-time-vs-cores.png){: width="650px"}

So we can see that as the number of cores increases, the run time of our program decreases. This makes sense, since we are splitting the calculation into smaller pieces which are executed at the same time.

## Amdahl's Law

If we use *n* processors, we should expect *n* times speedup. But this is rarely, if ever, the case! In a program, there is always some portion of it which *must* be executed in serial (such as initialisation routines, I/O operations and inter-communication) which cannot be parallelised. This limits how much a program can be speeded up, as the program will always take *at least* the length of the serial portion. This is actually known as *Amdahl's Law*, which states that a program's serial parts limit the potential speedup from parallelising the code.

We can think of a program as being operations which *can* and *can't* be parallelised, i.e. the part of the code we can and can't be speeded up. The time taken for a program to finish executing is the sum of the fractions of time spent in the serial and parallel portion of the code,

> *Time to Complete (T) = Fraction of time taken in Serial Portion (F<sub>S</sub>) + Fraction of time taken in Parallel Portion (F<sub>P</sub>)*
>
> *T = F<sub>S</sub> + F<sub>P</sub>*

When a program executes in parallel, the parallel portion of the code is split between the available cores. But since the serial portion is not split in this way, the time to complete is therefore,

> *T<sub>n</sub> = F<sub>S</sub> + F<sub>P</sub> / n*

We can see that as the number of cores in uses increases, then the time to complete decreases until it approaches that of the serial portion. The speedup from using more cores is,

> *Speedup = T<sub>1</sub> / T<sub>n</sub> = ( F<sub>S</sub> + F<sub>P</sub> ) / ( F<sub>S</sub> + F<sub>P</sub> / n )*

To simplify the above, we will define the single core execution time as a single unit of time, such that *F<sub>S</sub> + F<sub>P</sub> = 1*.

> *Speedup = 1 / ( F<sub>S</sub> + F<sub>P</sub> / n )*

Again this shows us that as the number of cores increases, the serial portion of the code will dominate the run time as when *n = ∞*,

> *Max speedup = 1 / F<sub>S</sub>*

## What's the Maximum Speedup?

From the previous section, we know the the maximum speedup achievable is limited to how long a program takes to execute in serial. If we know the portion of time spent in the serial and parallel code, we will theoretically know by how much we can accelerate our program. However, it's not always simple to know the exact value of these fractions. But from Amdahl's law, if we can measure the speedup as a function of number of cores, we can estimate that maximum speed up.

We can rearrange Amdahl's law to estimate the parallel portion *F<sub>P</sub>*,

> *F<sub>P</sub> = n / ( n - 1 ) ( ( T<sub>1</sub> - T<sub>n</sub> ) / T<sub>1</sub> )*

Using the above formula on our example code we get the following results:

| Cores (n) | T<sub>n</sub> | F<sub>p</sub> | F<sub>s</sub> = 1 - F<sub>p</sub> |
|-----------|---------------|---------------|-----------------------------------|
| 1         | 3.99667       | -             | -                                 |
| 2         | 2.064242      | 0.967019      | 0.0329809                         |
| 4         | 1.075068      | 0.974678      | 0.0253212                         |
| 8         | 0.687097      | 0.946380      | 0.0536198                         |
| 16        | 0.349366      | 0.973424      | 0.0265752                         |
|-----------|---------------|---------------|-----------------------------------|
|           | **Average**   | 0.965375      | 0.0346242                         |
|-----------|---------------|---------------|-----------------------------------|

We now have an estimated percentage for our serial and parallel portion of our code. As you can see, as the number of cores we use increases, the time spent in the serial portion of the code increases.

> ## Differences in Serial Timings
>
> You may be wondering why the serial run time seems to vary depending on the run. There are several factors that are impacting our code. Firstly these were run on a working system with other users, so runtime will be affected depending on the load of the system. Throughout DiRAC, it is normal when you run your code to have exclusive access, so this will be less of an issue. But if, for example, your code accesses bulk storage then there may be an impact since these are shared resources. As we are using the MPI library in our code, it would be expected that the serial portion will actually increase slightly with the number of cores due to additional MPI overheads. This will have a noticeable impact if you try scaling your code into the thousands of cores.
{: .callout}

If we have several values, we can take the average to estimate an upper bound on how much benefit we will get from adding more processors. In our case then, the maximum speedup we can expect is,

> *Max speedup = 1 / F<sub>S</sub> = 1 / ( 1 - F<sub>P</sub> ) = 1 / ( 1 - 0.965375 ) = 29*

Below is a table of the expected maximum speedup for a given F<sub>P</sub>.

| F<sub>P</sub> | Max Speedup |
|---------------|-------------|
| 0.0           | 1.00        |
| 0.1           | 1.11        |
| 0.2           | 1.25        |
| 0.3           | 1.43        |
| 0.4           | 1.67        |
| 0.5           | 2.00        |
| 0.6           | 2.50        |
| 0.7           | 3.33        |
| 0.8           | 5.00        |
| 0.9           | 10.00       |
| 0.95          | 20.00       |
| 0.99          | 100.00      |
|---------------|-------------|

> ## Number of Cores vs Expected Speedup
>
> Using Amdahl's Law and the percentages of serial and parallel proportions of our example code. Fill In or create a table estimating the expected total speedup and change in speedup when doubling the number of cores, in a table like the following (with the number of cores doubling each time until a total of 4096):
>
> | Cores (n) | T<sub>n</sub> | Speedup | Change in Speedup |
> |-----------|---------------|---------|-------------------|
> | 1         | 3.99667       |         |                   |
> | 2         |               |         |                   |
> | 4         |               |         |                   |
> | ...       |               |         |                   |
> | 4096      |               |         |                   |
>
> When does the change in speedup drop below 1%?
>
> > ## Solution
> > Hopefully from your results you will find that we can get close to the maximum speedup calculated earlier, but it requires ever more resources. From our own trial runs, we expect the speedup to drop below 1% at 4096 cores, but it is expected that we would never run this code at these core counts as it would be a waste of resources.
>{: .solution}
> 
{: .challenge}


> ## How Many Cores Should we Use?
>
> From the data you have just calculated, what do you think the maximum number of cores we should use with our code to balance reduced execution time versus efficient usage of compute resources.?
>
> > ## Solution
> >
> > Within DiRAC we do not impose such a limit, this is a decision made by you. Every project has an allocation and it is up to you to decide what is efficient use of your allocation. In this case I personally would not waste my allocation on any runs over 128 cores.
>{: .solution}
> 
{: .challenge}


## Calculating a Weak Scaling Profile

Not all codes are suited to strong scaling. As seen in the previous example, even codes with as much as 96% parallelizable code will hit limits. Can we do something to enable moderately parallelizable codes to access the power of HPC systems? The answer is yes, and is demonstrated through *weak scaling*.

The problem with strong scaling is as we increase the number of cores, then the relative size of the parallel portion of our task reduces until it is negligible, and then we can not go any further. The solution is to increase the problem size as you increase the core count -- this is [Gustafson's law](https://en.wikipedia.org/wiki/Gustafson%27s_law). This method tries to keep the proportion of serial time and parallel time the same. We will not get the benefit of reduced time for our calculation, but we will have the benefit of processing more data. Below is a re-run of our π code. But this time, as we increase the cores we also increase the samples used to calculate π.

| Cores (n) | Run Time | Result       | Error         | % Improved Error |
|-----------|----------|--------------|---------------|------------------|
| 1         | 4.149807 | 3.141592655  | 0.00000003182 |                  |
| 2         | 4.362416 | 3.141592654  | 0.00000001591 | 50.00%           |
| 4         | 5.205988 | 3.141592654  | 0.00000000795 | 75.02%           |
| 8         | 4.356564 | 3.141592654  | 0.00000000397 | 87.52%           |
| 16        | 4.643724 | 3.141592654  | 0.00000000198 | 93.78%           |

![Improved Error vs Cores]({{ site.url }}{{ site.baseurl }}/fig/scalability-improved-error-vs-cores.png){: width="650px"}

As you can see, the run times are similar. Just slightly increasing. However, the accuracy of the calculated value of π has increased. In fact our percentage improvement is nearly in step with the number of cores.

When presenting your weak scaling it is common to show how well it scales, this is shown below:

![Weak Scaling - Cores vs Time]({{ site.url }}{{ site.baseurl }}/fig/scalability-weak-scaling-time.png){: width="650px"}

We can also plot the scaling factor. This is the percentage increase in run time compared to base run time for a normal run. In this case we are just using *T<sub>1</sub>*:

![Weak Scaling - Cores vs Scaling Factor]({{ site.url }}{{ site.baseurl }}/fig/scalability-weak-scaling-factor.png){: width="650px"}

The above plot shows that the code is highly scalable. We do have an anomaly with our 4 core run, however. It would be good to rerun this to get a more representative sample, but this result is a common occurrence when using shared systems. In this example we only did a single run for each core count. When compiling your data for presentation or submitting applications, it would be better to do many runs and exclude outlying data samples or provide an uncertainty estimate.

> ## Maximum Cores to Use?
>
> It would be hard to estimate the max cores we could use from this plot. Can you suggest an approach to get a clearer picture of this code's weak scaling profile?
>
> > ## Solution
> >
> > The obvious answer is to do more runs with higher core counts, and also try to resolve the *n = 4* sample. This should give you a clearer picture of the weak scaling profile.
>{: .solution}
> 
{: .challenge}

> ## Obtaining Resources to Profile your Code
>
> It may be difficult to profile your code if you do not have the resources at hand, but DiRAC can help. If you are in a position of wanting to use DiRACs facilities but do not have the resources to profile your code, then you can apply for a [Seedcorn project](https://dirac.ac.uk/seedcorn/). This is a short project with up to 100,000 core hours for you to profile and possibly improve your code before applying for a large allocation of time.
{: .callout}

> ## Profiling Pi Yourself
>
> If you would like to reproduce these sample runs on your system, the code is available on the [DiRAC GitHub page](https://github.com/DiRAC-HPC/MPI_PI).
{: .callout}


{% include links.md %}
