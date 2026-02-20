# Frequently Asked Questions

This page answers common questions about pooltool's capabilities, physics models, and
performance characteristics.

---

## Can pooltool calculate ball trajectories from a 2D ball layout?

**Yes.** You provide each ball's (x, y) position on the table and pooltool computes the
full 3D trajectory—position, velocity, and angular velocity over time—for every ball
in the system.

```python
import pooltool as pt

# Build a system with a custom ball layout
cue_ball = pt.Ball.create("cue", xy=(0.5, 0.5))
one_ball = pt.Ball.create("1", xy=(0.5, 1.0))
two_ball = pt.Ball.create("2", xy=(0.4, 1.1))

table = pt.Table.default()
cue   = pt.Cue(cue_ball_id="cue")
shot  = pt.System(table=table, balls=[cue_ball, one_ball, two_ball], cue=cue)

# Aim and shoot
shot.cue.set_state(V0=3.0, phi=pt.aim.at_ball(shot, "1"))

# Simulate – every collision, roll, spin and pocket event is computed
pt.simulate(shot, inplace=True)
```

After `simulate`, `shot.events` contains the full ordered sequence of events (collisions,
transitions, pockets) and each ball's `history` holds its kinematic trajectory at each
event time.  Use {py:func}`pooltool.continuize` to densely sample the trajectory at a
fixed time step if you need evenly-spaced frames.

---

## Can pooltool model carom (karom) shots, banks, and kicks?

**Yes** – all three shot types are handled naturally by the physics engine:

| Shot type | What happens | Supported? |
|-----------|-------------|------------|
| **Carom / karom** | Cue ball contacts two or more object balls | ✅ Multi-ball collisions are resolved in event order |
| **Bank shot** | Ball rebounds off one or more cushion rails | ✅ Cushion collisions are a first-class event type |
| **Kick shot** | Cue ball hits a rail before contacting the object ball | ✅ Same cushion-collision mechanism |
| **Three-cushion billiards** | Cue ball must contact three cushions | ✅ `GameType.THREECUSHION` is a supported game type with its own ruleset |

The simulator raises a `BALL_LINEAR_CUSHION` or `BALL_CIRCULAR_CUSHION` event every
time a ball contacts a straight or curved cushion segment, and a `BALL_BALL` event for
every ball-ball collision (see {py:class}`~pooltool.events.EventType`).  You can query
`shot.events` to count or inspect each type.

---

## Does pooltool model spin, including spin interactions with the rails?

**Yes.** Every ball carries a full 3D angular velocity vector $\boldsymbol{\omega}$.
The simulator tracks five distinct motion states:

| State | Description |
|-------|-------------|
| `stationary` | Ball at rest |
| `sliding` | Ball translating with spin that differs from rolling condition |
| `rolling` | Ball in pure rolling contact with the cloth |
| `spinning` | Ball spinning in place (only top/back-spin component remains) |
| `pocketed` | Ball in a pocket |

When a spinning ball hits a cushion, the angular velocity is incorporated into the
post-collision velocities. Several cushion models explicitly account for spin:

- **Han (2005)** – widely used cushion model that models the effect of ball spin on
  rebound angle and speed.
- **Mathavan et al. (2010)** – differential-equation-based cushion model validated
  against high-speed camera measurements.
- **Impulse Frictional Inelastic** – impulse-based model with tangential friction at
  the cushion contact point.
- **Stronge Compliant** – accounts for tangential compliance, allowing the slip
  direction at the contact point to reverse during the collision.

---

## What physics does pooltool model?

Pooltool is built on **Newtonian rigid-body mechanics** and uses an
**event-based simulation algorithm** (no fixed time steps).

### Ball motion between events

Between collisions the equations of motion are solved analytically for each
motion state:

- **Sliding** – cloth sliding friction decelerates the ball and couples into spin.
- **Rolling** – rolling friction decelerates the ball; top-spin and back-spin
  already match the rolling condition.
