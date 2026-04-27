# A9 Control Tower Guide

Date: 2026-04-27
Central issue: https://github.com/jakebeep00/A9-Dataset-Control-Tower-/issues/1

This guide is the single source of truth for coordinating the three Codex threads working on the A9 dataset and deploy-compatible PointPillar package.

## 1. Final Goal

Prepare an A9 dataset pipeline and PointPillar output that can be deployed on the lidar server without modifying lidar server code.

Final deploy unit:

- OpenPCDet checkpoint: `.pth`
- Matching PointPillar model config: `.yaml`
- Lidar runtime param file: `.param.yaml`

The current target is not full training. The current target is:

```text
A9 data contract -> A9 config/param split -> smoke checkpoint -> lidar server actual load test
```

Full training is allowed only after smoke validation proves that the deploy contract is correct.

## 2. Thread Map

| Thread ID | Short Name | Workspace / Server | Responsibility |
|---|---|---|---|
| `019dcca8-389a-7033-87f4-204909603c9e` | Command Center | Current control thread | User command intake, final report consolidation |
| `019d759d-9bd8-70d1-84b0-266e4d89ef90` | Build Station | `Model_Training_Server` | A9 conversion, config/param preparation, smoke training |
| `019d75bd-2f2c-7cd2-9fa1-46a4f239d99c` | Model QA / Coordinator | `Model_Training_Server` | Risk gates, decision tracking, go/yellow/no-go judgment |
| `019d75fd-6ada-7b31-9eed-5673b2826f9e` | Lidar Runtime Reviewer | `Lidar_Detection_Server` | Lidar runtime contract, deploy path, launch/load validation |

Important naming note:

`019d75bd...` and `019d75fd...` may both appear as `QA_Station` in Codex. Do not distinguish them by title. Distinguish them by server:

- `019d75bd...` = model server QA / coordinator
- `019d75fd...` = lidar server runtime reviewer

## 3. Communication Protocol

The threads cannot automatically message each other through Codex desktop. This repository and issue #1 are the shared mailbox.

Preferred workflow:

1. Command Center writes the request or target in issue #1.
2. Each participant reads this guide and issue #1.
3. Each participant performs only its assigned role.
4. Each participant posts a report to issue #1 if GitHub access is available.
5. If a participant cannot post to GitHub, it returns the report in its own thread and the user or Command Center relays it.
6. Coordinator reads Build Station and Lidar Runtime Reviewer reports, then marks the state as `GREEN`, `YELLOW`, or `NO-GO`.
7. Command Center gives the user the final consolidated report.

Do not use one thread to guess the other server's facts. The model server should ask for lidar runtime facts. The lidar server should ask for model artifact facts.

## 4. Fixed Deployment Contract

The following contract is fixed unless the Coordinator explicitly changes it after evidence from both servers.

| Item | Contract |
|---|---|
| Architecture | `PointPillar` |
| Checkpoint | OpenPCDet `.pth` |
| Deploy set | `.pth + yaml + param` |
| Class count | `3` |
| Class order | `Car`, `Pedestrian`, `Cyclist` |
| Class ID meaning | `1=Car`, `2=Pedestrian`, `3=Cyclist` |
| Runtime threshold array | `[0.7, 0.4, 0.4]` |
| Feature count | `num_features=4` |
| Inference feature policy | `x, y, z, 0` |
| Runtime path policy | `config_file` and `model_file` use package-share relative paths |

The fourth feature channel must not be left implicit. For the smoke phase, the A9 training conversion must explicitly create `(N, 4)` points and set column 4 to zero.

## 5. Known Facts From Existing Threads

### Model Server Facts

Observed from `019d759d...` and `019d75bd...`:

