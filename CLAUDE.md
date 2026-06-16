# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project type

TwinCAT 3 / Beckhoff XPlanar sample. Engineering tool version `3.1.4026.x` (see `TcVersion` in `XPlanarCoord.tsproj`). Distributed by Beckhoff Automation LLC as illustrative sample code — see the disclaimer in `README.md`.

- Solution: `XPlanarCoord.sln` (opens in TwinCAT XAE / Visual Studio).
- System project: `XPlanarCoord.tsproj` — defines real‑time tasks, CPU isolation, NC/MC/PLC subprojects, and TcCOM module wiring.
- PLC project: `PLC/PLC.plcproj` — Structured Text, with placeholder references to `Tc2_MC2`, `Tc2_Standard`, `Tc2_System`, `Tc3_Module`, and `XPlanarApplication` (the Beckhoff XPlanar library that supplies `FB_Component_XPlanar`, `FB_XPlanarMover`, `MoverVector`, `ip.Movers[...]`, etc.).
- Target NetId in `XPlanarCoord.tsproj`: `192.168.137.77.1.1`. Change this for a different runtime.

## Build / run

There is no command-line build script. The workflow is:

1. Open `XPlanarCoord.sln` in TwinCAT XAE (or TcXaeShell).
2. Activate configuration and download to the target listed in `TargetNetId`.
3. The `PlcTask` runs `MAIN` every 10 ms (`PLC/PlcTask.TcTTO`). XPlanar runtime tasks run on isolated cores at 2.5 ms (CPUs 3–5, see `<Tasks>` in `XPlanarCoord.tsproj`).

A CLI build is possible via `TcXaeShell.exe XPlanarCoord.sln /Build`, but no script is checked in — confirm with the user before invoking.

## Editing `.TcPOU` / `.TcDUT` files

POU/DUT files are XML wrappers around the actual ST source. Two conventions matter:

- The ST code lives inside `<![CDATA[ ... ]]>` blocks under `<Declaration>` and `<Implementation><ST>`. Edit the contents of the CDATA, not the surrounding XML.
- Every POU has a GUID `Id="{...}"` in the opening tag. Do not invent new GUIDs when editing existing files; only generate one when creating a new POU.

When adding a new POU/DUT, also add a `<Compile Include="...">` entry to `PLC/PLC.plcproj` — TwinCAT does not pick up files by directory scan.

## Big-picture architecture

The application is a one‑mover demo coupling an NC C‑axis (rotary tool) to an XPlanar mover, then feeding a transformed world‑frame pose back to the XPlanar via `RunExternalSpAbsoluteMode`. The interesting code is the rotation math, not the state machine.

### Coordinate chain (`PLC/Transformation/MoverToWorldTransform.TcPOU`)

Forward transform: `World (Tile) ← Mover Pose ← Tool Offset ← Tool Tip`.

- Rotation convention: **intrinsic XYZ Tait‑Bryan** — Rx applied first, then Ry around the new Y, then Rz around the new Z. Combined matrix `R = Rz · Ry · Rx`. Both the mover pose (`MoverRx/Ry/Rz`) and the tool offset (`ToolOffsetRx/Ry/Rz`) use this convention. Any new transform you add must follow it, or document the deviation.
- Output angles `WorldRx/Ry/Rz` are **unwrapped and absolute** — they accumulate past ±360°. There is a `Reset` rising-edge input to clear the unwrap history (`UnwrapOffR*`, `FirstCalc`).
- The asin extraction of `Ry` clamps `R_W[3,1]` to `[-1, 1]` and sets `Error := TRUE` near the ±90° pitch singularity (gimbal lock). At that point only Rx ± Rz is meaningful.
- The block runs every scan cycle with no enable handshake — wire inputs each cycle and read outputs.

### `MAIN` state machine (`PLC/MAIN.TcPOU`)

Drives one `FB_XPlanarMover` through `E_Steps` (`init → enable → enabled → moveCenter → idle → enableExtSp → runExtSp → error`, defined in `PLC/E_Steps.TcDUT`). The interesting transition is `runExtSp`:

- Reads `Mover.ActualPosition` (xyz + a/b/c) into `MoverToWorldTransform`.
- Feeds `CAxis.NcToPlc.ActPos` in as `ToolOffsetRz` — i.e. the NC C‑axis angle is interpreted as a tool rotation about Z.
- Pushes the transformed world pose plus velocity/acceleration (currently quick‑and‑dirty: vel/acc duplicated across a/b/c) into `ip.Movers[1].MoveCommand.RunExternalSpAbsoluteMode(...)` each cycle. The comments in MAIN explicitly note that velocity needs proper differentiation for real applications.
- Note the sign flip `Position.b := -MoverToWorldTransform.WorldRy;` — the XPlanar API expects opposite Y handedness from the transform output.

### `ATAN2` (`PLC/ATAN2.TcPOU`)

`Tc2_Standard` ships `ATAN` but not a quadrant-aware `atan2`. This file provides one returning a result in `(-π, π]`. The Euler extraction in `MoverToWorldTransform` depends on it. If you remove or rename it, the transform breaks.

## Repository layout notes

- `PLC/Transformation/` is where transform POUs are intended to live (per `PLC.plcproj`). Some files currently sit under `PLC/generated/chat_0/` per `git status` — `XPlanarForwardTransform.TcPOU` and `XPlanarForwardTransform_.TcPOU` look like earlier drafts; `MoverToWorldTransform.TcPOU` in `Transformation/` is the live one referenced by MAIN.
- `_Config/` contains the activated TwinCAT system configuration (NC axes, MC project, TcCOM XPlanar modules, PLC instance). These are large XML/XTI snapshots; edit through XAE, not by hand, unless making a targeted text fix.
- `_Boot/`, `_CompileInfo/`, `_Libraries/` are gitignored build artifacts.

## Conventions to preserve when extending

- Keep angles in degrees at the POU interface; convert to radians internally with the `DegToRad` constant.
- When adding new Euler‑extraction code, mirror the clamp + asin + atan2 pattern in `MoverToWorldTransform` rather than calling `ASIN` directly on a possibly‑denormal value.
- Continuous outputs that feed motion commands (positions, unwrapped angles) should not have rollover discontinuities — use the same `PrevRaw* / UnwrapOff*` pattern.
- `E_Steps` has attributes `qualified_only`, `strict`, `to_string` — reference enum members by `E_Steps.<name>`, not by integer.
