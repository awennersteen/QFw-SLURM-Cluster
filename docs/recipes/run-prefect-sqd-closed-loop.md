# Run the Prefect SQD Closed Loop

This recipe runs the Prefect-orchestrated SQD closed-loop workflow from
[qiskit-community/qcsc-prefect](https://github.com/qiskit-community/qcsc-prefect)
on the cluster. The workflow optimizes LUCJ circuit parameters by differential
evolution; each walker samples a circuit, recovers configurations with
qiskit-addon-sqd, and runs the SBD Davidson solver as a Slurm MPI job. Its
upstream tutorial is
[run_sbd_closed_loop_workflow_qfw-slurm.md](https://github.com/qiskit-community/qcsc-prefect/blob/main/docs/tutorials/run_sbd_closed_loop_workflow_qfw-slurm.md).

The cluster adds two host tools and a compose file so that the tutorial's
manual setup becomes repeatable:

| Piece | Purpose |
| --- | --- |
| `tools/provision-prefect.sh` | Build the workflow environment on the shared workspace. |
| `tools/prefect-sqd` | Start the Prefect server, create the blocks, submit runs, stop. |
| `docker-compose.prefect.yml` | Publish the Prefect UI on the Docker host. |

Everything runs as a non-root application user. The Prefect server, the
served deployment and the workflow process live in `slurmctld`; the SBD
solver runs on the `normal` partition through `sbatch` and `srun`.

## Prerequisites

- The cluster is running and provisioned (`tools/provision-qfw-cluster.sh`),
  so `user-a` exists with the `qfw-test` Slurm account.
- The Docker host can reach PyPI and GitHub from inside `slurmctld`.
- The site service plane is not needed for the `random` quantum source.
  See [The qfw quantum source](#the-qfw-quantum-source) for what the real
  QFw path needs.

## Publish the Prefect UI

The Prefect server listens on port 4200 inside `slurmctld`. To reach its UI
from the Docker host, append the Prefect compose file to `COMPOSE_FILE` in
`.env`:

```text
COMPOSE_FILE=docker-compose.yml:docker-compose.override.yml:docker-compose.telemetry.yml:docker-compose.prefect.yml
```

The mapping binds `127.0.0.1:4200` and takes effect when `slurmctld` is
recreated, which also means running `tools/provision-qfw-cluster.sh` and
`qfw-site-services start` again. Without it, drive the workflow from the
command line as shown below.

## Provision the Workflow Environment

From the Docker host:

```bash
cd /path/to/QFw-SLURM-Cluster
tools/provision-prefect.sh
```

The script is safe to rerun. Its one root step creates the Prefect
workspace, `/workspace/qfw-container-base/qfw-prefect` (`QFW_PREFECT_ROOT`
overrides), and hands it to `user-a`: the shared mount's root belongs to
whoever owns it on the Docker host, so an application user cannot create
anything there directly. Everything else runs as `user-a` inside that
workspace:

1. Clones `qcsc-prefect` to `qcsc-prefect/` at the commit this recipe was
   validated with (`QCSC_PREFECT_REF` overrides).
2. Creates `venv/` with the image venv's exact package set, so QFw's pins
   (qiskit, qrmi, the QDMI adapters) stay in force, then adds Prefect, Ray,
   ffsim, pyscf, qiskit-addon-sqd and the qcsc-prefect packages. One image
   pin is overridden: `ruamel-yaml`, which the IQM QDMI adapter pins at 0.17
   while every `pydantic_numpy` that imports under NumPy 2.5 needs 0.18.
   Nothing on the Prefect path uses it. FastAPI is held below 0.150: Prefect
   3.8.8's server still passes `min_items` to `Body()`, which FastAPI dropped
   inside the range Prefect allows, and the server then fails to start.
3. Rebuilds ffsim from source when its wheel dies with an illegal instruction,
   which it does on Apple Silicon hosts.
4. Builds the native SBD solver with the image's OpenMPI and OpenBLAS.
5. Installs the quantum-stage sampler and its submitter under `bin/`.
6. Verifies the venv under `qfw-activate`, which is how the workflow uses it.

The workspace lives on the shared mount and survives a container recreate.
Expect several minutes on first run, mostly the ffsim build.

## Start the Server and Serve the Deployment

```bash
PREFECT_RAY_NUM_CPUS=4 tools/prefect-sqd start
tools/prefect-sqd blocks
tools/prefect-sqd status
```

`start` launches the Prefect server in the background and serves the
`riken-sqd-de/riken_sqd_de` deployment from a `nohup` process
(`/workspace/home/user-a/sbd-deploy.log`). `PREFECT_RAY_NUM_CPUS` bounds the
Ray task runner that fans the walkers out; set `SBD_TASK_RUNNER=concurrent`
to use threads instead.

`blocks` writes `/workspace/home/user-a/sbd_blocks.toml` for this cluster
(account `qfw-test`, partition `normal`, one node, `SBD_MPIPROCS` ranks,
`SBD_WALLTIME`), creates the Slurm SBD blocks from it, registers the
`QFwResource` block type from the main thread and creates the `qfw-runner`
block for `QFW_PREFECT_DEVICE` (default `ornl-iqm-20q`).

## Run the Workflow

```bash
tools/prefect-sqd run random 10000 1
```

The arguments are the quantum source, the SQD subspace dimension and the
number of differential-evolution iterations. The command submits a flow run
for the N2 example and waits for it. Each iteration launches four walkers,
and each walker submits one SBD job:

```bash
docker exec slurmctld squeue -u user-a
docker exec slurmctld sacct -u user-a -X \
  --format=JobID,JobName%20,State,ExitCode,Elapsed,NodeList
```

Job directories, generated `sbd_solver.slurm` scripts and solver output
land under `/workspace/home/user-a/sbd_jobs/`. The run's energies are in the
`sqd-telemetry` artifact, which the UI shows under the flow run or
`prefect artifact inspect sqd-telemetry` prints.

On this cluster the command above finishes in about half a minute: the four
walkers run in Ray workers, each SBD job spends about five seconds on its
node, and the four walkers report energies near -134.77 Ha for N2 with the
fast solver preset.

With the UI published, the same run starts from the deployment page with
**Run → Custom run** and the tutorial's parameter table.

## The qfw Quantum Source

With `quantum_source=qfw`, the workflow loads the `qfw-runner` block and
samples through QFw's QRMI driver **inside the workflow process**: it reads
the device-access file named in the block, resolves the calling user's
credentials from that file's credential database, opens the QRMI resource
and runs the circuit. It does not reserve the QPM through qfw-slurm, does not
talk to the site QPM service, and leaves no trace in QFw's telemetry.

That has consequences on this cluster:

- The site device-access file and credential database under
  `/etc/openqse/qfw/device` are root-only. The workflow user needs its own
  readable device-access file whose `credential-db` holds that user's
  records. Point the block at it with `QFW_PREFECT_DEVICE_ACCESS` before
  `tools/prefect-sqd blocks`.
- Only QRMI devices work: `ornl-iqm-20q` (needs the IQM tunnel) and
  `ibm-156-nh` (needs an IBM key). The NWQSim simulator, the fake IQM QPM and
  the QDMI devices, Braket included, cannot be selected.
- Reset mitigation is off in the run command; the tutorial recommends that
  for direct-access systems.

The cluster's gitignored IQM files, `shared-dir/iqm-device-access.yaml` and
`shared-dir/iqm-creds.json`, already form such a pair for `root` through the
IQM tunnel. With the tunnel up, run the workflow process as root against
them. The Prefect server keeps running as `user-a`; only the served
deployment changes hands, and root's SBD work directory must be on the
shared workspace because root's home is not:

```bash
tools/prefect-sqd stop serve
QFW_PREFECT_USER=root QFW_PREFECT_DEVICE=ornl-iqm-20q \
  QFW_PREFECT_DEVICE_ACCESS=/mnt/iqm-device-access.yaml \
  SBD_WORK_DIR=/workspace/qfw-container-base/sbd_jobs \
  tools/prefect-sqd blocks
QFW_PREFECT_USER=root PREFECT_RAY_NUM_CPUS=4 tools/prefect-sqd start
QFW_PREFECT_USER=root tools/prefect-sqd run qfw 10000 1
```

The SBD jobs of that run are then submitted by root. Afterwards,
`QFW_PREFECT_USER=root tools/prefect-sqd stop serve` and a plain `blocks`
and `start` return the deployment to `user-a`. This path has not been
exercised on the cluster yet; the tunnel was down when the recipe was
written.

## Stop

```bash
tools/prefect-sqd stop
```

This stops the served deployment and the Prefect server. The venv, checkout,
blocks and run history stay on the shared workspace.
