Stage 3 is UpdateSeek(). Its job is to take the ideal slot point from GetSeekPosition() and turn it into a better moving chase target before steering.
Code: AIActionRam.cpp:177
Start state:
•
seekPosition = the ideal world-space slot beside/behind the target
•
but that point is tied to a moving car
•
if you steer at it naively, you tend to lag behind and oscillate
So UpdateSeek() does this.
1.
Gather motion state
It reads:
•
my position AIActionRam.cpp:178
•
my forward vector AIActionRam.cpp:179
•
my velocity AIActionRam.cpp:181
•
target velocity AIActionRam.cpp:183
If my velocity is almost zero, it substitutes my forward vector:
•
AIActionRam.cpp:186
That matters because pursuit math needs a meaningful “my current movement direction.”
If the car is nearly stopped, velocity is noisy/useless, so forward is more stable.
2.
Start from the raw slot
It initializes:
•
newseek = seekPosition
•
AIActionRam.cpp:190
So if the target is not moving, it can just pursue the raw slot.
3.
If the target is moving, shift the slot forward along target motion
This block runs only when target speed is meaningful:
•
AIActionRam.cpp:191
It computes:
•
seekoff = seekPosition - myPosition
◦
the vector from me to the current slot
◦
AIActionRam.cpp:193
Then:
•
blah = Dot(seekoff, targetVelocity) / |targetVelocity| * 0.7
◦
AIActionRam.cpp:194
What this means:
•
Dot(seekoff, targetVelocity) / |targetVelocity| is basically the component of seekoff along the target’s direction of travel
•
in plain terms: “how far ahead of me is the slot in the direction the target is moving?”
•
then it multiplies by 0.7
•
that 0.7 is a damping/prediction factor, not full prediction
So if:
•
the slot is far ahead in the same direction the target is moving, blah is large positive
•
the slot is sideways relative to target motion, blah is small
•
the slot is behind along target motion, blah could go negative
Then it adjusts seekoff:
•
seekoff += targetVelocity * (blah / |targetVelocity|)
•
AIActionRam.cpp:195
Since targetVelocity / |targetVelocity| is target direction, this is:
•
“push the desired slot forward a bit along target travel, by an amount proportional to how far ahead that slot already is”
Then:
•
newseek = myPosition + seekoff
•
AIActionRam.cpp:196
So newseek becomes a predicted version of the slot, not the raw current slot.
4.
Feed that into pursuit steering
Finally:
•
AISteer::Pursuit(seek, position, velocity, newseek, targetVelocity);
•
AIActionRam.cpp:198
This is the real point of Stage 3.
It does not return “point to newseek directly”. It returns a steering vector seek based on:
•
my position
•
my velocity
•
the predicted target point newseek
•
target velocity
That means the steering output is aware of relative motion, not just geometry.
Why this helps near the slot Imagine the raw slot is right beside the perp, and both cars are moving fast.
If you aim at the raw slot:
•
by the time you turn toward it, the slot has moved
•
once you get close, the direction flips quickly
•
you overshoot and weave
Stage 3 reduces that by:
•
shifting the slot forward along target motion
•
then using pursuit steering instead of direct seek
So near the slot, the cop is chasing where the slot is going, not where it was.
Simple mental model Raw slot:
•
“I want to be here”
newseek:
•
“I want to be where that slot is drifting toward”
AISteer::Pursuit output:
•
“given both our motions, this is the direction I should move now”
Then Stage 4 turns that direction into a drive target ahead of the car.
Tiny numeric intuition Suppose:
•
target moves forward at 20 m/s
•
slot is 30 m ahead of me in that same direction
Then:
•
projection onto target direction is about 30
•
scaled by 0.7 gives about 21
•
so newseek gets pushed about 21 m farther along target motion
That means I stop chasing the current slot center and instead chase a point significantly ahead of it.
If you want, I can rewrite UpdateSeek() into clean pseudocode with descriptive variable names