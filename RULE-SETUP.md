# Rule calibration checklist — MyGeotab

Work through this in order. Each step says what to change, where, and how to
tell it worked. Nothing here is urgent enough to rush; the ordering exists so
you never have a gap where neither the old nor the new rule is recording.

**Why this is needed:** the legacy accelerometer rules are miscalibrated in
both directions — braking so tight that 68% of trucks never trigger it in a
week, acceleration and cornering so loose that they mostly count potholes. They
also flag a Chevy Colorado driver for cornering **3.2× as often** as a GMC
Sierra driver at identical settings, which means any driver score built on them
ranks partly by which truck someone was handed.

All rules live under **Groups & Rules → Rules**, section **SAFETY**.

---

## Phase 0 — Snapshot first

- [ ] Run the baseline snapshot. This is your rollback record and the
      "before" measurement:

```bash
cd ~/Documents/geotab-scripts && python3 rule_snapshot.py before
```

It writes `rule_snapshot_before.json` containing every rule's full condition
tree, so any change here can be reversed by hand.

> **Note:** a rule switched **Off** disappears from the API's rule list
> entirely, and its exception events stop being retrievable. The raw telemetry
> survives — that's what Reprocess Data rebuilds from — but don't rely on the
> events being there while a rule is off.

---

## Phase 1 — Reseat ten devices *(do this first, it's independent)*

Ten vehicles produce 95% of all physically impossible readings — accelerations
no vehicle can achieve on tyres. That's the signature of a GO device that isn't
seated firmly in the OBD port, so it rattles and reports the shock as driver
behaviour. It corrupts those drivers' scores and nothing else will fix it.

- [ ] DET-148 — Ford Maverick *(59 bad samples, peak 5.48 g — worst by far)*
- [ ] #8 Rental Ram 1500 *(13, peak 3.77 g)*
- [ ] #3 Rental Ram 1500 *(11, peak 2.20 g)*
- [ ] AMF-75 — Ford Maverick *(9, peak 2.81 g)*
- [ ] #6 Rental Ram 1500 *(8, peak 3.28 g)*
- [ ] #5 Rental Ram 1500 *(7 and 4 across two devices, peak 2.03 g)*
- [ ] IND-125 — Chevrolet Colorado *(4, peak 2.29 g)*
- [ ] CLE-103 — Ford Maverick *(3, peak 1.90 g)*
- [ ] #1 Rental Frontier *(3, peak 1.94 g)*
- [ ] #10 Rental Sierra *(2, peak 2.17 g)*

**Verify:** re-run `rule_snapshot.py` a week later — the impossible-g list at
the bottom should be empty or close to it.

---

## Phase 2 — Turn the GPS rules back on

These read GPS-derived acceleration rather than the accelerometer, and pick
their threshold per vehicle from the VIN (an 8-class lookup). That is what
removes the model bias — you cannot hand-tune one number that is correct for a
Maverick, a Colorado and a Ram 1500 at the same time.

- [ ] **Harsh Braking (New)** → On
- [ ] **Harsh Acceleration (New)** → On
- [ ] **Harsh Cornering (New)** → On

Then, with those three selected:

- [ ] Click **Reprocess Data** (top toolbar) to rebuild history from the
      retained telemetry.

**Verify — this is the step that matters most.** Reprocess is the one thing in
this checklist I could not confirm in advance. After it finishes:

```bash
cd ~/Documents/geotab-scripts && python3 rule_snapshot.py after
```

- Expect **Harsh Braking (New)** around **1.8 per 1,000 mi**,
  **Harsh Acceleration (New)** around **1.2**, **Harsh Cornering (New)**
  around **34**.
- If the counts only cover the days *since* you enabled them, reprocess did
  **not** backfill. Say so before building any month-over-month comparison —
  the scoring window will have a seam in it.

---

## Phase 3 — Turn off the legacy duplicates

Only after Phase 2 shows real event counts. Running both briefly is harmless;
a gap is not.

- [ ] **Hard Acceleration** (0.36 g) → Off — 31/1,000 mi, mostly road artifact,
      and 0.36 g is roughly what these trucks do at full throttle anyway
- [ ] **Harsh Cornering** (0.40 g) → Off — 44/1,000 mi, peaked at 5.48 g lateral

**Leave Harsh Braking (legacy) alone for now.** At −0.54 g it fires 1.26/1,000
mi and is close to inert, so it costs nothing to leave on, and it's a useful
cross-check that the new braking rule is behaving. Revisit once Phase 2 is
confirmed.

> Earlier I suggested retuning it to −0.45 g. **Ignore that** — it was based on
> cube-van dynamics before I knew the fleet is Mavericks and Colorados. If you
> do want a manual knob, set the vehicle class to **Passenger Car**, not
> Truck/Cube Van, which is what all three are wrongly set to today.

---

## Phase 4 — Speeding

- [ ] **Create a rule at 80 mph.** Your written policy (Megan Warner, 2026-07-31)
      makes 80+ a final warning, but nothing enforces it — Excessive Speed only
      fires at 90. Duplicate the 90 mph rule and change the threshold.
- [ ] **Consider: Speeding (New) → On.** It fires at 20% over for 5 seconds
      rather than a flat 10 mph. On a 25 mph residential street that's 30 mph;
      your flat rule needs 35. For a fleet on neighbourhood streets all day
      that's a real gap. **Watch it for two weeks before scoring on it** so it
      doesn't double-count with the posted rule.
- [ ] Leave **Speeding** (posted, 10 mph over) exactly as it is. It's the basis
      of the existing dashboards.

---

## Phase 5 — Leave on, but keep out of the score

No changes needed. Recording these is useful; scoring them is not.

| Rule | Why not scored |
|---|---|
| Driver Drinking or Eating | 26/1,000 mi — would swamp genuine distraction signal |
| Driver Yawning / Eye Rubbing | Scoring fatigue gives people a reason to hide it. Scheduling problem, not coaching |
| Idling / Idling Within Zone | Cost, not safety |
| After Hours Usage | Policy and utilisation |
| All collision rules | A collision should flag the driver outright, not cost points |

Also worth a look independently of the scorecard: **Unauthorized Device Removal
(302 events / 30 days)** and **Cabin Camera Obstruction (359)**. Someone is
unplugging devices and covering cameras, which silently corrupts everyone
else's numbers. And **Driver Fatigue fired 0 times while Driver Yawning fired
961** — worth checking that rule is actually configured.

---

## Phase 6 — Settle, then recalibrate

- [ ] Let it run **at least two weeks** after Phase 3.
- [ ] Re-run the scorecard calibration. Every event rate changes, so the
      deduction factors have to be re-derived from the new distribution.
- [ ] Only then put a ranked list in front of a manager.

Showing a leaderboard before this point would rank people partly by their truck
and partly by which devices were loose.

**On the rentals:** they're 43% of current miles and expected to leave. Deriving
the scorecard's factors live from the current window lets it self-correct as
the fleet turns over — at the cost of scores drifting between windows. Freeze
them once the fleet stabilises, and version the change the way `violin.html`
handles tier drift.