- Existing KITTI/PointPillar work proved the training pipeline direction, but A9 is the real target.
- Current state is `YELLOW`: feasible, but not ready for full training.
- Model server exact OpenPCDet commit was not confirmed from `.git`.
- Model server environment was reported as approximately:
  - Python: `3.10.19`
  - torch: `1.13.1`
  - CUDA: `11.7`
  - spconv: `2.3.6`
  - pcdet: `0.6.0+0000000`
- A9 source dataset location and original class names were not confirmed inside the project tree.
- Existing custom config path contains `Vehicle` assumptions and must not be reused blindly.
- A9-specific config, param, and conversion script must be new files.
- Training-side fourth channel zero injection was not yet implemented.

### Lidar Server Facts

Observed from `019d75fd...`:

- Runtime uses `PointPillar` successfully with current KITTI-style artifacts.
- Current runtime param values include:
  - `config_file: cfgs/kitti_models/pointpillar.yaml`
  - `model_file: checkpoints/pointpillar_7728.pth`
  - `num_features: 4`
  - `threshold_array: [0.7, 0.4, 0.4]`
- Actual runtime package share path was observed as:
  - `/root/workspace_lidar/lidar_detection_ws/install/pcdet_ros2/share/pcdet_ros2`
- Lidar server contract was reported as:
  - OpenPCDet commit: `8cacccec11db6f59bf6934600c9a175dae254806`
  - pcdet: `0.6.0+8caccce`
  - torch: `2.1.2+cu118`
  - spconv: `2.3.6`
  - CUDA: `11.8`
  - Python: `3.8.10`
- Param file can be launched from an explicit path, but `config_file` and `model_file` inside the param are package-share relative.
- Deploy validation must include install/share reflection and actual node launch.

## 6. Current Risk Register

| Risk | Owner | Severity | Required action |
|---|---|---|---|
| Model/lidar OpenPCDet or checkpoint compatibility mismatch | Coordinator + Build Station + Lidar Reviewer | High | Prove with smoke `.pth` load on lidar server |
| A9 original class names unknown | Build Station | High | Locate A9 labels and define mapping to `Car/Pedestrian/Cyclist` |
| Existing custom files use `Vehicle` | Build Station | High | Create A9-specific files; do not overwrite existing custom/KITTI files |
| Fourth feature policy not implemented in training conversion | Build Station | High | Force `(N,4)` and zero-fill column 4 in conversion script |
| Runtime path misunderstanding | Lidar Reviewer | Medium | Verify package-share relative paths and installed files |
| Class ID / threshold order mismatch | Coordinator + Lidar Reviewer | High | Verify `1/2/3` maps to `Car/Pedestrian/Cyclist` at runtime |
| Full training starts before smoke validation | Coordinator | High | Keep state `YELLOW` or `NO-GO` until smoke gates pass |

## 7. Per-Thread Instructions

### 7.1 Command Center

Thread ID: `019dcca8-389a-7033-87f4-204909603c9e`

Role:

- Receive user instructions.
- Keep this GitHub repo as the coordination source.
- Ask each participant thread to read the guide and issue #1.
- Consolidate reports into one final user-facing decision.

Do:

- Maintain the final task framing.
- Ask for missing reports from Build Station, Coordinator, or Lidar Runtime Reviewer.
- Summarize state as `GREEN`, `YELLOW`, or `NO-GO`.

Do not:

- Guess server-specific facts without reports.
- Approve full training by itself.

Prompt to send to this thread:

```text
You are the Command Center for A9 work.
Read https://github.com/jakebeep00/A9-Dataset-Control-Tower-/blob/main/docs/A9_CONTROL_TOWER_GUIDE.md and issue #1.
Collect reports from the three participant roles and give me the final decision, risks, and next actions.
```

### 7.2 Build Station

Thread ID: `019d759d-9bd8-70d1-84b0-266e4d89ef90`

Workspace/server: `Model_Training_Server`

Role:

- Build A9 dataset conversion and model-side artifacts.
- Prepare smoke training.
- Produce deploy-compatible `.pth + yaml + param` candidates.

