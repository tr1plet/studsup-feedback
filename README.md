# Studs Up! — aerial play & volley feedback

Sent via the in-game report tool on Nov 6, 2026. I verified every finding by reading the GDScript from my own installed copy (no code quoted, findings only). The in-game fields have a 2000 char cap, so the full write-up lives here.

---

VOLLEY + AERIAL PLAY: WHAT'S WRONG AND HOW TO FIX IT

I've played since launch day. The ground game is the best football feel on PC right now. The air game drags it down, and I wanted to know exactly why, so I read the GDScript from my own installed copy. No code quoted below, only findings, each anchored to a system you already ship. Two files matter for this report: the player controller and the FootHeight helper.

PART 1: WHY VOLLEYS FEEL BROKEN (VERIFIED)

1. The foot lives on a 2D rail system. In control mode the foot target moves on the body's XZ plane, constrained to the dribble ellipse (about 0.65m to the sides, 0.9m forward, 0.6m back) or the foot box (x -0.7 to 0.8, z -0.7 to -0.15). Elevation is a separate value. The foot can't travel sideways around a ball, so you can't strike across it, only push it roughly forward.

2. The manual height binds can't work in live play. foot_height_up/down steps 0.05m per press across a 0 to 1.0m range. That's twenty presses to reach max lift while a cross comes down. Great as a fine-tune in practice mode, dead in a match.

3. The auto-lift disengages in silence. Tracking runs only while ALL of these hold: ball within 3.0m horizontally, ball at least 0.34m above foot rest, ball rising slower than 1.5 m/s. Cap sits at 0.8m (volley_track_height). Any gate fails and the lift stops, with no feedback. Players read this as randomness.

4. Volley swings run flat. Once _kick_is_volley is set, the swing holds the captured foot height and adds follow-through, so the foot travels a horizontal line at one height. It never passes through the ball's flight path. Contact quality punishes height error on top of that: clean band is about 0.12m, and past 0.32m of error the shot drops to 75% power. The elevation must be perfect while the swing itself can't correct toward the ball.

5. The camera locks in control mode. With _free_look off, the mouse doesn't rotate the body at all. Only A/D turns, at turn_speed. You can't open your hips or rotate into the strike, which is half of what a real volley is.

6. Airborne options are headers only. Volley tracking does keep running in the air, but the foot base rides the leg tuck, so the reference height shifts while you aim. That moving baseline is a lot of the "wanky" aerial feel.

PART 2: WHAT YOU ALREADY HAVE

This sets the price tag of the fix:

- Ball flight prediction. Gravity at 1.75x scale, drag, Magnus spin, bounce handling, running every frame to feed auto-lift. It can hand you the intercept point anywhere on the pitch at any moment.
- An impact model that already handles airborne balls. Loft blends by ball height, the impulse includes body velocity, leg speed saturates at the top end. The math for a bicycle kick already exists.
- Landing impact measurement on touchdown. It drives the landing squash today.
- Settings infrastructure. auto_volley_tracking already lives there as a toggle.

The air game is wiring work on systems you already ship.

PART 3: THE PROPOSAL, AERIAL CONTROL MODE

Core change: gate the foot constraint on grounded state. Same input, same mode. Grounded keeps the ellipse and the box, untouched, so every ground mechanic follows its current code path. Airborne detaches the foot target from the plane.

Step 1: reach sphere (fixes findings 1, 2, 6).
Airborne, the foot target moves on a sphere around the hip pivot, radius about 0.9m to match the existing forward reach. Mouse X sweeps azimuth around the body, mouse Y sweeps elevation from ankle to overhead. Two mouse axes, no new binds. Clamp azimuth and elevation so poses stay anatomically legal, and clamp target speed so the whip stays readable.

Step 2: arc swing through the intercept (fixes finding 4).
When a volley kick fires airborne, take the predicted arrival point from the tracker and make it the apex of the swing. The foot travels an arc from its current position through that point, instead of a flat line at captured height. Power scales with target speed at contact plus fall speed, and the impulse model already consumes both. Mishit windows can stay exactly as they are.

Step 3: torso follow (this is what unlocks the bicycle kick).
When the foot target goes above shoulder height, tilt the pelvis and torso toward the target, capped around 120 degrees of pitch (reads as a full bicycle pose without full inversion), damped, springing back to upright when the target drops. Pull the target up and behind the head and the body ends up in backflip position, which is the bicycle kick stance.

Step 4: partial camera coupling (fixes finding 5).
First person at full body pitch will sicken half the playerbase. Couple the camera to torso tilt at 30 to 50 percent, damped, same smoothing style _turn_body uses. Expose it in settings as aerial body cam: partial / full / off, default partial.

Step 5: landing, reuse what exists.
You already measure landing impact. Trigger the fall and get-up when impact passes a threshold or the torso sits more than 50 to 60 degrees off upright at touchdown. Below those numbers, land on feet as it works now, so small hops never interrupt play. For v1 reuse the tackled fall plus get-up. A short ragdoll before get-up fits the physics identity and can wait for a later pass.

Step 6: what stays untouched.
Ground path, ball physics, impact math. High boots in aerial duels become ref territory, which is the point of having cards. Keepers gain high-ball reach, which they need.

PART 4: QUICK WINS IF THE FULL MODE WAITS FOR A LATER PATCH

- Kill the silent disengage (fixes finding 3). A tiny indicator when a gate (range, height, rising speed) fails would fix most of the "random" feeling.
- Raise or expose the 3.0m range and 0.34m floor. They cut off plenty of legitimate volley chances.
- Arc the swing through the predicted intercept even for grounded volleys (softens finding 4 today). Single biggest feel improvement in this list.
- Ball landing marker. Information only, no automation, matches the game's identity.

CLOSING

The loop this hands the player: cross comes in, jump, control mode, mouse up tilts the body back, foot whips through the predicted arc, overhead volley, roll through the fall, get up, celebrate.

The ground game already beats the genre. Reading the code convinced me the air game was one branch away the whole time: the predictor, the airborne impact math, and the landing data are all in place. Volleys and bicycle kicks are the clips that sell a game like this one. If only one thing from a suggestion box makes the roadmap this year, I'd want it to be this.
