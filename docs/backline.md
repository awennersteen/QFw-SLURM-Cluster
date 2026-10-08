# Backline QPM (`backline-local`)

`backline-local` is a client-executed QPM (QFw `services/svc_backline_qpm`, branch
`backline-qpm`). A reservation picks a Backline controller, QEC decoder coprocessor and
transport from the site inventory; the application reads that placement and runs its
PennyLane/Catalyst program on it. It never sends circuits through QRI.

It runs on `qec-backline-head` and is opt-in: `qfw-site-services start` (target `all`) does not
start it.

## Set up once

The Backline QPM is on the `backline-qpm` branch of
[josephleekl/QFw](https://github.com/josephleekl/QFw/tree/backline-qpm). The image keeps its
stock QFw; the Backline QPM runs from a development override built from that branch (see
"Build and use a development QFw override" in the README).

On the host:

```bash
git clone -b backline-qpm https://github.com/josephleekl/QFw-SLURM-Cluster
cd QFw-SLURM-Cluster
SHARED=/path/to/shared-dir
./do_configure.sh --prefix "$SHARED"
git clone --recursive -b backline-qpm https://github.com/josephleekl/QFw "$SHARED/QFw"
./do_build.sh
./do_startup.sh
./do_qfw_build.sh --clean                     # the override, in $SHARED/qfw-install
```

In the cluster:

```bash
./do_ssh.sh                                   # into the cluster, as root
/path/to/QFw-SLURM-Cluster/tools/install-backline-venv.sh   # shared venv: Catalyst 0.16.0b1 wheels
qfw-site-services start                       # directory, gateway and the default QPMs
export QFW_INSTALL_PREFIX=/workspace/qfw-container-base/qfw-install   # the override
qfw-site-services --target backline start
qfw-site-services --target backline status
```

The `backline` environment module (`modulefiles/backline`) points `CATALYST_LIB` at the
shared venv, so the QPM's inventory finds the precompiled decoders.

## Run a Backline program

The job script is `qri-backline/examples/job.sh`, cloned to
`/workspace/qfw-container-base/qri-backline` by `tools/install-backline-venv.sh`:

```bash
sbatch -A <account> /workspace/qfw-container-base/qri-backline/examples/job.sh
```

qfw-slurm requires a Slurm account (`-A`); the provisioned test users use `qfw-test`.

The job makes two requests:

1. **Quantum resources.** The `#SBATCH --qpu=backline-local ...` lines. qfw-slurm reserves
   `backline-local`'s quantum budget while the job is allocated and passes the reservation to
   each step in `QFW_RESERVATIONS`. The allocation's teardown releases it.
2. **Classical QEC resources.** The program asks the QPM for them against that reservation
   (`reserve` with `for_reservation` and the `resource_intent` in
   `intents/cpu-steane-memcpy.yaml`: a CPU controller, a Steane decoder, memcpy). The QPM
   starts a `catalyst-executor` on `qec-backline-head` for the reservation and returns its
   address in the placement; the program compiles locally and runs controller and decoder
   there. The executor stops when the reservation ends.

`--max-qubits` counts logical qubits.

## Files

| File | Role |
| --- | --- |
| `docker-compose.yml`, `slurm.conf`, `tools/provision-qfw-cluster.sh` | the `qec-backline-head` node |
| `config/site-services.yaml` | the `backline-local` service, loading the `backline` module |
| `config/device-access.yaml`, `config/qpu-users.json` | device record and user entitlement |
| `config/qfw-slurm/resources.lua`, `config/qfw-slurm/plugin.conf` | `--qpu=backline-local` |
| `modulefiles/backline`, `tools/install-backline-venv.sh` | the shared Backline venv and the demo checkout |
| `tools/qfw-site-services` | `--target backline` start, stop, status |
