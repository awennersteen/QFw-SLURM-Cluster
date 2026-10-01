# Backline QPM (`backline-local`)

`backline-local` is a client-executed QPM (QFw `services/svc_backline_qpm`, branch
`backline-qpm`). A reservation picks a Backline controller, QEC decoder coprocessor and
transport from the site inventory; the application reads that placement and runs its
PennyLane/Catalyst program on it. It never sends circuits through QRI.

It runs on `backline-head` and is opt-in: `qfw-site-services start` (target `all`) does not
start it.

## Set up once

```bash
./do_ssh.sh                                   # into the cluster, as root
/path/to/QFw-SLURM-Cluster/tools/install-backline-venv.sh   # shared venv: Catalyst 0.16.0b1 wheels
qfw-site-services start                       # directory, gateway and the default QPMs
qfw-site-services --target backline start
qfw-site-services --target backline status
```

The `backline` environment module (`modulefiles/backline`) points `CATALYST_LIB` at the
shared venv, so the QPM's inventory finds the precompiled decoders.

## Run a Backline program

```bash
#!/bin/bash
#SBATCH --qpu=backline-local --workload-kind=quantum
#SBATCH --circ-count=1 --max-qubits=3 --max-depth=3 --max-shots=10
#SBATCH -N 1

module load backline
source "${QFW_PREFIX}/bin/qfw-activate" --venv "${BACKLINE_VENV}"
qfw-srun demo_qri_1_local_memcpy.py --qfw
```

qfw-slurm reserves `backline-local` while the job is allocated; the QPM applies its default
intent (Steane decoding, any matching controller and coprocessor). Each step gets
`QFW_RESERVATIONS`; the program reads its placement with `get_reservation` and does not
release it. The allocation's teardown does.

`--max-qubits` counts logical qubits.

## Files

| File | Role |
| --- | --- |
| `docker-compose.yml`, `slurm.conf`, `tools/provision-qfw-cluster.sh` | the `backline-head` node |
| `config/site-services.yaml` | the `backline-local` service, loading the `backline` module |
| `config/device-access.yaml`, `config/qpu-users.json` | device record and user entitlement |
| `config/qfw-slurm/resources.lua` | `--qpu=backline-local` |
| `modulefiles/backline`, `tools/install-backline-venv.sh` | the shared Backline venv |
| `tools/qfw-site-services` | `--target backline` start, stop, status |