Must do now:

1. Locate A9 source data and label files.
2. Identify original A9 class names.
3. Define mapping into exactly `Car`, `Pedestrian`, `Cyclist`.
4. Create A9-specific conversion logic.
5. Ensure converted point arrays are `(N,4)` with the fourth feature set to zero.
6. Create or propose `cfgs/custom_models/pointpillar_a9.yaml`.
7. Create or propose `config/pcdet_a9_pointpillar.param.yaml`.
8. Keep all A9 files separate from existing KITTI/custom files.
9. Run only smoke training when gates are ready.
10. Report exact artifact paths and commands.

Must not do:

- Do not start full training.
- Do not use `Vehicle` as the target class unless Coordinator explicitly changes the contract.
- Do not overwrite existing KITTI/custom configs, params, or checkpoints.
- Do not assume lidar runtime paths without Lidar Runtime Reviewer confirmation.

Build Station report must include:

```text
[Build Station Report]
Thread ID: 019d759d-9bd8-70d1-84b0-266e4d89ef90
A9 source path:
Original A9 classes:
Class mapping to Car/Pedestrian/Cyclist:
Conversion script path:
Proof of (N,4) zero-filled fourth channel:
A9 config path:
A9 param draft path:
Smoke training command:
Smoke checkpoint path:
Blockers:
Questions for Lidar Runtime Reviewer:
Go/Yellow/No-Go recommendation:
```

Prompt to send to this thread:

```text
You are Build Station for A9 work.
Read https://github.com/jakebeep00/A9-Dataset-Control-Tower-/blob/main/docs/A9_CONTROL_TOWER_GUIDE.md and issue #1.
Act only as thread `019d759d-9bd8-70d1-84b0-266e4d89ef90`.
Your job is A9 dataset conversion, A9 PointPillar config/param preparation, and smoke training preparation on Model_Training_Server.
Do not start full training. Report using the Build Station format from the guide.
```

### 7.3 Model QA / Coordinator

Thread ID: `019d75bd-2f2c-7cd2-9fa1-46a4f239d99c`

Workspace/server: `Model_Training_Server`

Role:

- Own the decision logic.
- Check whether Build Station and Lidar Runtime Reviewer reports satisfy the deployment contract.
- Maintain `GREEN`, `YELLOW`, or `NO-GO` state.

Must do now:

1. Keep status as `YELLOW` until smoke evidence exists.
2. Verify Build Station removed `Vehicle` assumptions from A9 artifacts.
3. Verify class order is exactly `Car`, `Pedestrian`, `Cyclist` everywhere.
4. Verify fourth feature zero-fill is implemented in the conversion path, not only described.
5. Verify model/lidar environment compatibility risk is either resolved or explicitly carried to smoke load test.
6. Verify Lidar Runtime Reviewer confirms installed package paths and launch behavior.
7. Decide whether smoke can proceed.
8. Decide whether full training remains blocked.

Coordinator report must include:

```text
[Coordinator Decision]
Thread ID: 019d75bd-2f2c-7cd2-9fa1-46a4f239d99c
Current state: GREEN / YELLOW / NO-GO
Evidence reviewed:
Satisfied gates:
Unsatisfied gates:
Required action from Build Station:
Required action from Lidar Runtime Reviewer:
Full training decision:
Reason:
```

Prompt to send to this thread:

```text
You are Model QA / Coordinator for A9 work.
Read https://github.com/jakebeep00/A9-Dataset-Control-Tower-/blob/main/docs/A9_CONTROL_TOWER_GUIDE.md and issue #1.
Act only as thread `019d75bd-2f2c-7cd2-9fa1-46a4f239d99c`.
Your job is risk gates, deployment-contract validation, and go/yellow/no-go judgment.
Do not implement Build Station tasks. Do not approve full training until smoke gates pass.
Report using the Coordinator Decision format from the guide.
```

