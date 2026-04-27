# A9 Dataset Control Tower

This repository is the coordination point for the A9 dataset and deploy-compatible PointPillar work.

Central issue: https://github.com/jakebeep00/A9-Dataset-Control-Tower-/issues/1

Primary guide: [docs/A9_CONTROL_TOWER_GUIDE.md](docs/A9_CONTROL_TOWER_GUIDE.md)

## Current Status

Status: `YELLOW`

The work is approved only for deployment contract alignment and smoke validation. Full training is blocked until the smoke gates in the guide pass.

## Participants

| Thread ID | Role | Workspace / Server |
|---|---|---|
| `019dcca8-389a-7033-87f4-204909603c9e` | Command Center | Current control thread |
| `019d759d-9bd8-70d1-84b0-266e4d89ef90` | Build Station | `Model_Training_Server` |
| `019d75bd-2f2c-7cd2-9fa1-46a4f239d99c` | Model QA / Coordinator | `Model_Training_Server` |
| `019d75fd-6ada-7b31-9eed-5673b2826f9e` | Lidar Runtime Reviewer | `Lidar_Detection_Server` |

## How Each Thread Should Start

Each thread should read the guide first, then act only within its assigned role.

Use this instruction:

```text
Read https://github.com/jakebeep00/A9-Dataset-Control-Tower-/blob/main/docs/A9_CONTROL_TOWER_GUIDE.md and issue #1.
Act only as the role assigned to your thread ID.
Report using the required format in the guide.
Do not start full training unless the Coordinator marks all smoke gates as passed.
```
