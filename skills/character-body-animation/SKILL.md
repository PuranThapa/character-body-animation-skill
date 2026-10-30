---
name: "character-body-animation"
description: "Use when animating any human or humanoid character in code (canvas, SVG, WebGL): natural joints, walks, sitting, gestures and body language without a robotic look."
---

# Character Body Animation (code-drawn characters)

Rules for making code-animated characters move like cartoon/anime characters instead of robots. They apply to any style (chibi, anime, western cartoon, realistic) and any rig (2D canvas, SVG, Three.js). Each rule comes from a real defect found and fixed in practice; the "Defect checklist" at the end lists them so every new animation can be checked against them.

## 0. Core idea

A character has two layers:
1. **Skeleton (invisible):** joint positions only (hip, knee, ankle, shoulder, elbow, wrist, neck, head). Used for maths, never drawn. Offer a "show bones" debug toggle.
2. **Skin (visible):** continuous shapes drawn *over* the skeleton. Limbs, torso and clothes must read as one body, never as parts pinned together.

If anything on screen looks like "two lines joined and bent", the skin is following the skeleton too literally.

## 1. Proportions and skeleton setup

- Define every length in one scale unit `s` per character (head radius, torso, upper arm, forearm, thigh, shin), so characters of different sizes share the code.
- Typical anime/chibi: head about 1/3 of body height; realistic: head about 1/7 to 1/8.
- Upper arm is roughly equal to forearm; thigh is roughly equal to shin.
- **Shoulder joint sits at the outer edge of the chest, a little below the top of the shirt line** (about 15-18% of torso height down). Too high makes shoulder-pad bumps; too far inward makes the arms hang over the chest.
- **Hip joints:** about 1/4 of hip width apart when facing front; almost 0 apart when side-on.
- Place the hips each frame so the **lowest foot touches the ground** (`hipY = ground - max(footY) - soleThickness - jump`). This gives natural bob while walking and crouching, and no floating or sinking.

## 2. Drawing limbs without visible joints

### 2.1 One continuous shape per limb
- Draw shoulder to elbow to wrist (and hip to knee to ankle) as **one path**, never as separate segments.
- **Fillet the joint:** replace the sharp corner at the elbow or knee with a curve whose size grows with the bend angle:
  `filletLen = min(upperLen, lowerLen) * (0.22 + 0.4 * bendAngle / PI)`.
  A single quadratic curve through the joint still creases on tight bends; the growing fillet fixes that.
- **Taper:** draw the limb as a filled ribbon whose width varies along its length (arm: thick at shoulder, thin at wrist, about `w*(1.18 - 0.32u)`; leg: add a slight calf bulge around u = 0.66).
- **Outline pass, then fill pass:** draw the ribbon slightly wider in the outline colour first, then the fill on top. Draw the hand or foot outline in the same outline pass, so the hand joins the wrist with no seam.
- **Elbow and knee crease:** only for bends over about 77 degrees (1.35 rad). Draw it as a short, faint curve on the inner side, with opacity rising with the bend (max about 0.45). Always-on creases look like scratches.

### 2.2 Shoulders blend into the torso
The shoulder is the most visible seam. Draw order per arm:
1. **Sleeve outline before the torso.** It then only shows where the sleeve sticks out past the body.
2. **Torso** (fill and outline).
3. **Bare forearm** (outline, then fill, then hand).
4. **Sleeve edge lines fading in:** opacity about 0.2 near the shoulder, rising to 1 by mid-sleeve. The arm appears to grow out of the body.
5. **Sleeve fill on top.** It covers the torso outline where they overlap, so shirt and sleeve read as one piece of fabric.
6. If the elbow bend is over about 86 degrees (1.5 rad) and the forearm folds back over the sleeve (hand to face), **redraw the forearm on top**: it comes toward the camera.

- Make the sleeve root only slightly wider than the sleeve; a wide root makes a bump above the shirt line.
- **Shrug:** raise the shoulder joint a few pixels when the arm lifts above about 50 degrees (`lift = k * clamp((|angle| - 0.9) / 1.4, 0, 1)`). Only lift when the arm is up.
- Use the **same blended method for every arm in every pose** (front, turned, far arm, near arm). Mixing the old and new method lets seams return in some frames.

