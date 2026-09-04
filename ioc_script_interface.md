# Runtime User Script Interface

## Purpose

The IOC currently exposes a fixed set of Python functions through the
`$(P)Py:UserScript1`, `$(P)Py:UserScript2`, and `$(P)Py:UserScript3` records.
Adding or changing one of these operations requires editing IOC support code and
restarting the IOC.

The proposed interface would let a user:

1. Define robot operations as ordinary Python functions in one designated module.
2. Select a function by name through an EPICS PV.
3. Execute it through a general-purpose command PV.
4. Modify the function and run the new version without restarting the IOC.

The goal is to provide this flexibility with a small addition to the existing
worker and status infrastructure, rather than redesigning the IOC.

## Proposed User Interface

Add two records:

```text
$(P)Py:ScriptName
$(P)Py:RunScript
```

A typical client interaction would be:

```sh
caput 12idUR:Py:ScriptName rotate_wrist
caput 12idUR:Py:RunScript 1
```

The selected function would be defined in a fixed module such as
`user_scripts.py`:

```python
def rotate_wrist(robot):
    robot.rotj(0, 0, 0, 0, 0, 45)
    robot.rotj(0, 0, 0, 0, 0, -45)
```

For the initial implementation, every callable script should have the signature:

```python
def script_name(robot):
    ...
```

Arguments beyond the connected robot object should not be included initially.
Typed argument PVs or a JSON argument record can be considered separately if a
concrete need develops.

## Execution Model

Processing `$(P)Py:RunScript` would:

1. Reset the command record to its idle value.
2. Read the function name from `$(P)Py:ScriptName`.
3. Validate the name before accepting the operation.
4. Submit the operation through the existing `RobotConnection` worker.
5. Recheck robot readiness when the worker starts the operation.
6. Reload `user_scripts.py` inside the worker thread.
7. Resolve the selected function in the reloaded module.
8. Verify that the selected object is callable.
9. Call the function with the currently connected robot.
10. Report completion through the existing operation status records.

The existing worker remains responsible for serializing connection and robot
operations. A runtime script must not execute directly in an EPICS record
processing callback.

The existing status interface would report the operation:

```text
$(P)Py:Running
$(P)Py:CurrentOperation
$(P)Py:OperationState
$(P)Py:OperationId
$(P)Py:CompletedId
$(P)Py:OperationError
```

`CurrentOperation` should contain the selected script name, or a value such as
`Script:rotate_wrist`, so users can identify the running code.

## Reload Behavior

The script module should be reloaded automatically immediately before each
execution. This is preferable to adding a separate reload command because it
guarantees that `RunScript` always attempts to use the latest saved source.

The implementation would use Python's import machinery conceptually as follows:

```python
import importlib
import user_scripts


def run_user_script(robot, script_name):
    module = importlib.reload(user_scripts)
    operation = getattr(module, script_name)
    if not callable(operation):
        raise TypeError(f"User script is not callable: {script_name}")
    operation(robot)
```

The actual implementation should validate `script_name` before `getattr()`.

Reloading affects the next operation only. It cannot replace or interrupt a
function that is already running.

### Reload Failures

Syntax errors, failed imports, missing functions, and runtime exceptions should
all fail only the submitted operation. They should:

- Set `OperationState` to `Failed`.
- Copy the exception type and message to `OperationError`.
- Write the complete traceback to the IOC log.
- Leave the IOC and robot connection running when possible.

Python's normal reload behavior modifies an existing module object. If reload
fails after executing part of the module, some old or partially updated module
state may remain. The next execution should always attempt another reload rather
than assuming that the previous module state is valid.

## Module Boundary

Only one fixed, explicitly configured module should be reloadable. The initial
implementation should not accept an arbitrary filesystem path or module name
through a PV.

Recommended:

```text
user_scripts.py
```

Not recommended:

```text
$(P)Py:ScriptPath = /arbitrary/path/code.py
$(P)Py:ModuleName = arbitrary.module
```

A fixed module provides the requested editing workflow without creating a
general-purpose remote Python loader.

`robot_operations.py` itself must not be reloaded. It owns long-lived IOC state,
including:

- The AprilTag tracker.
- IOC lifecycle hooks.
- I/O Intr scan lists.
- Command support objects attached to records.
- References used by the robot worker.

Reloading that module could duplicate hooks and scan lists, replace global state,
or leave EPICS records attached to obsolete Python objects.

The user script module should contain function definitions and ordinary helper
code. It should avoid module-level robot motion, threads, sockets, open files, or
other persistent resources because all module-level statements run during every
reload.

## Name Validation

The selected name should be restricted to a public Python identifier:

- It must be a string accepted by `str.isidentifier()`.
- It must not begin with `_`.
- It must refer directly to an attribute of the designated script module.
- Dotted paths such as `robot.dashboard.stop` must not be accepted.

Examples:

```text
rotate_wrist       accepted
pick_sample_2      accepted
_private_helper    rejected
robot.movej        rejected
../other_file      rejected
```

This is not a security sandbox. A permitted function can execute arbitrary Python
and receives the live robot object. Filesystem permissions and EPICS access
security remain the real authorization boundaries.

## Database Changes

The minimal database addition is approximately:

