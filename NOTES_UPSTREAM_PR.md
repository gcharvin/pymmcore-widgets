# Notes For Upstream PR

## Context

This fix was validated against a real TiEclipse microscope setup where:

- stage position changes were triggered from the positions table
- `Move Stage to Selected Point` was enabled
- the microscope had a broken `TIFilterBlock1` filter block turret
- autofocus / PFS may or may not be active depending on the current test

## Branch

- local branch: `codex/position-wait-devices`

## Problem

Selecting a saved position caused:

- stage movement to the requested XY/Z location
- a noticeable lag after the move
- `RuntimeError: Wait for device "TIFilterBlock1" timed out after 5000ms`

This was confusing because the operation was conceptually only a stage move.

## Root cause

`CoreConnectedPositionTable._on_selection_change()` used broad:

- `waitForSystem()`

after moving to the selected position, and also in the autofocus path.

On a microscope where another device is broken or permanently busy, `waitForSystem()` blocks on that unrelated device as well. In the tested case, the broken device was:

- `TIFilterBlock1`

So a stage move looked slow or faulty even though the stage itself had already reached the target.

## Fix

Replace global system waits with targeted waits on the devices actually involved in the action:

- wait for XY stage only when XY movement is requested
- wait for focus stage only when Z movement is requested
- wait for autofocus devices only when autofocus logic is actually used
- also wait for the autofocus offset device when one exists, such as Nikon TI `TIPFSOffset`

## Autofocus-specific handling

The patch explicitly keeps autofocus-aware behavior:

- if autofocus is not in use, no autofocus wait is performed
- if autofocus offset is restored or autofocus is run, waits are scoped to autofocus-related devices
- focus/Z can still be waited on during autofocus operations when appropriate

## Validation

Observed on the tested setup:

- clicking a saved position now moves the stage without triggering the unrelated `TIFilterBlock1` timeout
- lag is removed or greatly reduced
- autofocus-related positioning behavior still works when autofocus is actually engaged

## Why This Matters Upstream

This is a robustness fix, not just a workaround for one broken microscope.

Any hardware stack with:

- one non-stage device in a bad state
- or one slow / timing-sensitive device elsewhere in the system

can suffer from the same issue whenever a stage-position action uses `waitForSystem()` instead of device-scoped waits.

## Suggested PR framing

The upstream PR for `pymmcore-widgets` can likely be framed as:

1. avoid broad `waitForSystem()` in saved-position stage moves
2. wait only on the devices actually touched by the operation
3. preserve autofocus-aware behavior for position moves with AF/PFS

## Things To Mention Explicitly In PR

- the issue was discovered because `TIFilterBlock1` was physically broken on the microscope
- the fix is still generally useful because it narrows synchronization to the relevant devices
- stage motion itself was not the failing part; the timeout came from unrelated global waiting
