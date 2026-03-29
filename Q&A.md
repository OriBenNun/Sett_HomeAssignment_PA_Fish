# Questions & Answers:
29/03/2026 | by: Ori Ben Nun | for: Shahaf @ Sett.ai

### Q: First, I'd like to give the full context for "What happened with the first attempt" as it's very relevant for the rest of the Q&A.

### A:

I believe the first time I attempted the task I misunderstood the real
intention behind the instructions (although it was mentioned in bold in few places, in retrospect).
Although I read the instructions carefully and several times, 
I was mistakenly in the mindset of "**This is a vibe coding task**. As long as I have plan and 
implementation prompts documented once, and I'll follow the rules -
it'll be good enough as it's about the playable and how I do with AI".

However, as I got more into the task I had a persisting feeling that I was missing something.
During the weekend (about 2 hours of net work time at this point), 
when I re-read the instructions (once again), 
through the "third of the time" rule I realized that I was totally off.

I believe the actual mindset I should have had was "**This is an automation simulation task**.
The third of the time rule is there to prevent me from vibe coding a playable and that's it.
Instead, I should simulate an automated process by using most of the time to
iterate between planning and implementation, until I have a near-finished
product that can be 'fine-tuned' by vibe-coding and manual code fixes".
And this is the mindset I embraced for the second attempt.

The reason I chose to start over and not just "iterate back to planning" was that 
**the iterative process I followed was incorrect** for a true
"this is automation simulation task" mindset.
This is also the reason why the documentation of the first attempt is not as clear 
as the second attempt. I'm sorry for that in advance, and I still tried to make everything accessible 
through either the chat links or the git history.

**Sorry for the long answer (that you didn't ask for), but I wanted to be as clear and 
transparent as possible :)**

### Q: How did you pick and evaluate the plan to use for the implementation generation?

### A:

For the first few outputs (during the first attempt) I got from GPT I 
read them pretty carefully and looked for big decisions 
that I wasn't sure about (like rendering method), or a clear issue with the plan
(like wrong state machine flow).
Once I found something, I would send a message 
back to the chat and discuss it to understand the pitfall better.

That way, within few iterations, I already had a good idea of how better and worse plans look like,
and what I should look for in the future 
(like Three.js Vs. Canvas2D, embedded sounds, the gauge visuals, JS/TS, etc.)

At this stage (the 2nd attempt) I was already confident in my understanding of the PRD
and the prompt that I've improved the more I learned
about the 'hard-to-carck-nuts' and repeating bugs of this PRD.
So I didn't actually want to read the whole
plan MD anymore (both to save time and to better simulate automation),
but instead I read the main decisions GPT made 
(which it sent me alongside the plan file itself), and just quickly went over the plan to check 
there isn't anything that would raise red flags.
As long as it looked good overall - I would pass it over to the implementation chat.

Eventually, the chosen plan was the one that yielded the best implementation in one-go.
Honestly, I couldn't even tell if it's the "best plan" as I didn't read and compared it manually
to the others.

**For me the idea was: the best plan is the one that 
yields the best output for the next step in the process with minimal intervention.**

### Q: How did you pick the initial implementation version you modified to create the final playable?

### A:

[this part describes only the process of the 2nd attempt, as for the 1st attempt I haven't followed a systematic process. It was mainly exploration and experimentation.]

Very similar idea to the one mentioned above, but this time I just needed to play it
in Claude's playground in the chat itself, and test:
1. Has it fallen into one of the known pitfalls?
2. Does it have game-breaking bugs?
3. How close it is to the example playable?

If it fails 1 or 2, it means going back to planning.

After few iterations between planning and implementation, I reached a point where my 
prompt adjustments didn't yield any better results (both in the plan and implementation chats), so I just stopped and decided to move on 
with the best version I saw so far (which needed only a small UI change to be fully functional).

**The idea was to start with the version which would need the least amount of
work outside the automated process (even if it's done with vibe coding).**

Also worth mentioning that I have worked on a few games so far, in many sizes and for many platforms,
so I believe I have a pretty decent instinct about what are small, easy-to-fix
issues that don't need a full re-iteration of the process. And from my experience, for those types of issues,
it matters very little what the technology is (and same goes for hard to fix issues, which I have dealt with, but during the planning).
Therefore, I felt quite comfortable when deciding to stop improving the prompts and choose the best version to move on with.

### Q: Describe the process you followed for this task.

### A:

I believe I answered a lot of the question already in my first context answer.
But if I'm looking at the two attempts as one continuous process,
I believe this is a good mental model for it:

1st attempt was about **Research through trial and error:** Understanding the problem domain, 
the common pitfalls with HTML5 PAs in general and this specific PRD,
and what GPT is good/bad at with this domain (which is new to me).

2nd attempt was about **Simulating an automated process**, and therefore I tried to have
a much more systematic process:
1. Plan
2. Implementation
3. Test
4. Iterate

Then, once I had the best initial implementation, I moved on to the bug fixes and game-feel:
1. Many small changes in one go (via vibe coding + manual fixes directly in the HTML file)
2. Play
3. Iterate

Overall, I believe the process as a whole was very fruitful (for me personally, at least) as
the first attempt allowed me to learn a lot about the domain,
while the second attempt allowed me to focus on the process and the
"let's simulate agentic automation pipeline" mindset.

I enjoyed it a lot :)

### Q: Describe how much time approximately you spend on planning generation, implementation generation and on the fixes to get the final playable version.

### A:

I think it would be reasonable to look again at the two attempts as one continuous process.

The first attempt took me about 2 hours, and it was about iterating over the same plan few times,
then iterating over the implementation while still dealing with the big stuff.
Basically each iteration, although didn't necessarily have a new plan MD because I messed the process,
was pretty much a new plan + implementation iteration because the changes were pretty major.

So ~2 hours of the first attempt go to Plan + Implementation iterations (again, sorry for the mess, but it's hard
to approximate them separately here).

For the 2nd attempt, I think it took me about ~4 hours of net time.
Out of that:
- ~1.5 hour of Planning iterations
- ~0.5 hour of Implementation iterations
- ~2 hours of iterations on the final implementation

Eventually, after ~6 hours total net time, I decided to stop and start wrapping up.
Not because I couldn't improve the playable, or I didn't know how to (I barely
improved the balance, juice and feel. Mainly touched the UI). I stopped because I was already
about 20% over the 5 hours suggested time, and I hoped that my skills and process have been
already demonstrated well enough by then.

## So in total:
- ~2.5 hours of Planning
- ~1.5 hours of Implementation
- ~2 hours of iterations on the final implementation

I would note that this is only net work time, but I spent much of the time since I got the task
thinking about it, reading the instructions, reading the PRD, playing the example playable, etc.
(I also worked on the task from 3 different devices, 3 different OSs, some with VPN,
which makes time counting even harder lol).

I'm also aware that the time I spent iterating over a single implementation is
"stretching it" a bit in terms of the third of the time rule.
However, I hope that the process as a whole, including me realizing my mistakes and trying to learn from them, 
can show the true quality of my process and the effort put into the task.

Thank you for reading and for the opportunity, sorry for the long answers :)