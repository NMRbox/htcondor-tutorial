
# HTCondor Tutorial 

Presented by: 
Jon Wedell 
Lead Software Developer, BioMagResBank 
wedell@uchc.edu 

## Background 

During this tutorial we will walk through submitting jobs to the HTCondor 
workload management system on NMRbox. We will then investigate how the jobs run in the HTCondor 
environment and explore how to manage submitted jobs. 

### The submit file 

The submit file is the least complex way to submit one or more jobs to HTCondor. It consists of 
the necessary information to specify what work to perform (what executable to call, which arguments 
to provide, etc.) and any requirements for your job to complete successfully (a GPU, a certain release of 
NMRbox, etc.) 

#### The basics 

Let's look at a simple submit file in detail to see what arguments are necessary for a submission. 


``` 
executable = /bin/ls 
arguments = / 

log = logs/basic.log 
output = logs/basic.out 
error = logs/basic.err 
getenv=True 
queue 
``` 

Going through the lines in order: 

* The executable specifies which command should actually be ran when the job runs. You 
must use an absolute path. 
* The arguments line, while technically optional, will almost always be used. It allows you 
to specify which command line arguments to provide to the executable. 
* The log, output, and error commands specify where HTCondor will log. The output and 
error will be populated with whatever the job writes to STDERR and STDOUT while the 
log will contain the HTCondor logs related to the scheduling and running of the job. 
* This line ensures that the HTCondor job runs with the same shell environment as existed in the shell 
where you submitted the job. Without specifying this, your job may fail due to not being able to locate 
a called executable in the path, or due to other issues. This is not a very portable option when submitting 
to a heterogeneous computing environment, but it works well on production NMRbox machines, and simplifies 
your life when developing your first submit files. 
* The queue parameter is special. Each time this argument is encountered in the file 
HTCondor will submit a job to the queue with whatever arguments were specified prior 
in the file. You can specify queue multiple times on different lines, and optionally 
change any of the parameters between the queue lines to customize the various runs 
of the job. You can also provide a number after `queue` to submit multiple jobs 
with the same parameters. (e.g. if doing a Monte Carlo simulation). You can see an example 
of this behavior used in practice in the "cluster.sub" file. 

#### Additional submit file options 

Now let's look at additional parameters which can be specified in the submit file. 

``` 
request_cpus = 1 
request_disk = 1MB 
request_memory = 4GB 
``` 

These arguments speak for themselves. They are telling the HTCondor matchmaker 
that in order to run your job the machine must be able to provide the specified 
resources.

*Note* - HTCondor will terminate your job (in the job log file it will report `Abnormal termination (signal 9)`)
if you use more than 2GB of memory when you omit `request_memory`. Therefore, if your jobs require more than 2GB of memory,
please ensure you request enough memory with the `request_memory` parameter.

``` 
requirements = NMRPIPE == "11.5 rev 2023.105.21.31"
+Production = True 
``` 

