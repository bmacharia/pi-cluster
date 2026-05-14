## Symptons vs Root Causes

A sympton is a visible problem. For example, the common cold has visible symptons like coughing, sneezing, and a high temperature. The root cause of the symptons are the virus that causes the cold. The mitigaton is the action taken to manage the symptons of the cold as the body's immune sympton and the cellular and molecular level battle the virus.

So what does this have to do with Kubernetes, or better yet SRE principles of Symptons and Root causes. That is an excellent question.  The Root cause is the underlying condition that caused the problem, and the symptons are the visible attributes of the root cause.

In short the the Sympton is what one sees. The Mitigation is what makes the sympton stop, and the root cause is why the sympton happened.
 
What we really want to learn is the why. Why did it happen, what we learned, and how we can avoid it happening again.

So with that being said I will be conduction lab scenarios on my Kubernetes cluster that will force me to write.

1. What was the sympton?
2. What was the mitigation?
3. What was the root cause?
4. What would prevent the reurrnece?

What this will do is build a Diagnostice Discipline. a root-cause workflow

1. Reconstruct the timeline.
2. Identify the trigger.
3. Map cause to effect.
4. Categorize the cause.
5. Plan and execute the fix.

Also I will be authoring Postmortems with the following structure

1. Summary
2. Timeline
3. Root Cause
4. What Worked
5. What Did Not Work
6. Action Items