---
title : What needle are you trying to move?
date : 2026-06-16
summary : "The gap between good engineers and great ones isn't technical skill. It's knowing which needle to move and when."
tags : ["work-life-balance", "productivity", "softwareengineering", "leadership"]
---

Most engineers join a new team with a clear picture of what good looks like. Six months later, that picture is gone, replaced by deployment rotations, massaging egos, random meetings and a tech debt ticket that's been "prioritized next quarter" for three quarters running. The ambition didn't disappear. The focus did.
I’ve found this pattern strangely familiar and common in my experience. 
I used to call it burnout. Eventually I thought maybe the team was not picking enough slack, 
but now I think it’s a lack of direction, figuring out what to work on, figuring out what needle you’re trying to move. 

There is something to be said about the team not picking up enough slack which leads me to the next section.

## Drop the ball

I think it’s critical to “drop the ball” when necessary. 
Compensating for a team's lack of work ethic or skill is not a great idea, 
and you’d be surprised how hard I’ve tried to convince myself that I wasn’t doing it. 

If the team is not picking up slack, that may not be your problem.
It is our responsibility to give feedback to peers and your manager and..that’s it. 
Unless you’re in a management position or hold considerable influence over the team, this is pretty
much all you can do. 

Deployments don’t happen unless you start the conversation? Drop it.
Code issues don’t get flagged by the team unless you review it? Let it through. 
Team missed an important deadline? Let it. 

When you drop the ball, whoever is responsible for the miss gets a strong signal which otherwise 
would have been lost. It is very important that this signal not be lost. 
It brings up the real issues in the team which would otherwise go unnoticed. 
Usually leads to burnout, lots of low-priority high-urgency tasks which should’ve been planned better. 
This might seem like I’m asking you not to care about the product, some folks may even gaslight you into 
thinking you don’t. Don’t listen to them.
Covering for non-ideal behaviour in a team enables the product to hobble on, 
never really going anywhere.

You see the gaps in the team, you see who’s performing well and who’s not. 
The team should ideally sort itself out given enough time. 
And if it does not, then you get a strong signal about the team and can make your next decision based on that.
Drop the ball, let the signal take its course and see where it leads. 

Let's talk about the product.

## The product wins, always

At the end of the day you’re trying to get the product to succeed. In my experience, most engineers get caught in this trap of looking at code or the system as the end product itself rather than solving end user concerns. This misconception costs a lot.

Engineers usually think of every problem as code, which is not half wrong but it does eventually lead us 
comfortably into limbo. Let me explain. 

We start by being handed a problem to solve, a product. We solve it and eventually the code becomes 
large enough we are no longer able to fit both the codebase and the problem it’s trying to solve in our mental 
model.The code begins to whisper to you about your CI pipeline not being the most efficient, 
the tests not running fast enough, some initial proof of concept being used as production code. 

This is the point where it feels like getting buy-in to push this through is a monumental task.

But we’re missing a piece of the puzzle. We do not have the full picture of the problem yet. 
The other piece of the puzzle might be leadership forcing the team to push out a feature, 
your manager trying to prove the team's worth by prioritizing some other piece of work, or just the fact 
that you have only 2 engineers working on the product. 
We’re seeing the problem through a telescope, all nicely zoomed in on the piece you’re trying to solve. 
The problem is actually much more constrained, which is what brings real complexity 
(People are way more complex than code). In an ideal world with enough capacity, time, money 
or whatever hidden impediment you don’t know about, solving for code is pretty straightforward 
(Especially with the AI coding agents) but our weakness lies in not understanding the full constraints of the 
problem.

Don’t get me wrong, all the problems mentioned above are definitely worth investing resources into, but it 
becomes a question of investing now, or waiting for later, based on whatever else is pushing the fate of the 
product. 

You could cause a lot of frustration by pushing too hard for the work to be prioritized and end up leaving a bad impression. Solving this feels like a dance in communication and tact. We need to let the product win, and therefore ourselves.

In some big enterprises, the problems we are solving look extremely different to the problems we solve in an 
early-stage product going from zero to one. The problems (and their solutions) cannot be generalized 
across. 
My experience with an early-stage product was rough for a couple of months because 
I was looking at it through the experience of an engineer who worked in a big enterprise with a very 
mature tech-ecosystem. I was worried about the wrong things, and it feels extremely counterintuitive.

In an early-stage product it's more important to deliver features quickly than to get the code right 
(AI slop seems to be making this true for every org, ugh). This feels wrong, but it has to be done if required.
It’s a balancing act.

Introduce a cache layer and reduce latency by some ms vs push out a new feature some VP was asking for. 
Ideally both. But we all know we don’t live in that utopia. Would reducing latency benefit the users enough
to convince leadership it’s worth the extra cost? Or is it better you deliver a new feature that lets the 
product prove itself? These are all questions worth asking.

Compare this to a mature product already deriving value from their spends, reducing latency would be a huge win given some extra $$$, since at their scale the economy plays in their favor. They aren’t scaling per user like an early-stage product, they’re scaling per thousand users or per millions of users. Deleting code to reduce maintenance might be extremely useful on a product that has hundreds of engineers working on it. They might just be able to squeeze out more work because you just freed up some of their mental capacity. The amount of money you have to get from zero to one is drastically different from, let's say, an established product  already earning enough money to support your technical endeavours. 

Different problem, different needle. The product still wins, always.

## Finding your work 

The real question is never "what should I work on?" It's "am I still aligned with the product?"

When you are, the decisions mostly make themselves. Work that moves the product forward is your work.

When you can't quantify the value of a piece of work at all, that's a signal. 
It doesn't mean the work has no value, it means you haven't understood it well enough yet. Dev hours saved, 
cognitive load freed up, a future incident that doesn't happen are all good value metrics. If you still can't find a proxy, be honest about whether the product actually needs it.

If work seems repetitive and already has a reference for it, 
there’s no value in doing it unless the speed of delivery is the value. 
You should document your way out of the work. Delegate. Document everything anyway, 
not as good practice but as a way of ensuring you're never “impossible” to replace for the wrong reasons. 
The engineer who is critical to the product because they own the hard problems is very different from the 
engineer who is critical because nobody else knows how to run the deployment script.

Spending time on these tasks pulls you off the vector you should be on.

No one to delegate to? Do it if it’s temporary, hire for it eventually or automate it, 
but if it looks permanent then that’s not an edge case, it’s the job. 
You have to think about whether that’s the job you want. 
Dry work like dashboarding and maintenance is a good signal too. 
It could mean your product is stabilizing enough to think hard about which direction to move in, 
in which case there’s a possibility of pushing the product to be better through harder problems. 

If it feels like that is the end goal, it might imply the product isn’t really moving forward or 
has other issues that are stopping it from growing. This is also an opportunity to figure out if 
that’s a problem for you to solve, whether you’re still in alignment with the product.

I spent years brute forcing whatever was handed to me while simultaneously trying to find the work 
I actually wanted to do. The cost was time. Time I could've spent arguing for the hard problems, time 
I could've spent moving metrics that actually mattered. The framework I've described here would've saved 
me a lot of that. Not because it's complicated, but because it reduces every decision to one check: are 
you and the product still moving in the same direction?

If yes, find the most valuable thing you can do for it right now and do that.

If no, that's the most important thing you've learned about your current situation.
