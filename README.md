Lerp in TF2 is calculated like so:
`lerp = MAX(cl_interp, cl_interp_ratio / cl_updaterate)`

15.2 is the lowest achievable lerp value, as the `cl_interp_ratio` cvar (convar) is server-bound to `sv_client_min_interp_ratio`, thus it can never go below 1.

Some examples for lerp values:
- 15.2 = `cl_interp_ratio 1` (also the interval of ticks in TF2)
- 30.3 = `cl_interp_ratio 2`
- 45.45 = `cl_interp_ratio 3`

Do note: When I refer to lerp like 15.2, I mean in milliseconds. The game sees them as seconds, so in actuality it's looks like 0.0152, 0.0303, etc.

## The Relationship Between Lag-Compensation and Lerp in TF2
The client is **always** behind the server since it determines the game's state in TF2. Valve's solution to handling this is client-side interpolation, prediction, and lag compensation (often called backtracking).

For lag compensation, the server stores snapshots, often called lag records. These snapshots contain information about the player, such as: position, angles, animation data, hitbox size, and the time of the record. The server stores a record every time it sends out updates to all of the clients. It then  erases all records that are older than a second.

Every time the server sends out an update containing player data, the client stores this information. It then uses interpolation to smooth out the gaps between updates every frame. This is done to prevent choppiness and is heavily controlled by the interpolation delay (lerp).

Every time you attack someone, the server starts lag compensation which takes into account a couple of things:
- the client's outgoing latency (ping)
- final lerp (as seen in `net_graph`)
- the tick of when the attack happened
There are more variables that play into this, but they'll be ignored for the sake of keeping this simpler.

The lag compensation process determines the most optimal lag record by:
- taking the tick of the attack and subtracting it by the client's lerp in ticks (see below), we'll call it "lerp time"
- it then attempts to calculate a corrected time by adding the outgoing latency and lerp, clamped to the maximum backtrack time (which is 1 second by default, it's defined in `sv_maxunlag`)
- finally, it compares the difference between our lerp time and the corrected time using server time. if the difference is over 200ms, it'll adjust the target time to use the corrected time which is based off-of server timers instead of client reported time.

Lerp values in ticks:
- 15.2 = 1 tick, 
- 30.3 = 2 ticks, 
- 45.45 = 3 ticks, etc.
To put it in perspective, a tick happens in TF2 once every 15.15ms (which is 1/tickrate=1/66).

So, let's say you backstab someone and the target time is 2499, that means the lag record used for backtracking will correspond to where the player was at that specific tick. Client interpolation does not guarantee that the variables will match their values on the server, although it does a pretty good job at approximating them.

The real problem with 15.2 lerp is that the selected record won't be aligned with the interpolated data, practically it will be ahead of where interpolation says the player will be at a given tick. Hence, you'll miss way more often. With 30.3 lerp, the selected record is more accurate and aligned with interpolation. Stabs that should work do work far more with 30.3 lerp.

At lower lerp values, lag compensation is going to rely on extrapolation, which increases the odds of missing. 30.3 lerp seems to yield accurate positions, whereas 45.45/60.60 lerp seem to yield positions much behind the model. As I said before, lower lerp values yield positions ahead of the model. This video demonstrates it pretty well: https://youtu.be/EZhGLGMoY68

However, using 30.3 lerp might come at a cost. Player angles are interpolated slower, which might make it harder to predict when you can backstab someone. This is not as much of an issue after getting accustomed to it, although I can see why this would affect people's opinions on higher lerp values. Using 30.3 lerp might also not be optimal for people who get lower ping. 

In conclusion, 15.2 lerp messes up backtrack, requires more time-critical inputs, and has a smaller window to hit backstabs, whereas 30.3 lerp is much more forgiving and has a much larger window to hit backstabs.

##### code for reference 
- https://github.com/ValveSoftware/source-sdk-2013/blob/48809cb86c0990b35f190404aa85d0483d306fa4/src/game/server/player_lagcompensation.cpp#L345
- https://github.com/ValveSoftware/source-sdk-2013/blob/48809cb86c0990b35f190404aa85d0483d306fa4/src/game/server/player_lagcompensation.cpp#L440
