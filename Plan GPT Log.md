# Model: Claude Sonnet 4.6 Extended



Your task:

&#x09;	Read the PRD carefully and produce a fully detailed technical plan describing how to implement the playable in a single HTML file. The plan should follow the PRD very strictly; you may add non-mentioned features only if they are trivial. This is not an actual playable ad, so no need to deal with ads and 3rd parties, but it should 100% look and feel like a PA.

The plan should include:

&#x09;	1.  Core systems and architecture.

&#x09;	2.  Gameplay flow.

&#x09;	3.  Entities and data.

&#x20;   4. VFX, SFX and particles

&#x20;   5. 3D visual approach

&#x09;	6.  Step by step implementation plan.



Other important things to pay extra attention for while reading the PRD and to describe very specifically in the plan:

1\. How the gauge looks and works (semi-circle, going bad<->perfect<->bad, pendulum with ease movement, etc)

2\. The 3 audio files (names below) are the only assets that we have for the PA, and will be provided along with the implementation plan.

background\_music.mp3

catch.mp3

rob\_use.mp3

3\. The end screen should only show a CTA which opens a blank new tab (for demonstration purposes).

4\. The round counter should be hidden.

5\. Rarer fish should feel much better visually and juice-wise.



Output format: single, clear Markdown document (up to 5 pages) meant to be read by a senior developer



9:10 PM



The plan is ready. Here's a quick orientation of the most opinionated decisions in it:



\*\*Gauge\*\* — driven by a pure `Math.sin(elapsed \* freq)` which gives free ease-in/out at the extremes without any tweening library. The arc is SVG with a red→yellow→green→yellow→red gradient baked in so the player instantly reads "center = deep = good." Frequency increases each round for urgency.



\*\*3D without real assets\*\* — Three.js r128 from CDN with primitive meshes only. Legendary fish get an animated point-light child and hue-shifting emissive so they glow and feel alive, while common fish are just flat grey boxes. The water is a custom ShaderMaterial so it undulates without any extra library.



\*\*Audio with 3 files\*\* — `catch.mp3` does triple duty: played at 1.0× for common, up to 1.4× playbackRate for legendary — no extra files, but a clear escalation the player feels subconsciously.



\*\*Rarity contrast\*\* — the plan deliberately over-invests VFX budget on legendary (full-screen flash, 60 particles, screen shake, camera hold) and under-invests on common (8 sparkles), so the gap is unmistakable.



\*\*No round counter, no CTA in-game\*\* — both strictly enforced by the architecture: round is a private integer that never touches the DOM, and the only CTA is the END\_SCREEN button.