### 7.4 Lidar Runtime Reviewer

Thread ID: `019d75fd-6ada-7b31-9eed-5673b2826f9e`

Workspace/server: `Lidar_Detection_Server`

Role:

- Verify actual lidar runtime contract.
- Confirm where config, checkpoint, and param must be placed.
- Validate actual launch/load behavior with smoke artifacts.

Must do now:

1. Confirm actual runtime param path.
2. Confirm package share path for `pcdet_ros2`.
3. Confirm whether `config_file` and `model_file` inside param are package-share relative.
4. Confirm current runtime class order and threshold dependency.
5. Confirm rebuild command and install/share reflection.
6. Prepare exact smoke validation command sequence.
7. When Build Station provides smoke artifacts, test actual load and launch.
8. Verify `/cloud_detections` publication and class ID meaning.

Must not do:

- Do not change the model architecture.
- Do not assume training-side conversion details.
- Do not approve model quality from runtime boot alone.

Lidar Runtime report must include:

```text
[Lidar Runtime Report]
Thread ID: 019d75fd-6ada-7b31-9eed-5673b2826f9e
Runtime param path:
Package share path:
Expected config_file relative path:
Expected model_file relative path:
Rebuild command:
Launch command:
Install/share reflection check:
/cloud_detections check:
Class ID check:
Smoke load result:
Blockers:
Questions for Build Station:
Go/Yellow/No-Go recommendation:
```

Prompt to send to this thread:

```text
You are Lidar Runtime Reviewer for A9 work.
Read https://github.com/jakebeep00/A9-Dataset-Control-Tower-/blob/main/docs/A9_CONTROL_TOWER_GUIDE.md and issue #1.
Act only as thread `019d75fd-6ada-7b31-9eed-5673b2826f9e`.
Your job is lidar server deployment contract verification and actual smoke artifact load/launch validation.
Do not guess model-server details. Report using the Lidar Runtime Report format from the guide.
```

## 8. Smoke Gate Checklist

Smoke can start only when these preparation gates are satisfied:

| Gate | Owner | Status |
|---|---|---|
| A9 source data path found | Build Station | `PENDING` |
| A9 original class names found | Build Station | `PENDING` |
| Mapping to `Car/Pedestrian/Cyclist` defined | Build Station + Coordinator | `PENDING` |
| A9 conversion creates `(N,4)` with fourth channel zero | Build Station | `PENDING` |
| A9 config is separate and class-consistent | Build Station + Coordinator | `PENDING` |
| A9 param is separate and package-share relative | Build Station + Lidar Reviewer | `PENDING` |
| Runtime package/share paths confirmed | Lidar Reviewer | `PENDING` |
| Rebuild and launch commands confirmed | Lidar Reviewer | `PENDING` |
| Smoke checkpoint generated | Build Station | `PENDING` |
| Smoke checkpoint loads on lidar server | Lidar Reviewer | `PENDING` |
| `/cloud_detections` publishes | Lidar Reviewer | `PENDING` |
| Coordinator marks smoke pass | Coordinator | `PENDING` |

## 9. Full Training Rule

Full training remains blocked while any of the following are true:

- A9 source data or labels are not confirmed.
- Class mapping is not proven.
- Any A9 artifact still uses `Vehicle` unexpectedly.
- Fourth feature zero-fill is not implemented in code.
- Smoke checkpoint does not exist.
- Lidar server has not actually loaded the smoke `.pth` with the A9 yaml and param.
- Runtime class IDs are not verified.
- Coordinator status is not `GREEN`.

## 10. Final Report Format For Command Center

When the Command Center reports to the user, use this format:

```text
[A9 Control Tower Final Report]
Current state: GREEN / YELLOW / NO-GO
Final goal:
What Build Station confirmed:
What Lidar Runtime Reviewer confirmed:
Coordinator decision:
Remaining blockers:
Next action for each thread:
Full training decision:
```