```db
record(stringout, "$(P)Py:ScriptName") {
    field(DESC, "User script function name")
    field(VAL, "user_script1")
}

record(bo, "$(P)Py:RunScript") {
    field(DESC, "Run selected Python script")
    field(DTYP, "Python Device")
    field(OUT, "@robot_operations RunScript $(P)Py:Ready $(P)Py:ScriptName")
    field(ZNAM, "Idle")
    field(ONAM, "Run")
}
```

A `stringout` record provides 40 characters in the EPICS versions relevant to
this IOC, which is sufficient for a Python function name. A long-string record is
not needed unless the interface later accepts larger structured input.

## Python Support Changes

`robot_operations.py` would gain support for a `RunScript` link with this form:

```text
@robot_operations RunScript $(P)Py:Ready $(P)Py:ScriptName
```

The support object should read `ScriptName` when `RunScript` is processed and
capture that value in the submitted operation. This is important because the
user could change `ScriptName` while the request is waiting in the worker queue.
The queued operation must run the name that was selected when it was submitted.

The operation passed to `RobotCommand` would reload the module and invoke the
captured function name. The implementation can reuse:

- `RobotCommandSupport` for command-record behavior.
- `RobotConnection.submit_robot_operation()` for readiness checks.
- `RobotConnection._run_operation()` for exception reporting and completion
  status.
- The existing single `Worker(max=1)` for serialization.

No change to the overall worker architecture is required.

The most likely implementation is a small dedicated support class because the
current `RobotCommandSupport` stores a fixed function when records attach during
IOC initialization. The selected function name instead needs to be read at each
record processing request.

## Existing UserScript Records

The initial implementation should retain `UserScript1`, `UserScript2`, and
`UserScript3` unchanged while adding the general interface. This minimizes risk
and provides known fallback commands while runtime reload is tested.

After the new mechanism has been used successfully, the fixed records could
either remain as stable shortcuts or be removed in a separate change. Their
removal is not required for this feature.

## Concurrency

Module reload and function lookup must occur inside the serialized worker
operation, not while the EPICS output record is processing. This ensures that:

- A module cannot be reloaded while another user script is executing.
- Robot commands remain serialized with connection and existing script commands.
- Slow imports do not block an EPICS database processing thread.
- Exceptions flow through the existing operation reporting path.

The current IOC silently ignores robot commands submitted while another operation
is running. The new command would initially inherit that behavior unless it is
changed separately. Explicit busy rejection would be clearer, but it is outside
the minimum runtime-script implementation.

## Safety and Operational Constraints

This interface intentionally allows trusted users to run arbitrary Python in the
IOC process. It is not appropriate for untrusted users.

Important constraints are:

- A script has the same operating-system privileges as the IOC.
- A script receives the live robot object and can call low-level methods.
- A Python exception does not guarantee that physical robot motion stopped.
- Reload cannot cancel a running function.
- EPICS Channel Access is not a safety-rated control system.
- Robot controller safeguards, workspace limits, emergency stops, and access
  security remain necessary.

The script directory should be writable only by authorized developers. EPICS
access security should restrict writes to `ScriptName` and `RunScript` to trusted
clients.

User scripts should be instructed to:

- Raise an exception when an operation fails.
- Check false or negative return values from `UR_12idb` methods.
- Avoid catching broad exceptions unless they are re-raised after cleanup.
- Avoid creating independent robot connections.
- Avoid background threads and module-level persistent state.
- Return only after the requested operation has actually completed.

## Testing

Focused tests should use a fake robot and temporary script module. They should
cover:

1. Running an existing public function.
2. Observing modified function behavior on the next execution.
3. Rejecting an invalid identifier.
4. Rejecting a private name.
5. Reporting a missing function.
6. Rejecting a non-callable module attribute.
7. Reporting a syntax or import error during reload.
8. Reporting an exception raised by the selected function.
9. Capturing the selected name before submission.
10. Preserving serialization with another robot operation.
11. Leaving the IOC support objects usable after a failed reload.

Hardware motion should not be required for these tests.

## Implementation Steps

1. Add a side-effect-free `user_scripts.py` module with one harmless example or
   move copies of the existing scripts into it.
2. Add `ScriptName` and `RunScript` records to `robot_operations.db`.
3. Add a runtime script support class to `robot_operations.py`.
4. Validate and capture the selected function name during command processing.
5. Reload and resolve the function inside the existing worker operation.
6. Route failures through the existing operation status and logging behavior.
7. Add focused tests using a fake robot and reloadable test module.
8. Optionally add `ScriptName` and `RunScript` controls to the existing UI.

## Estimated Scope

The basic production implementation is small:

- User script module scaffolding: approximately 10 to 30 lines.
- Runtime dispatch support: approximately 20 to 40 lines.
- Database records: approximately 15 lines.
- Focused tests: approximately 50 to 100 lines.
- Optional UI controls: a small separate change.

No changes should be required in `UR_12idb`, the connection constructor, or the
worker architecture.

## Recommendation

Implement a fixed `user_scripts.py` module, automatic reload before every run, a
`ScriptName` PV, and a `RunScript` PV. Keep the existing three fixed user-script
records during initial deployment.

This provides most of the desired flexibility while limiting the change to one
small support class and two records. It also keeps long-lived IOC state outside
the reload boundary and preserves the IOC's existing robot-operation
serialization and status reporting.
