# xgolock
alternative to xautolock
# why 
xautolock is annoying to work with, and alternative tools doesnt quite provide the functionality nedeed, so i decided to try and implement my own
## core 
core module is scheduler, that allows to build nested scheduled jobs, to build off of it xautolock and xidlelock alternative  
scheduler should be able to launch jobs by different triggers, not only by time  
also scheduler should be able to launch jobs by their ids, and take shortest path to the job it needs to run.
Jobs should have three stages: start, subjobs, stop, and triggers to start it, so basically when the job is started it creates a trigger, which is once triggered starts it, then the subjobs, and after that stops current job.
jobs should be able to fall off - trigger should be able not only start the job, but altogether prevent the job from executing, and stop all current jobs, or skipp the job - more in examples  
### exapmles
here ill use yaml kind of syntax to describe how jobs looks like and how different behaviours should be handled
```yaml
-   trigger: inactive 20s, falloff if active # launches after 20s
    start: # start 1st job
    subjobs: # none
    stop: # stops 2nd job
    
-   trigger: inactive 20s, skip if active 
    start: start 2 job
    subjobs: # none
    stop: 
    
-   trigger: inactive 20s, falloff if active 
    start: start 3 job
    subjobs: # none
    stop: 
```
here after scheduler launched, it will wait 20s, if trigger oject recieves any events, it will trigger falloff function and the scheduler will stop imidiatelly.
After 20s of no events it will start 1st job, wait for it execution and imidiatelly stop it after it executes.
2nd job will also wait for 20s of no events before starting, but trigger can call skip function, so if it recieves an event, it will skip right to the next job. 
If it recieves the signal after job is started current behaviour is not to do anything, wait for it to finish and then call stop, but i think we can add a setting if a job should be terminated.

Then after that there is a third job that works exactly as 1st  
Example 2  
lets actually look at the example of a job that is related to locking
```yaml
-   trigger: inactive 30s, falloff if active # launches after 20s
    start: # dim lights
    mode: wait for subjobs
    subjobs: 
        -   trigger: inactive 90s, fallof if active
            start: # enable power saving and show notification
            mode: wait for subjobs
            subjobs:
                -   trigger: inactive 360s, falloff if active
                    start: # launch locker
                    mode: falloff subjobs on job finished
                    subjobs: 
                        -   trigger: inactive 3600s
                            start: # hybernate
            stop: # restore power saving mode and show notification
                
    stop: # restore light level
```
so here after 20s lights will dim, after another 90s power saving mode will be switched and notification will be shown. after another 3 min of inactivity locker will be launched, but, as stated in previous example, even after recieving a signal of activity, we will wait until locker is finished - wich means that nothing will be undone b4 we unlock. And finally there is an hour timer to hybernate, but if it will recieve activity it'll just restart the clock.
There is still issues with that kind of scheduler( how for example will you undim lights after regained activity but lock is launched ) but i think for now ill stick to it, after implementing a basis ill think what i can do about it

There is also no examples of launching a job directly bypassing triggers, so here is one, lets assume that in a job from previous example we assigned each job some id
```yaml
-   trigger: inactive 30s, falloff if active # launches after 20s
    start: # dim lights
    id: 1
    subjobs: 
        -   trigger: inactive 90s, fallof if active
            start: # enable power saving and show notification
            id: 2
            subjobs:
                -   trigger: inactive 360s, falloff if active
                    start: # launch locker
                    id: 3
                    subjobs: 
                        -   trigger: inactive 3600s
                            start: # hybernate
            stop: # restore power saving mode and show notification
                
    stop: # restore light level
```
and if we wanted to launch job 2 bypassing all the triggers this will happen:
- immediately `1` will be executed bypassing its triggers
- immediately `2` will be executed bypassing its triggers
- immediately `3` will be executed bypassing its triggers
- job `4` will be scheduled
So now all we have to do to lock immediately is to give locker job some unique known id, and call it, finishing all nested jobs up to nearest parrent of current job and locker job, and going down to the locker job bypassing the triggers
