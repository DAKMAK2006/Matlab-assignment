# Pick and Place — Recruitment Assignment

You've seen robotic arms on factory floors — grabbing, moving, and placing objects with mechanical precision. Now build one.

You're given a **Simscape Multibody model** of a pick-and-place system: a gripper arm riding on a post, and two conveyor belts bringing objects in and carrying them out. Every block you need is already in the file. Nothing is connected. When you run it, MATLAB will flood you with errors.

Your job is to understand what each block does, wire the system correctly, configure the transforms so the belts face the right direction, and get the whole thing running in 3D.

The `anim.gif` in this repo shows what the finished simulation looks like. That's your target.

---

## Before You Touch Anything

### 1. Install the required toolboxes

You need these four — no others, no shortcuts:

- MATLAB
- Simulink
- Simscape
- Simscape Multibody

If you're missing any of them, go to **Home → Add-Ons → Get Add-Ons** in MATLAB and install them. Without Simscape Multibody specifically, none of the 3D blocks will even load.

### 2. Initialize the library

Before opening the model, run this in the MATLAB command window:

```matlab
run('Pick_And_Place/Scripts_Data/startup_Contact_Forces_Library.m')
```

This sets up all the paths and registers the custom contact force library that the model depends on. If you skip this step, the model will open with missing block errors that have nothing to do with what you're supposed to fix.

### 3. Open the model

```
Pick_And_Place/Examples/3D/Gripper_2Belts/Gripper_2Belts.slx
```

Run it. Read every error. Do not start fixing things yet.

---

## What You're Looking At

The model has five major parts. Understand each one before touching any wires.

**Gripper** — A post with a revolute joint carries a gripper head. The gripper has two fingers (A and B) driven by prismatic joints. Finger A moves in one direction; Finger B moves in the opposite direction (a gain of −1 handles this). The gripper closes to grip a box and opens to release it.

**Box** — A rigid body with a 6-DOF joint that lets it sit on and move with the belt. Contact forces between the box and belt surface make it move — this is handled by the `Box to Belt Force` blocks.

**Belt Out** — The outgoing conveyor. Has 5 rollers driven by revolute joints. A speed signal controls roller rotation through an integrator.

**Belt In** — The incoming conveyor. Same structure as Belt Out. Already connected correctly — use it as your reference when fixing Belt Out.

**Commands** — The signal source. Sends position and velocity commands to the post, gripper, and belts on a timed schedule.

---

## What Is Actually Broken

The top-level model is missing all of its connections and several blocks. Specifically:

- There is no **World Frame** block — the model has no ground reference
- There are no **Rigid Transform** blocks positioning the belts in space
- There are no **Goto/From** blocks routing the belt frame signals
- None of the top-level blocks are connected to each other

The subsystem internals (gripper, belt roller chains) are already correct. The puzzle is the top level.

---

## How to Learn What You Need

None of this requires prior Simscape experience. Everything below is learnable from scratch in a focused sitting. Work through these in order — don't skip ahead.

### Simscape Multibody basics

Start here. This 20-minute official tutorial teaches you what frames, rigid transforms, and joints actually mean in Simscape:

**→ [Getting Started with Simscape Multibody](https://www.mathworks.com/help/sm/getting-started-with-simscape-multibody.html)**

Pay attention to how the **World Frame** anchors the simulation and why every mechanism needs one. Pay attention to how **Rigid Transform** blocks define position and orientation offsets between frames. These two concepts are 80% of what you need.

### Rigid Transform — rotations and translations

The two belts need to be positioned and rotated into the correct orientation relative to the world frame. A Rigid Transform block lets you specify:
- A **rotation** (axis + angle)
- A **translation** (x, y, z offset)

Read this page to understand how Standard Axis rotations work:

**→ [Rigid Transform block documentation](https://www.mathworks.com/help/sm/ref/rigidtransform.html)**

The Belt In transform needs a **−90° rotation about +Z**. The Belt Out transform needs a **180° rotation about +Z**. Before you type those numbers in, draw the coordinate frames on paper and convince yourself why those specific values align the belts correctly. If you can't explain it, you don't understand it yet.

### Goto and From blocks

These are standard Simulink blocks that let you route a signal from one part of a model to another without drawing a wire across the canvas. A **Goto** block tags a signal with a name; a **From** block picks it up anywhere in the model using the same tag.

**→ [Goto block documentation](https://www.mathworks.com/help/simulink/slref/goto.html)**

In this model, the belt frame outputs need to reach the `Box to Belt Force` blocks via Goto/From pairs. Look at how the existing `[In]` and `[Out]` tags are already set up in the model — your job is to wire the Goto blocks to the correct sources.

### Belt Out internals — the one subsystem you need to understand

Open the Belt Out subsystem (double-click it). Compare it carefully with Belt In. One connection is incomplete. You'll need to understand the roller chain — each Roller body connects to a Revolute joint, which connects to a Transform block, which positions the roller along the belt frame. The speed signal at the bottom drives all rollers through a gain and integrator.

### Contact Forces

The box doesn't have a motor driving it — it gets pushed by friction forces between its bottom face and the belt surface. The `Box to Belt Force` blocks compute these contact forces. You don't need to modify them, but you should understand what inputs they expect:

- `In` — the belt frame (world position of the belt surface)
- `bus On` — whether the belt is active
- `Out` — the frame output back to the box's 6-DOF joint

**→ [Contact Forces Library README](Pick_And_Place/CFL_Core/README_contact_forces_core.txt)**

Read this file. It explains what the library does and what each port expects.

---

## Step-by-Step: What to Add and Where

This is not a solution. It tells you what blocks to add — not how to configure them or where to connect them. Figure that part out yourself.

**Step 1 — Add a World Frame block**

Search for `world frame` in the Simulink Library Browser. Add one to the top-level model. This is the ground reference that everything else connects to.

**Step 2 — Add Transform Belt Out**

Search for `rigid transform`. Add one, name it `Transform Belt Out`. Configure its rotation and translation to position the outgoing belt correctly. The translation offset is `belt_out_offset` (already defined in the workspace by the startup script).

**Step 3 — Add Transform Belt In**

Same as above, name it `Transform Belt In`. Translation offset is `belt_in_offset`.

**Step 4 — Add Goto and Goto1**

Search for `goto`. Add two. Name one `Goto` with tag `Out`, name the other `Goto1` with tag `In`. These broadcast the belt frame signals so the `Box to Belt Force` blocks can receive them.

**Step 5 — Wire the top level**

Connect everything. The signal flow is:

```
World Frame → Transform Belt Out → Belt Out (Ctr port)
                                 → Box to Belt Out Force (In port)
                                 → Goto [Out]

World Frame → Transform Belt In  → Belt In (Ctr port)
                                 → Box to Belt In Force (In port)
                                 → Goto1 [In]
```

The Commands block drives the belt speed signals, gripper commands, and post position. Trace each output port and connect it to the right destination.

**Step 6 — Fix Belt Out internals**

Open the Belt Out subsystem. Find the missing connection. Use Belt In as your reference — the structure is identical.

---

## Checking Your Work

Run the plot script after a successful simulation:

```matlab
run('Pick_And_Place/Examples/3D/Gripper_2Belts/Gripper_2Belts_plot1boxposition.m')
```

This plots the box's position over time. If the pick-and-place worked, you'll see the box move along the incoming belt, pause (gripper closing), lift, move, and descend onto the outgoing belt. A flat line means the box never moved — the belts aren't connected. An exploding position means a frame is pointing the wrong way.

Compare your 3D animation against `anim.gif`.

---

## Submission

Push to a **public GitHub repository** containing:

```
/
├── Gripper_2Belts_solution.slx       ← your completed model
├── screen_recording.mp4              ← full screen recording of the 3D simulation running
└── README.md                         ← explain what was broken, what you changed, and why
                                         the transform values are what they are
```

The model must run from a single click of the Run button after `startup_Contact_Forces_Library.m` has been executed. Document any additional steps in your README.

---

## Resources

| What | Link |
|---|---|
| Simscape Multibody getting started | https://www.mathworks.com/help/sm/getting-started-with-simscape-multibody.html |
| Rigid Transform block | https://www.mathworks.com/help/sm/ref/rigidtransform.html |
| Goto block | https://www.mathworks.com/help/simulink/slref/goto.html |
| Conveyor belt example | https://www.mathworks.com/help/sm/ug/conveyor-belt.html |
| Prismatic joint | https://www.mathworks.com/help/sm/ref/prismaticjoint.html |
| Revolute joint | https://www.mathworks.com/help/sm/ref/revolutejoint.html |
| Contact Forces Library README | `Pick_And_Place/CFL_Core/README_contact_forces_core.txt` |