### 2.3 Hands and feet
- Hands are simple shapes rotated along the forearm: a fist (palm ellipse, thumb bump, 2 knuckle lines), an open mitten, or a pointing hand. Swap between 3-5 hand poses; don't animate every finger.
- A plain circle for a hand reads as a ring, especially in front of the face. Avoid it.
- When holding an object, draw the object first, then the fist on top, so fingers wrap it.
- Feet point in the direction of travel and get longer as the body turns side-on.

## 3. Joint direction (inverse kinematics)

Use 2-bone IK whenever a hand or foot must reach a target (holding hands, giving an object, touching the face, catching, sitting on a chair).

### 3.1 Always choose the anatomically valid solution
Two-bone IK has two answers (elbow in or out). **Never hard-code the bend sign per pose.** Solve both and score them:
```js
function ikOut(rig, side, target, forearmScale = 1) {
  const sh = side === 'L' ? rig.shL : rig.shR, out = side === 'L' ? -1 : 1;
  let best = null, bestScore = -Infinity;
  for (const bend of [1, -1]) {
    const r = ik(sh, target, rig.upper, rig.fore * forearmScale, bend);
    const ex = sh.x + Math.sin(r.a) * rig.upper, ey = sh.y + Math.cos(r.a) * rig.upper;
    const score = (ex - rig.hip.x) * out + 0.35 * (ey - sh.y); // outside the body, and lower
    if (score > bestScore) { bestScore = score; best = r; }
  }
  return best;
}
```
- Elbows point **outward and down**. They never fold across the chest when facing front.
- Knees point **forward**, toward the direction of travel or the direction the character faces. They never bend backward.

### 3.2 Foreshortening
A forearm or shin pointing toward the camera looks shorter. Give each limb segment a length scale (0.3 to 1, sprung). Examples:
- Holding something out in front: the upper arm hangs at the side, forearm scale about 0.4, hand just in front of the waist.
- Pointing at the viewer: upper arm forward (short), forearm about 0.3 to 0.5.
- Sitting, seen from the front: thighs come toward the camera, thigh scale about 0.35 to 0.5.

## 4. Body turn and depth

Give each character a `turn` value from -1 to 1 (0 = facing camera, ±1 = profile), driven by a spring. Turn affects:
- **Hip width:** `hipW * (1 - 0.85*|turn|)`. Side-on, both legs come from one point, so steps go forward and back instead of scissoring.
- **Shoulder width:** `* (1 - 0.5*|turn|)`. **Torso width:** `* (1 - 0.32*|turn|)`.
- **Near and far limbs:** turning toward screen-right brings the screen-left limbs nearer. Draw order: far arm, far leg, near leg, torso, near arm, head. Shade far limbs and the far half of the chest about 15-30% darker.
- **Only treat an arm as "behind the body" when clearly side-on (|turn| > 0.55).** A borderline threshold (for example 0.3) flickers between methods frame to frame.
- **Face:** eyes shift toward the turn; the far eye narrows (about 40%); the chin moves toward the turn; the fringe shifts slightly.
- **Feet:** offset and lengthen toward the turn.
- **Rest arms:** the outward hang angle shrinks as the turn grows (`rest * (1 - 0.6*|turn|)`).
- **Characters walking left or right must be turned (|turn| about 0.8).** A front-facing chest that moves sideways is the most common "looks wrong" defect.
- When two characters talk or interact, turn them slightly toward each other (about 0.3) instead of both facing the camera.

## 5. Motion quality

