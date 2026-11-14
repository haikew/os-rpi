# Hardware tests

The [`hardware-test`](../workflows/hardware-test.yml) workflow answers one question: **does this
ImSwitch image, or this pallet, work on a real microscope?** It runs on a self-hosted GitHub
Actions runner on the microscope's Raspberry Pi:

1. `swap-in.sh` puts the image or pallet to test onto the machine.
2. The hardware tests run inside the ImSwitch container. They are part of ImSwitch
   ([`imswitch/imcontrol/_test/hil`](https://github.com/openUC2/ImSwitch/tree/master/imswitch/imcontrol/_test/hil)),
   so they always fit the ImSwitch version under test.
3. `restore.sh` puts the machine back, whether the tests passed, failed or were cancelled.

**The tests move the stage and switch light.**

## Starting a test

- **A pallet:** add the label `hardware-test` to a pull request in this repository. The label is
  removed when the run is done; add it again to test again. Or run the workflow by hand from the
  *Actions* tab on any branch, with the `image` input empty.
- **Only an ImSwitch image:** run the workflow by hand from the *Actions* tab and enter the image's
  tag (e.g. `sha-7d3adda`) as `image`. The machine keeps the pallet it runs.

The artifact `hil-report-pdf` contains `hil-report.pdf`: test results and photos followed by
this run's original FRAME calibration PDF. The complete `hil-report` artifact also includes
HTML, JUnit, the original calibration PDF and the pre-calibration setup backup. A summary is
shown on the run's page. Both uploads have a three-minute timeout and run before deployment
restoration; restoration is still attempted if an upload fails.

Calibration is attempted last even after preceding tests fail. New calibration results are
retained in ImSwitchConfig: restoring the deployment does not restore the setup. A calibration
that fails may have written partial changes. The backup permits manual recovery.

A pallet's files in `/etc` and `/usr` only take effect after a reboot, so only its containers are
tested.

## By hand, on the Pi

```bash
.github/hil/swap-in.sh image sha-7d3adda
.github/hil/swap-in.sh pallet github.com/openUC2/os-rpi@<commit>
```

```bash
docker exec imswitch-server-1 python3 -m imswitch.imcontrol._test.hil /tmp/report board_link camera_capture
docker cp imswitch-server-1:/tmp/report/. report
```

```bash
.github/hil/restore.sh
```

To test what the machine already runs, skip `swap-in.sh` and `restore.sh`. Leave out the test
names (`board_link camera_capture`) to run every test; add `--sync-firmware` to flash every board to the
firmware server's version first.

## How the swap works

Both modes use Forklift the usual way. `swap-in.sh` saves the number of the applied stage and a
copy of the local pallet to `~/.local/state/hil/`. Then it either edits the ImSwitch image in the
local pallet and stages it (`image`), or switches the local pallet to the one to test (`pallet`),
and applies the new stage. `restore.sh` applies the saved stage again, copies the saved pallet
back and removes the images which were downloaded for the test.

- A reboot in the middle of a test comes back on the original stage: `swap-in.sh` sets it as the
  next stage right after applying the one to test.
- If a run dies before it could restore, the next run restores first. `restore.sh` can also be
  run by hand at any time; it does nothing if nothing is swapped in.
- Pull requests from forks are not tested, because the workflow runs their code on our machine.

## Setting up the runner

Register the Pi under *Settings → Actions → Runners → New self-hosted runner* (Linux, ARM64) with
the label `uc2-rig`, and install it as a service:

```bash
sudo ./svc.sh install pi
sudo ./svc.sh start
```

It must run as `pi`: the user who owns the Forklift workspace and may use Docker without sudo.