These arguments allow us to further refine which machines our jobs will execute on. 
* The `requirements` line is specifying that we only want to run our job on two specific 
releases of NMRbox. This is useful if you require a specific version of an installed software 
package which is not available on all releases of NMRbox. Remember - you can check which releases 
of NMRbox have a given version of a software package in them using the NMRbox software registry on 
the NMRbox web site. Here is an example [for NMRpipe](https://nmrbox.org/software/nmrpipe). 
* The `+Production=True` job ensures that our job only runs on [production NMRbox machines](https://nmrbox.org/hardware). (Normal 
NMRbox computational machines). This line is automatically added if you submit your job from a production 
NMRbox machine, but it's good to have it explicitly present. We have additional compute nodes in our pool 
which you can access, but they don't have the full set of NMR software packages installed, so using them 
requires extra work when setting up your submission. If you don't require any specific software to be installed 
on the worker node (for example, if you are running a staticly compiled binary executable or a pure-python 
program) you can specify `+Production=False` to get access to these additional computational nodes. 

``` 
transfer_executable = FALSE 
should_transfer_files = NO 
``` 

In most circumstances, HTCondor expects you to specify exactly which input files your job needs and then 
it will manage transferring those files to the worker node for you, and transferring any created output files back. 
While this is maximally portable, it has increased overhead and requires more work by the job submitter. On the 
NMRbox computational pool, all the nodes share a filesystem, so it's possible to bypass this transfer mechanism 
and rely on the shared file system, which avoids you needing to specify which input files the job needs. 

The only thing to be careful of is that you have your job located in your home directory (or a subdirectory) 
and not in your scratch directory or other machine-specific temporary directory. 

#### Machine attributes

The following attributes are defined on NMRbox:
- **Production** is a machine with [NMRbox software](https://nmrbox.nmrhub.org/software) installed. By default, jobs are 
   limited to production machines.
- **Release** is a machine version as listed on the [hardware](https://nmrbox.nmrhub.org/hardware) page.
- Slightly modified software versions are available as attributes. Underscores and dashes in software names
  are removed.
    - *NMRpipe* versions will be implemented as **NMRPIPE**.
    - *CcpNmr Analysis Assign* is implemented as **CCPNMRANALYSISASSIGN**. 

#### Variables! 

It is possible to specify custom variables in your submit file, and have them interpreted in the appropriate 
places. Here is an example of the concept: 

``` 
arguments = example $(JobName).pdb 
output = logs/$(JobName).stdout 
error = logs/$(JobName).stderr 

JobName = 2dog 
queue 
JobName = 2cow 
queue 
JobName = 1rcf 
queue 
``` 

You should be familiar with the `arguments`, `output`, and `error` line from earlier in the tutorial, but now 
you can see that they are using the value of a variable called `JobName` to determine their value. Where is 
this variable coming from? From us, later in the submit file! You can define any variables you want, other 
than ones that would conflict with built-in HTCondor parameter names. In this example, we specify three 
different JobNames, and queue up three different jobs. Each of those three jobs will write to a different 
output and error file, and will use a different file for their input. 

#### Variables, continued 

In addition to defining your own variables, some are automatically available. In the 
code above, the example shows running a given executable against three different input files, 
and performing some action. But what if you have only a single input file, and you want to run a given 
computation 1000 times? For example, imagine you are using Monte Carlo methods. 

Fortunately it is easy to achieve this as well. Look at this example `.sub` file stub: 

``` 
executable = monte_carlo_method.py 
arguments = -seed $(PROCID) 

log = logs/mc.log 
output = logs/mc_$(PROCID).out 
error = logs/mc_$(PROCID).err 
queue 1000 
``` 

In this case we've submitted 1000 processes with this single submit file, all of which belong to a single 
cluster. `$(PROCID)` is automatically interpreted for each individual process, triggering all 1000 
processes to run with a different seed. 

When you submit such a job, Condor will print out the job ID. Furthermore, while 
each time queue is specified in the file creates an additional process, they share they same 
job ID, it is the process ID that is incremented. When looking at the queue, you 
can see these numbers in the form "jobID.procID". Running `condor_rm` with a job ID 
will remove all of the jobs with that ID, while running `condor_rm` with a jobID.procID 
will only remove the specific instance of the job with the given procID. 

### The queue 

Condor manages a queue of submitted jobs which are waiting to run. By default, the jobs run in 
roughly the order they were submitted. To view your job queue, you can run the `condor_q` command. 

You will get a result similar to this one: 

``` 
-- Schedd: wiscdaily.nmrbox.org : <127.0.0.1:51028?... 
 ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD 1.0   jwedell         7/10 14:59   0+00:00:00 I  0   0.1  zenity --info --te 
1 jobs; 0 completed, 0 removed, 1 idle, 0 running, 0 held, 0 suspended 
``` 

This example shows a single job (with ID 1) is idle. If you have a submit file with multiple "queue" 
statements but only see one job, you can use `condor_q -nobatch` to see each individual job separately. 

Here are some common states of jobs that you should be familiar 
with: 

* `I/Idle` - This means the job is idle. All jobs will be in this state for at least a brief period before 
they begin running. If they remain in this state for an extended period of time, it may be the case 
that no machines match your job requirements. To check why a job hasn't yet run, you can use the 
`condor_q` command to investigate the status of the job. Adding `-better` will perform a "better analysis" 
of the job. So for the example queue above, `condor_q -better 1` would explain why the job hasn't yet 
started running. 
* `R/Running` - This means the job is currently executing on a machine in the pool. Great! 
* `H/Held` - This means that something went wrong when attempting to run the job. There are many reasons 
this could happen - to check the error message, you can again use `condor_q -better` with the job ID, and 
you can also look in the file you defined as the "log" for the job. 

If you realize there is a problem with your requirements or submit file, you can remove the job with the 
`condor_rm` command followed by the job ID. If you have a job with the queue statement used more than once, 
you will remove all of them if you use the integer ID for the submission. You can then make any necessary 
changes and resubmit the job. 

### The pool 

You can also investigate the Condor pool to see what resources are available. To see the pool, 
use the `condor_status` command. Note that our pool uses something called "dynamic partitioning". What 
this means is that a machine with 128 cores and 500GB of RAM will only appear as one resource when you 
run this command. But that doesn't mean it can only run one job. Instead, what happens is that HTCondor automatically 
divides up the resources on this machine according to what was requested in the submit file. 

Therefore, such a machine may wind up running 100 jobs which only require 1 GB of RAM and a single core, 
1 job which requires 20 cores and 10 GB of RAM, and 1 job which requires 1 CPU and 1 GPU. This ensures 
that our resources can be used most effectively, and it's why it is important that you enter realistic numbers 
for `request_memory` so that you don't ask for more memory than you'll need. 

Here is an example of what you may see when you run the `condor_status` command: 

``` 
Name                        OpSys      Arch   State     Activity LoadAv Mem      ActvtyTime 

slot1@babbage.nmrbox.org    LINUX      X86_64 Unclaimed Idle      0.000 2063937  5+13:10:01 
slot1@barium.nmrbox.org     LINUX      X86_64 Unclaimed Idle      0.000  193335 11+22:11:08 
slot1@bismuth.nmrbox.org    LINUX      X86_64 Unclaimed Idle      0.000  364790  0+16:49:41 
slot1@bromine.nmrbox.org    LINUX      X86_64 Unclaimed Idle      0.000  385574 11+22:24:05 
slot1@calcium.nmrbox.org    LINUX      X86_64 Unclaimed Idle      0.000  191880 23+21:50:56 
slot1@chlorine.nmrbox.org   LINUX      X86_64 Unclaimed Idle      0.000  191880  6+20:35:04 
slot1@chromium.nmrbox.org   LINUX      X86_64 Unclaimed Idle      0.000  191880 45+12:21:35 
slot1@cobalt.nmrbox.org     LINUX      X86_64 Unclaimed Idle      0.000  176292 24+17:32:17 
slot1@compute-1.bmrb.io     LINUX      X86_64 Unclaimed Idle      0.000  515522  4+05:21:42 
... 
slot1@zinc.nmrbox.org       LINUX      X86_64 Unclaimed Idle      0.000  385576  1+00:39:18 

 Total Owner Claimed Unclaimed Matched Preempting Backfill  Drain 
 X86_64/LINUX    50     0       9        41       0          0        0      0 
 Total    50     0       9        41       0          0        0      0 
 ``` 

This shows the current slots in the pool and their status. Remember that a machine can spawn multiple 
slots depending on what resources it has available and are requested. 

If you want to check a given requirement against the pool to see which machines would be eligible 
to run your job, you can do that using the `-const` argument to `condor_status`. Here are a few examples: 


* Check which machines have NMRPIPE version 11.5 rev 2023.105.21.31:

  * `condor_status -const '(NMRPIPE == "11.5 rev 2023.105.21.31")'`
* Check which machines have at least 100 CPUs 
	* `condor_status -const '(cpus > 100)'` 


### Submitting a job 

With an understanding of the essential parts of a Condor submission file, let's go ahead and 
actually submit a job to Condor. 

'pdb.sub' in this github repository uses many of the options described above. The executable simply 
echoes some text to STDOUT, but it illustrates how you could run a given computation 
against a set of PDB structures. 

To submit a `.sub` file to Condor, use the `condor_submit` command as such: 

``` 
condor_submit pdb.sub 
``` 

The moment you enter this command your job will be in the condor job queue. (If you have 
multiple "queue" statements in your job, those are still considered one clustered job with 
multiple processes). It will print off the job ID: 

``` 
Submitting job(s)... 
3 job(s) submitted to cluster 1. 
``` 

### Check the results 

If you're able to run the `condor_q` command quickly enough before 
your job completes running, you will see the job you just submitted in the queue. 

``` 
-- Schedd: strontium.nmrbox.org : <155.37.253.57:9618?... @ 06/07/22 06:47:35 
OWNER   BATCH_NAME    SUBMITTED   DONE   RUN    IDLE  TOTAL JOB_IDS 
jwedell ID: 1        6/7  06:47      _      _      3      3 1.0-2 

Total for query: 3 jobs; 0 completed, 0 removed, 3 idle, 0 running, 0 held, 0 suspended Total for jwedell: 3 jobs; 0 completed, 0 removed, 3 idle, 0 running, 0 held, 0 suspended Total for all users: 3 jobs; 0 completed, 0 removed, 3 idle, 0 running, 0 held, 0 suspended 
``` 

With `-nobatch`: 

``` 
-- Schedd: strontium.nmrbox.org : <155.37.253.57:9618?... @ 06/07/22 06:48:54 
 ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD 
 1.0   jwedell         6/7  06:47   0+00:00:00 I  0    0.0 echo example 2dog.pdb 
 1.1   jwedell         6/7  06:47   0+00:00:00 I  0    0.0 echo example 2cow.pdb 
 1.2   jwedell         6/7  06:47   0+00:00:00 I  0    0.0 echo example 1rcf.pdb 
Total for query: 3 jobs; 0 completed, 0 removed, 3 idle, 0 running, 0 held, 0 suspended 
Total for jwedell: 3 jobs; 0 completed, 0 removed, 3 idle, 0 running, 0 held, 0 suspended Total for all users: 3 jobs; 0 completed, 0 removed, 3 idle, 0 running, 0 held, 0 suspended 
``` 

Continue to use `condor_q` to watch and see when your job has completed. When it completes, 
take a look at the output files in the `output` folder and the `log` folder. 

### Interactive debugging 

Let's explore a very useful technique for examining why a job might not be running the way 
that you expect. Update the submit file to replace the executable with `/bin/sleep` and update 
the `arguments` to be `1000`. This will just run the linux sleep command for 1000 seconds, giving you 
time to see your job running in the queue. Submit it again using `condor_submit` as before. 

After a few seconds, you should see that one or more of the processes in your submission have entered 
the running job state. (Remember, you can use `condor_q` to check.) Determine the job and process ID of one 
you would like to explore futher, and run 

`condor_ssh_to_job clusterID.procID` where clusterID and procID are replaced with the value for your job, 
which you can get from `condor_q` or `condor_q nobatch`. 

This will open up an interactive SSH session to the exact machine and location your job is running. You 
can use this to manually step through the actions your job would take and explore and unexpected behavior. 

### GPU usage
To request a GPU and a certain amount of GPU memory
```
request_gpus = 1
# Replace 1 in the line below with the amount of GPU memory required in megabytes
require_gpus = (GlobalMemoryMb >= 1)
# When we update our condor pool version, you can replace the `require_gpus` argument with this simplified form. (This page will be edited.)
# gpus_minimum_memory = 1MB
```

It is best not to request excessive resources, as this with lower your relative user priority compared
to other htcondor users.

#### File transfer 

In the vanilla universe, HTCondor will automatically transfer back any files created 
during the execution of a job to your local machine. (Though it will not transfer 
back folders created or files within them automatically - you must manually specify 
those using the `transfer_output_files` argument if you want them to be preserved.) 

As mentioned before, you can avoid this entirely using the `should_transfer_files` and `transfer_executables` 
options, and relying on the shared filesystem. 


## Version
NMRbox is currently running the HTCondor version 23 [feature channel](https://htcondor.org/htcondor/release-highlights/).

### Helpful hints 

Here are some things to keep in mind while working to use Condor to take advantage of distributed computing: 

1) Try to keep the runtime of an individual job process to less than 8 hours. 
2) You'll generally make better use of the resources if you can break your work up into compute units 
of a single core. This is because it is much easier to find a single available core in the pool to start 
running your job than to find, for example, a machine with 48 simultaneously free cores. So rather than 
run a multiprocess workflow with `request_cpus=48` see if you can instead run 48 condor jobs/processes 
which each require just one CPU. Note that if the pool has a queue of single core jobs, your multicore jobs will
not run at all until the queue of single core jobs is exhausted. As such, multicore jobs are strongly discouraged.
3) To take advantage of our additional "compute-only" nodes, you can't count on NMR software being installed 
and will need to ensure your executable can stand alone. A python virtual environment would allow this, as would 
a staticly compiled binary. 
4) Don't be afraid to [reach out](mailto:support@nmrbox.org)! We're happy to help you take maximum utility from our compute resources to 
advance your research. 


## Summary 

This has been a very brief demonstration of the basic features of HTCondor job 
submission and management. For more details on the submit file format, please 
see the [HTCondor documentation](https://htcondor.readthedocs.io/en/latest/users-manual/quick-start-guide.html).# HTCondor Tutorial 

Presented by: 
Jon Wedell 
Lead Software Developer, BioMagResBank 
wedell@uchc.edu 
