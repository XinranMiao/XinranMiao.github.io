---
title: CHTC Examples
author: Xinran Miao
layout: post
active: blog
permalink: /blog/20230313chtc/
---

This post prepares a few examples on running jobs on the [Center for High Throughout Computing (CHTC)](https://chtc.cs.wisc.edu) at UW-Madison.


## Getting Started
<ol start="1">
  <li>  Requesting an account.</li>
  <li> Logging in with your username.</li>
</ol>

{% highlight bash%}
     ssh username@submit.chtc.wisc.edu # some may need to log in to submit2.chtc.wisc.edu
{% endhighlight %}
 
 <ol start="3">
  <li>  Write two files: a submit file including submit information and a `.sh` to execuate your job.</li>
</ol>
{% highlight bash%}
# r_example.submit
universe = docker
docker_image = xinranmiao/r_example:20221006
log = r_example.log
error = r_example.err
output = r_example.out
executable = r_example.sh
request_disk = 1GB
queue
{% endhighlight %}

{% highlight bash%}
# r_example.sh
#!/bin/bash
R -e "installed.packages()"
{% endhighlight %}

 <ol start="4">
  <li>  Submit your job.</li>
</ol>
{% highlight bash%}
condor_submit r_example.submit
{% endhighlight %}

 <ol start="5">
  <li>  (Optional) Check your job and/or edit.</li>
</ol>
{% highlight bash%}
# query your jobs
condor_q
# you'll find your job_id, say it's 9644580 

# query jobs on hold
condor_q -hold

# analyze a job (e.g., why a job keeps idle)
condor_q -better-analyze 9644580 

# log into a job
condor_ssh_to_job auto-retry 9644580 

# change requested resources
# use it when a job is held due to insufficient resources
condor_qedit 9644580 RequestMemory 2200

# remove a job
condor_rm 9644580 
{% endhighlight %}


## Parallel Jobs
### Example 1: Run Multiple Jobs
Suppose you wish to run 20 jobs, you may change the `queue` in `r_example.submit` to be `queue 20`.
{% highlight bash%}
#r_example.submit
universe = docker
docker_image = xinranmiao/r_example:20221006
log = r_example$(Process).log
error = r_example$(Process).err
output = r_example$(Process).out
executable = r_example.sh
request_disk = 1GB
queue 20
{% endhighlight %}

### Example 2: Read Arguments from File
Suppose one wishes to read arguments from `args.txt` in order to run an R file `run.R` taking several input arguments.
{% highlight r%}
# run.R
opts <- commandArgs(trailingOnly = TRUE)
print(opts)
{% endhighlight %}

{% highlight bash%}
# args.txt
1,3,100
2,3,200
{% endhighlight %}

Then we can modify the last line of the submit file so that it reads from the `args.txt`. Remember to transfer both `args.txt` and `run.R` by setting `transfer_input_files`.

{% highlight bash%}
# parallel_example.submit
universe = docker
docker_image = xinranmiao/r_example:20221006
log = parallel_example$(job_id).log
error = parallel_example$(job_id).err
output = parallel_example$(job_id).out

executable = parallel_example.sh
arguments = $(job_id) $(n) $(d)

transfer_input_files = args.txt, run.R

request_cpus = 1
request_memory = 1GB
request_disk = 2GB
requirements = (has_avx == true)

queue job_id, d, n from args.txt
{% endhighlight %}

We may also update the bash script so that it runs the `run.R` file.

{% highlight bash%}
# parallel_example.sh
#!/bin/bash
Rscript run.R $2 $3
{% endhighlight %}


## Environment.
### Docker Images
The CHTC supports using pulling docker images; see their [documentation](https://chtc.cs.wisc.edu/uw-research-computing/docker-jobs.html). In fact, all the examples above used a toy docker image I made from [this dockerfile](https://github.com/XinranMiao/docker_chtc_example/blob/main/R_example/Dockerfile). You may find some toy examples for R and python from [my earlier repo](https://github.com/XinranMiao/docker_chtc_example).

### Instaliing Packages
Their are official tutorials on running [R](https://chtc.cs.wisc.edu/uw-research-computing/r-jobs.html), [Julia](https://chtc.cs.wisc.edu/uw-research-computing/julia-jobs.html), [Python](https://chtc.cs.wisc.edu/uw-research-computing/python-jobs.html), and others on CHTC.

## Resources:

[CHTC User Guide](https://chtc.cs.wisc.edu/uw-research-computing/guides.html)

