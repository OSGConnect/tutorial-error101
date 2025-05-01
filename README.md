---
ospool:
    path: htc_workloads/submitting_workloads/tutorial-error101/README.md
---

# Troubleshooting Job Errors

In this lesson, we'll learn how to troubleshoot jobs that never start or fail in unexpected ways. 

# Troubleshooting techniques
We will look into 4 criteria. The criteria are listed below

## Diagnostics with condor_q
 - [Job is held](#job-is-held)
 - [Job completed but was unsuccessful](#job-completed-but-was-unsuccessful)
 - [Job does not start](#job-does-not-start)
 - [Job is running longer than expected](#job-is-running-longer-than-expected)

## Job is held
HTCondor puts your jobs on hold when something goes wrong in the process of managing your jobs. HTCondor provides a ``Hold Reason`` that explains what went wrong. To see what went wrong you can use the command `condor_q -hold`. A typical hold message may look like the following

	$ condor_q -hold
	ap40.uw.osg-htc.org : <128.105.68.62:9618?... @ 08/01/24 15:01:50
 	ID         OWNER          HELD_SINCE  HOLD_REASON
	130.0       alice           8/1  14:56 Transfer output files failure at ⋯

In this particular case, the message indicates that HTCondor cannot transfer back output files, likely because they were never created. This hold condition can be triggered if a user has this option in their HTCondor submit file:

	transfer_output_files = outputfile

However, when the job executed, it went into the Held state. To learn more, `condor_q -hold JOB-ID` can be used. An example from this scenario is shown below:

	$ condor_q -hold JOB-ID

	Transfer output files failure at access point… while receiving files from the execution point. Details: Error from ….execute point … failed to send file(s) at apxx; failed to read from file /path: (errno 2) No such file or directory

The reason why the file transfer failed is because `outputfile` was never created on the worker node. Remember that at the beginning we said that the user specifically requested `transfer_outputfiles = outputfile`! HTCondor could not complete this request, and so the job went into the Held state instead of finishing normally.

It's quite possible that the error was simply transient, and if we retry, the job will succeed. We can re-queue a job that is in Held state by using `condor_release`: 

	condor_release JOB-ID 

### Common scenarios for job going on hold

Some of the common scenarios and example hold mesaages where the job goes on hold are described below
<ul>
<li> The executable is using more resources(memory) than requested in the job submit file. An example `hold` message for this scenario will look like:</li>
	
	Error from …: memory usage exceeded request_memory
	Job in status 2 put on hold by SYSTEM_PERIODIC_HOLD due to memory usage ...

<li> Typo(s) in the `transfer_input_files` path: If there are typos in the file path or the file does not exist in the provided path, then HTCondor will put the job on hold and will provide a hold message similar to</li>

	Transfer input files failure at access point apxx while sending files to the execution point. Details: reading from file /path: (errno 2) No such file or directory

<li> If a job ran more than the default allowed run time-HTCondor will put the job on hold providing a message like</li>

	The job exceeded allowed execute duration of 20:00:00

<li> You have reached your disk quota and HTCondor can not write the output file to your access point. This may happen while performing [checkpointing](https://portal.osg-htc.org/documentation/htc_workloads/submitting_workloads/checkpointing-on-OSPool/) jobs or regular jobs when your disk quota is reached. The hold message will look like:</li>

	Transfer output files failure at access point… while receiving files from the execution point. Details: Error from ….execute point … failed to send file(s) at apxx; failed to create directory /path Disk quota exceeded
</ul>
For these cases, sometimes it is better to just remove the jobs, fix the issues and then resubmit them. In some cases (e.g. excessive memory usage) using [condor_qedit](https://htcondor.readthedocs.io/en/latest/man-pages/condor_qedit.html) to fix the issue and releasing them might be convenient. Whenever in doubt please reach out to our [support](https://portal.osg-htc.org/documentation/support_and_training/support/getting-help-from-RCFs/) team.  

## Job completed but was unsuccessful
Under this case two things will be considered. First, your job has run but the code did not execute correctly or in the expected manner. Second, your job ran but it did not produce/transfer the desired output files.

For these scenarios looking at the `error` and `log` files will provide more information about your job. In addition, having various `ls` or `echo` statements that shows the contents of the working directory and the code's progression are helpful in diagnosing the actual issue. An example is shown here:

	echo "Here are the files at the start of the job:"
	ls -R
	echo "Executing main command..."
	my_command	#Part of the original code
	echo "... finished. Here are the files at the end of the job:"
	ls -R

While using the above commands, looking at the `output` file will provide the status of your code. In addition, for your commands if enabling debugging or verbose logging is an option then do those. Moreover, [condor_chirp](https://www.google.com/url?q=https://htcondor.readthedocs.io/en/latest/man-pages/condor_chirp.html&sa=D&source=editors&ust=1740162417565289&usg=AOvVaw1HJ8p454psoN6alPpujatn) can be a very useful tool in this regard as it sends information directly back to the Access Point. For example, the following will add a statement to your .log file:
	command1
	condor_chirp ulog "Finished executing command1"
	command2
## Job does not start

Matchmaking cycle (the process of finding an execution point to run the submitted jobs in queue) can take more than 5 minutes to complete and it can be longer if the server is busy. In addition, the more/more-specific resources are requested, typically the longer is the wait. For example if you are submitting a lot of GPU jobs, there may be fewer `GPUs` (than other resources) to run your jobs. This type of issue can be diagnosed using the `-better-analyze` flag 
with `condor_q` to see the detailed information about why a job isn't 
starting.                          

	$ condor_q -better-analyze JOB-ID 

Let's do an example. First we'll need to login as usual, and then load the tutorial *error101*.

	$ ssh username@apxx...
	
	$ git clone https://github.com/OSGConnect/tutorial-error101.git
	$ cd tutorial-error101
	$ condor_submit error101_job.submit 

We'll check the job status the normal way:

	condor_q username

For some reason, our job is still idle. Why? Try using `condor_q
-better-analyze` to find out. 

	$ condor_q -better-analyze JOB-ID 
	 
	# Produces a long ouput. 
	# The following lines are part of the output regarding the job requirements.  
	
	The Requirements expression for your job reduces to these conditions:
	
	         Slots
	Step    Matched  Condition
	-----  --------  ---------
	[0]       10674  TARGET.Arch == "X86_64"
	[1]       10674  TARGET.OpSys == "LINUX"
	[3]       10674  TARGET.Disk >= RequestDisk
	[5]           0  TARGET.Memory >= RequestMemory
	[8]       10674  TARGET.HasFileTransfer

By looking through the match conditions, we see that many nodes match our requests for the Linux operating system and the x86_64 architecture, but none of them match our requirement for memory. 

Let's look at our submit script and see if we can find the source of this error:

	$ cat error101_job.submit 
	Universe = vanilla
	
	Executable = error101.sh
	
	# to sleep an hour
	Arguments = 3600
	
	request_memory = 2 TB
	
	Error = job.err 
	Output = job.out 
	Log = job.log 
	Queue 1 

See the `request_memory` line? We are asking for 2 Terabytes of memory, when we meant to only 
ask for 2 Gigabytes of memory. Our job is not matching any available job slots because 
none of the slots offer 2 TB of memory. Let's fix that by changing that line to read `request_memory = 2 GB`.

	$ nano error101_job.submit

Let's cancel our idle job with the `condor_rm` command and then resubmit our edited job:

	$ condor_rm JOB-ID
	$ condor_submit error101_job.submit

Alternatively, you can edit the resource requirements of the idle job in queue:

	condor_qedit JOB_ID RequestMemory 2048

## Job is running longer than expected

To troubleshoot this issue we recommend checking your .log files and see if 
<ul>
<li>The job has been continuously running on the same slot</li>
<ul>
Debugging tools: 
<li>[condor_tail](https://htcondor.readthedocs.io/en/latest/man-pages/condor_tail.html)</li>- It returns the last X bytes of the job standard output or error files
<li>[condor_ssh_to_job](https://htcondor.readthedocs.io/en/latest/man-pages/condor_ssh_to_job.html). It allows to log in to the execution point. Point to be noted-It **does not work on all nodes**. 
</ul>
<li>The job has been interrupted and restarted on another slot</li>
<ul>
Debugging tips:
<li>If it happens once or twice, adjust your expectation of the runtime.</li>
<li>If it happens many times, your job runtime may be too long for the system or [contact the support team](maito:support@osg-htc.org) expectation of the runtime.</li>
</ul> 
<li>The job is stuck on the file transfer step</li>
<ul>
Debugging tips:
<li>Check the `.log` file for meaningful errors</li>
<li> Can happen if you or someone else is transferring a lot of data (large size or many files) and the `Access Point` is overwhelmed</li>
</ul>
</ul>