- **Springs on every joint:** move toward a target pose with a damped spring, never by setting the value directly. Give shoulders stiffer springs than elbows (for example 230 vs 120) so forearms and hands lag slightly behind. This gives overlapping action and follow-through for free.
- **Easing:** never use linear motion between poses (smoothstep or ease-in-out).
- **Arcs:** hands and heads travel along curves, not straight lines.
- **Anticipation:** crouch before a jump, pull back before a throw, lean back slightly before stepping forward.
- **Secondary motion:** hair, skirts, ribbons, tails and ears follow head and hip velocity through their own looser springs (low damping, about 0.3), plus a light idle breeze.
- **Squash and stretch:** a slight crouch on landing; a slight stretch at the top of a jump.
- **Idle life:** blink every 3-5 seconds at irregular intervals, add a small breathing bob, and shift weight occasionally. A perfectly still character looks dead.
- **Asymmetry:** two characters should never move in perfect sync; offset phases by 0.1-0.3 s.

## 6. Walk cycles

Tie the walk phase to **distance travelled** (`phase = x / strideLength * 2*PI`), not time, so feet never slide. Fade the swing amplitude with speed so starting and stopping are smooth.

### 6.1 Side walk (profile or 3/4, turn about 0.8)
- Legs swing forward and back from one hip point, about ±0.36 rad thigh swing.
- Knee bends only on the passing or recovering leg, bending forward.
- Body bob comes from foot contact (lowest point at contact, highest at passing).
- Lean slightly into the walk (about 0.06-0.07 rad).
- **Arms swing opposite to legs** (left leg forward means left arm back), about ±0.3 rad, with elbows slightly bent forward (forearm swings a little further than the upper arm).
- The far arm is mostly hidden behind the body and shaded.
- Hair and skirt trail behind the direction of travel.

### 6.2 Front walk (toward the camera, turn = 0)
- Legs barely move sideways. Show stepping with **knee lift and foot height**, plus a slight scale-up of the stepping foot (it comes closer).
- Hips sway side to side a little (a weight shift over the standing leg), and shoulders counter-rotate slightly.
- Arms swing forward and back, which from the front shows as **forearm foreshortening and hands rising and falling**, not sideways movement. Keep the hands beside the hips.
- The whole character slowly scales up as it approaches (depth) and the ground shadow grows.

### 6.3 Back walk (away from the camera)
- Same as the front walk, with the back of the head (hair only, no face) and no visible chest details.
- Heels and soles show as feet lift; the character scales down slowly.
- Draw arms behind the torso only where they are actually hidden.

### 6.4 Walking backward (moonwalk-style retreat, still facing forward)
- The body faces one way while travel goes the other way: `turn` follows the facing direction, not the velocity.
- Shorter, careful steps; toes touch first, then heel.
- Lean slightly forward (the body counterbalances).
- Arms stay closer to the body, with smaller swings, often slightly forward for balance.
- Glance back over the shoulder occasionally (head turn without a body turn).

### 6.5 Character-specific walk flavour (style choices, not rules)
The mechanics stay the same; adjust parameters for personality:
- **Energetic or sporty:** longer stride, bigger arm swing, more bounce, forward lean.
- **Calm or graceful:** narrower step line (feet land near one line), smaller bob, relaxed arms, more hip sway.
- **Shy:** short steps, head down, arms close, hands clasped or holding a strap.
- **Confident:** upright chest, shoulders back, steady rhythm.
- **Tired or sad:** heavy feet, low arm swing, head down, slow.
- **Running:** both feet off the ground in part of the cycle, high knee lift, elbows bent about 90 degrees, strong forward lean, fists.
Common anime convention gives boys more energetic or confident walks and girls more graceful or shy ones, but pick from the character's personality, not gender alone.

## 7. Sitting

Sitting is a target pose reached with IK: the hips go to the seat point, the feet go to the floor (or dangle), and the hands go to the knees, lap or seat.

### 7.1 Transition to sitting (about 0.8-1.2 s)
1. Stand near the seat and glance at it.
2. Bend the knees and push the hips back; the torso leans forward to balance (about 0.3-0.4 rad).
3. The hips descend onto the seat with ease-out.
4. The torso straightens and settles with a small overshoot (springs do this).

Standing up is the reverse: lean forward first, then the legs push.

