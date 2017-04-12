# Swarm

Swarm is a script designed to simplify submitting a group of commands to the Biowulf cluster. 

## How swarm works

The swarm script accepts a number of input parameters along with a file containing a list of commands that otherwise would be run on the command line.  swarm parses the commands from this file and writes them to a series of command scripts.  Then a single batch script is written to execute these command scripts in a slurm job array, using the user's inputs to guide how slurm allocates resources.

### Bundling versus folding

Bundling a swarm means running two or more commands per subjob serially, uniformly.  For example, if there are 1000 commands in a swarm, and the bundle factor is 5, then each subjob will run 5 commands serially, resulting in 200 subjobs in the job array.

Folding a swarm means running commands serially in a subjob only if the number of subjobs exceed the maximum array size (maxarraysize = 1000).  This is a new concept.  Previously, if a swarm exceeded the maxarraysize, then either the swarm would fail, or the swarm would be autobundled until it fit within the maxarraysize number of subjobs.  Thus, if a swarm had 1001 commands, it would be autobundled with a bundle factor of 2 into 500 subjobs, with 2 commands serially each.  With folding, this would still result in 1000 subjobs, but one subjob would have 2 serial commands, while the rest have 1.

Folding was implemented June 21, 2016.

## Behind the scenes

swarm writes everything in /spin1/swarm, under a user-specific directory:

```
/spin1/swarm/user/
├── 4506756 -> /spin1/swarm/user/YMaPNXtqEF
└── YMaPNXtqEF
    ├── cmd.0
    ├── cmd.1
    ├── cmd.2
    ├── cmd.3
    └── swarm.batch
```

swarm (running as the user) first creates a subdirectory within the user's directory with a completely random name.  The command scripts are named 'cmd.#', with # being the index of the job array.  The batch script is simply named 'swarm.batch'.  All of these are written into the temporary subdirectory.

### Details about the batch script

The batch script 'swarm.batch' hard-codes the path to the temporary subdirectory as the location of the command scripts.  This allows the swarm to be rerun, albeit with the same sbatch options.

The module function is initialized and modules are loaded in the batch script.  This limits the number of times 'module load' is called to once per swarm, but it also means that the user could overrule the environment within the swarm commands.

### What happens after submission

When a swarm job is successfully submitted to slurm, a jobid is obtained, and a symlink is created that points to the temporary directory.  This allows for simple identification of swarm array jobs running on the cluster.

If a submission fails, then no symlink will be created.

When a user runs swarm in development mode (--devel), no temporary directory or files are created. 

## Clean up

Because the space in /spin1/swarm is limited, old directories need to be removed.  We want to keep the directories and files around for a while to use in investigations, but not forever.  The leftovers are cleaned up daily by /usr/local/sbin/swarm_cleanup.pl in a root cron job on biowulf.  At the moment, subdirectories and their accompanying symlinks are deleted under these circumstances:

* Proper jobid symlink (123456789 -> XXXXXXXXXX, meaning that the job was successfully submitted)
  * the directory and symlink are removed **five days after the entire swarm job array has ended**
  * they are also removed if the modification time of the directory exceeds the --orphan_min_age (default = 60 days) and the user is not currently running any jobs -- this is because sacct only contains job information for a restricted period of time
* Job submission failed
  * the subdirectory and symlink are removed when the **modification time of the directory exceeds one week**

When run in --debug mode, swarm_cleanup.pl prints out a very nice description of what it might do:

```
$ swarm_cleanup.pl --debug
Getting jobs
Getting job states and ages since ending
Getting symlinks for real jobs
Walking through directories
user              basename          STA   AGE   DSE  TYP : ACTION  EXTRA
================================================================================
chenp4            4073854           Q/R   14.9  ---  LNK : KEEP    states
ebrittain         4400424           END   10.0  0.2  LNK : KEEP    FAILED
sudregp           4467927           END    8.9  8.9  LNK : DELETE  COMPLETED,FAILED
  rm -f /spin1/swarm/sudregp/4467927
  rm -rf /spin1/swarm/sudregp/XhlnoNq0EF
sudregp           4467928           SKP    6.9  ---  LNK : KEEP
bartesaghia       4501010           END    9.2  9.2  LNK : DELETE  COMPLETED
  rm -f /spin1/swarm/bartesaghia/4501010
  rm -rf /spin1/swarm/bartesaghia/dR_NPaOkr3
bartesaghia       4501011           END    9.1  9.1  LNK : DELETE  COMPLETED
  rm -f /spin1/swarm/bartesaghia/4501011
  rm -rf /spin1/swarm/bartesaghia/OePKGVUa9x
```

Each subdirectory is given as a single line.  The user and basename (jobid for successful submissions) start each line.  The other fields are:

**STA:** state of the job
* Q/R: queued or running
* END: the job has ended
* SKP: sacct was skipped, so no information is known about the job
* DEV: developemnt run
* FAIL: submission failed
* UNK: unknown state

**AGE:** modification time of the subdirectory

**DSE:** days since ending, only known for ended jobs

**TYP:** type of basename, either symlink (LNK) or directory (DIR)

**ACTION:** what action is taken, either (DELETE) or (KEEP)

**EXTRA:** extra information, such as the unique list of states of all the subjobs within the swarm

## Indexes

Two index files are created within the /swarm directory:

* .tempdir.idx -- contains a timestamp, user, tempname, number of subjobs, and p value for the swarm
* .swarm.idx -- contains a timestamp, tempname, and slurm jobid for the job array

These index files will be used in the future for managing swarms.

## Logging

* swarm logs to /usr/local/logs/swarm.log
* swarm_cleanup.pl logs to /usr/local/logs/swarm_cleanup.log

## Testing

Swarm has several options for testing things.

**--devel:** This option prevents swarm from creating command or batch scripts, prevents it from actually submitting to sbatch, and prevents it from logging to the standard logfile.  It also increases the verbosity level of swarm.

**--verbose:** This option makes swarm more chatty, and accepts an integer from between 0 (silent) and 4.  Running a swarm with many commands at level 4 will give a lot of output, so beware.

**--debug:** This option is similar to --devel, except that the scripts are actually created.  The temporary directory for the swarm.batch and command scripts begins with 'dev', rather than 'tmp' like normal.

**--no-run:** A hidden alacarte option, prevents swarm from actually submitting to sbatch.

**--no-log:** A hidden alacarte option, prevents swarm from logging.

**--logfile:** A hidden alacarte option, redirects the logfile from the standard logfile to one of your choice.

**--no-scripts:** Don't create command and batch scripts.

In the tests subdirectory, there are two scripts that can be run to test the current build of swarm.  **test.sh** runs a series of swarm commands that are expected to succeed, and **fail.sh** runs a series of swarm commands that are expected to fail.  They are run in **--devel** mode, so nothing is ever submitted to the cluster nor logged.

The script **sample.pl** extracts the last 100 or so lines from the swarm logfile and generates possible options for testing swarm.  The **--sbatch** option is screwed up because it doesn't contain any quotes, so you will need to add those back in to construct proper swarm commands.