- **Spinning** – only the perpendicular (z-axis) angular velocity component
  remains; spinning friction brings it to zero.

Physical parameters are configurable per ball via {py:class}`~pooltool.objects.BallParams`:

| Parameter | Symbol | Description |
|-----------|--------|-------------|
| Mass | $m$ | Ball mass |
| Radius | $R$ | Ball radius |
| Sliding friction | $\mu_s$ | Cloth sliding coefficient of friction |
| Rolling friction | $\mu_r$ | Cloth rolling coefficient of friction |
| Spinning friction | $\mu_{sp}$ | Cloth spinning coefficient of friction |
| Ball-ball friction | $\mu_b$ | Ball-ball sliding coefficient of friction |
| Ball-ball restitution | $e_b$ | Coefficient of restitution for ball-ball collisions |
| Cushion restitution | $e_c$ | Coefficient of restitution for ball-cushion collisions |
| Cushion friction | $f_c$ | Coefficient of friction for ball-cushion collisions |
| Gravity | $g$ | Gravitational acceleration |

### Collision models

Pooltool ships with multiple, interchangeable physics models for each interaction type.
You can mix and match models via the {py:class}`~pooltool.physics.PhysicsEngine`.

**Ball-ball collision models** (`BallBallModel`):

| Model | Description |
|-------|-------------|
| `FRICTIONLESS_ELASTIC` | Simplest model – frictionless, elastic, equal-mass |
| `FRICTIONAL_INELASTIC` | Adds ball-ball friction and coefficient of restitution |
| `FRICTIONAL_MATHAVAN` | Full Mathavan et al. (2014) model accounting for spin effects on exit angles |

**Ball-cushion collision models** (`BallLCushionModel` / `BallCCushionModel`):

| Model | Description |
|-------|-------------|
| `HAN_2005` | Classic Han (2005) model |
| `MATHAVAN_2010` | Mathavan et al. (2010) – numerically integrated differential equations |
| `IMPULSE_FRICTIONAL_INELASTIC` | Impulse-based, includes tangential friction |
| `STRONGE_COMPLIANT` | Impulse-based with tangential compliance; no numerical integration |
| `UNREALISTIC` | Perfect specular reflection (spin unchanged) |

**Stick-ball model** (`StickBallModel`):

| Model | Description |
|-------|-------------|
| `INSTANTANEOUS_POINT` | Point-like instantaneous impact; computes post-impact velocity + angular velocity including squirt (deflection) |

### Supported game types

`GameType.EIGHTBALL`, `GameType.NINEBALL`, `GameType.THREECUSHION`,
`GameType.SNOOKER`, `GameType.SUMTOTHREE`.

Each game type comes with a configurable ruleset ({py:func}`~pooltool.get_ruleset`) and
a default ball rack ({py:func}`~pooltool.get_rack`).

---

## How fast is pooltool?

Pooltool uses two complementary strategies to achieve high simulation throughput:

1. **Event-based algorithm** – instead of marching forward in small fixed time steps,
   the simulator computes the exact time of every future event (collision, transition to
   rolling, etc.) analytically, then jumps directly to that event.  This avoids millions
   of unnecessary integration steps.

2. **JIT compilation with Numba** – all hot paths in the simulation kernel (event-time
   solvers, ball-motion equations, collision resolvers) are compiled to native machine
   code at runtime using [Numba](https://numba.readthedocs.io/).

In practice, a typical pool shot (e.g. a nine-ball break) involving 10 balls and
dozens of collisions simulates in **milliseconds** on a modern laptop.  Because the
cost scales with the number of events (not with simulated time), long-duration but
low-activity scenarios are especially cheap.

---

## How many lines of code does the library contain?

The `pooltool` Python package contains approximately **24,000 lines of code**
across ~150 source files.  This includes the physics engine, event-based simulator,
3D visualisation interface, game rulesets, AI utilities, and serialisation layer.

```{eval-rst}
.. note::

   These numbers reflect the state of the codebase at the time this page was written
   and will change as the project evolves.
```