### 7.2 Side view sitting
- The thigh is horizontal along the seat and the knee bends about 90 degrees, so the shin is vertical to the floor.
- The back is upright or slightly reclined; hands rest on the thighs or the seat edge.

### 7.3 Front view sitting
- **Thighs point toward the camera:** use foreshortening (thigh scale about 0.35-0.5), so knees appear just below the hips.
- Shins are vertical, with feet on the floor slightly apart or together.
- The body is shorter overall; don't just shift the standing pose down.

### 7.4 Sitting styles (pick by character and situation)
- **Relaxed or open:** knees apart, hands on knees or leaning back on the arms, feet flat.
- **Neat or polite:** knees together, feet together or ankles crossed, hands folded on the lap.
- **Legs crossed:** one thigh over the other knee; the upper shin angles down and across.
- **Side-saddle (common for skirts in anime):** both knees together and angled to one side, feet tucked to the same side, one hand on the lap.
- **On the ground, cross-legged:** knees out wide, feet tucked under the opposite knees, hands on knees or ankles.
- **On the ground, knees hugged:** knees up to the chest, arms wrapped around the shins, chin on the knees (shy or thoughtful).
- **Seiza (kneeling):** knees together on the ground, sitting on the heels, hands on the thighs (formal, Japanese settings).
- **Swinging legs:** sitting high with feet dangling; shins swing out of phase (childlike or happy).
Anime often shows boys relaxed or open and girls neat or side-saddle, but this is a style choice. Clothing matters more: skirts suggest knees together or side-saddle; shorts and trousers allow any pose.

### 7.5 Contact rules while sitting
- The hips stay locked to the seat (no sliding) and the feet stay planted unless a gesture moves them.
- The skirt drapes over the knees and seat edge.
- Keep breathing and blinking going; seated characters still shift weight.

## 8. Gestures and body language

Every gesture has three parts: **anticipation**, then **action**, then **settle**. Hold the key pose for at least 0.3-0.5 s so it reads.

| Gesture | Arm and hand | Body and face |
|---|---|---|
| Wave hello | Upper arm raised about 130 degrees, forearm oscillates ±0.4 rad at about 1.7 Hz, open hand | Head tilts slightly, smile, open mouth |
| Point | Arm extends toward the target via IK, pointing hand; forearm foreshortened if pointing at the viewer | Head and eyes toward the target first, then the arm |
| Give an object | Arm reaches toward a midpoint between the characters, elbow down and out | Lean toward the receiver; slight blush or smile |
| Receive | Other hand reaches the same midpoint; swap the object's parent at contact | Small surprise, then a smile |
| Hold hands | Both hands IK to a shared point that bobs slightly | Characters turned slightly toward each other |
| Shy | Hands clasped in front of the hips, elbows out, shoulders slightly raised | Head down, eyes aside, blush, small sway |
| Embarrassed | Hand scratches the back of the head (elbow up and out) | Closed happy eyes, blush lines |
| Happy, overwhelmed | Hands cover the cheeks | Closed eyes, blush, body sway |
| Laugh | Hand on the stomach or covering the mouth | Body shakes (a fast small lean oscillation that decays), closed ^ ^ eyes |
| Think | Hand at the chin, other arm across the waist | Head tilt, eyes up |
| Arms crossed | Both forearms across the chest (forearms in front of the torso is fine; upper arms stay at the sides) | Weight on one leg |
| Hands on hips | Wrists at the hips, elbows far out | Chest up, confident |
| Sad | Arms hang limp, shoulders forward | Head down, slow blinks |
| Surprise | Hands up near the chest, open | Quick step back, wide eyes, small jump |
| Smell or show an object | Elbow down, forearm up toward the face; the object is near the nose, not the fist | Eyes closed, gentle smile |

Body language basics:
- **Open posture** (arms out, chest up) reads as confident or friendly; **closed posture** (arms in, shoulders forward) reads as shy, sad or defensive.
- **The head leads:** eyes move first, then the head, then the body. Never move all three at once.
- **Lean toward** someone you like or are listening to; **lean away** from something unpleasant.
- **Look at things:** characters' eyes and heads track the important object (a flower, a butterfly, the other character).
- Interacting characters should face, or partly face, each other.

## 9. Defect checklist (check every frame range before delivering)

Render still frames at every key moment and inspect them. Common defects and their fixes:

1. **Elbow or knee looks like two bent lines** → fillet that grows with the bend, tapered ribbon, outline then fill.
2. **Hand looks like a ring** → fist or mitten shape aligned with the forearm, drawn in the same outline pass.
3. **Shoulder seam or ring** → sleeve outline under the torso, sleeve fill over it, fading sleeve edge.
4. **Seam reappears in some frames** → the same method for all arms; turn threshold not borderline (> 0.55).
5. **Shoulder bump above the chest** → shoulder pivot lower and inside the shirt line; slim sleeve root.
6. **Arms hang over the chest, hands in front of the hips** → shoulder pivot at the chest edge; rest arm angle outward (about 0.15-0.2 rad).
7. **Elbow folds across the chest** → `ikOut` (elbow outside and lower), plus forearm foreshortening for forward reaches.
8. **Walking sideways with the chest facing the camera** → `turn` toward travel; narrow hips, shoulders and torso; far limbs behind and shaded.
9. **Legs scissor or cross when walking** → hip width shrinks to near 0 when side-on; knees bend forward.
10. **Feet slide** → walk phase from distance, not time.
11. **Floating or sinking** → hips placed from the lowest foot.
12. **Robotic timing** → springs, easing, overlapping action, offsets between characters.
13. **Stiff idle** → blink, breathe, sway, hair and skirt in the breeze.
14. **Object held in the wrong place** (fist covers the face) → move the target so the object, not the hand, reaches the face.
15. **Crease marks look like scratches** → show only on strong bends, faint.

### Known gaps to watch for (improve when they show up)
- **Neck:** add a slight neck bend and stretch when the head tilts strongly; keep the collar covering the neck base.
- **Hips and waist:** add a small pelvis tilt during walks and sitting; the torso should counter-rotate.
- **Fingers:** add more hand poses (peace sign, thumbs up, open palm, holding a cup) as separate shapes.
- **Clothing collision:** sleeves and skirts must not pass through the body; clamp the skirt swing and let it rest on the thighs when sitting.
- **Hair and face overlap:** front hair over the face, back hair behind the body; hair locks over the shoulder go in front of the arm.
- **Ground contact:** add a toe bend during push-off, a heel strike, and a contact shadow under each foot.
- **Turn transitions:** for a full 180-degree turn, swap to a back-view head midway and hide the face features.
- **Overlaps between characters:** sort drawing by depth (y position) so the closer character is drawn last.

## 10. Workflow

1. Define the rig (lengths, pivots, rest angles) and palettes per character.
2. Write the story as timed beats; for each beat, set target poses (angles, IK targets, turn, face, expression).
3. Let springs move the actual pose toward the targets; solve the skeleton; place the hips from the feet.
4. Draw in depth order: far limbs, legs, torso (with sleeve blend), near limbs, head, hair, held objects.
5. Add secondary motion, effects and the environment.
6. Provide debug toggles: show bones, and a robotic-limbs comparison.
7. Render still frames at each beat (headless browser), check them against section 9, fix, and re-check.

## 11. Prompt snippet for an animation request

> Animate the characters with a hidden skeleton and visible skin. Draw each limb as one tapered ribbon with filleted joints (outline pass, then fill). Blend sleeves into the torso (sleeve outline under the torso, sleeve fill over it, fading edge line), with the shoulder pivot at the chest edge just below the shirt line. Use IK that always picks the elbow outside the body and lower, knees bending forward, and foreshortening for limbs pointing at the camera. Turn characters toward their walking direction (narrow hips, shoulders and torso; far limbs behind and shaded), and turn them slightly toward each other when interacting. Arms hang beside the hips at rest. Drive all joints with springs (shoulders stiffer than elbows), ease everything, tie walk cycles to distance, and add hair and clothing secondary motion, blinking and breathing. Check still frames at every story beat against the defect checklist before delivering.