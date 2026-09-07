# AlpaSim Setup Troubleshooting Q&A

Issues hit while setting up and running AlpaSim on an aarch64 (GB300/Grace) Linux
workstation, and their fixes. Not all of these are aarch64-specific.

## `uv`/setup

### `Sorry, home directories outside of /home needs configuration` during `source setup_local_env.sh`

Cause: `$HOME` is outside `/home` (e.g. `/localhome/<user>`), which snap's
confinement blocks by default for any snap-packaged tool in the chain
(including a snap-installed `uv`).

Fix:

```bash
sudo snap set system homedirs=/localhome
```

If `uv` itself was installed via `sudo snap install astral-uv --classic`, also
replace it with the official installer so it isn't snap-confined at all:

```bash
sudo snap remove astral-uv
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
which -a uv   # should show only ~/.local/bin/uv, not /snap/bin/uv
```

### `rustup could not choose a version of cargo to run` while building `utils_rs`

Cause: no default rustup toolchain configured. `utils_rs` (maturin/PyO3
extension) needs `cargo` to build.

Fix:

```bash
rustup default stable
cargo --version   # confirm it now works
```

### `pyqt5-qt5` has no wheel for `manylinux_2_39_aarch64`

Cause: `setup_local_env.sh` always requests `--extra all`, which pulls in
`alpasim-tools` (`src/tools/pyproject.toml`), which hard-depends on `pyqt5`
(only used for the manual/keyboard-driver GUI). PyQt5's `pyqt5-qt5` wheel has
no aarch64 Linux build upstream.

Fix: skip `setup_local_env.sh`'s `uv sync --extra all ...` step and sync
directly with every extra except `tools`:

```bash
uv sync --extra plugins --extra controller --extra eval --extra grpc \
  --extra runtime --extra utils --extra physics --extra driver \
  --extra wizard --extra trafficsim --extra transfuser
```

Only skip this if you don't need the local manual-driver UI
([MANUAL_DRIVER.md](docs/MANUAL_DRIVER.md)).

Don't re-run `source ./setup_local_env.sh` afterward — it always re-triggers
this failure via its hardcoded `--extra all`. Use `uv run <cmd>` going
forward; the venv at `.venv/` is created automatically by `uv sync`, no
manual activation needed.

## Running the tutorial (`uv run alpasim_wizard ...`)

### `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`

Cause: user isn't in the `docker` group.

Fix:

```bash
sudo usermod -aG docker $USER
```

Then start a **fresh login session** (log out/in, or reconnect over SSH) —
`newgrp docker` was unreliable (prompted for a password). Verify with
`docker ps` (should work without `sudo`).

### `pull access denied for alpasim-base, repository does not exist or may require 'docker login'`

Cause: `alpasim-base` is never published to a registry. Per
`src/wizard/configs/base_config.yaml`:

```yaml
base_image: "${defines.image_registry}alpasim-base:${repo-base-image-tag:}"
```

with `image_registry=""`, so the image must already exist in the local
Docker image store, tagged to match `_read_base_image_tag()`
(`src/wizard/alpasim_wizard/setup_omegaconf.py`) — the repo's `pyproject.toml`
version (e.g. `0.134.0`) when on `main` with no uncommitted `pyproject.toml`
diff.

Fix: get the exact tag the wizard expects, then build it:

```bash
uv run python -c "from alpasim_wizard.setup_omegaconf import _read_base_image_tag; print(_read_base_image_tag())"
docker build -t alpasim-base:<tag_from_above> .
```

The root `Dockerfile` auto-detects architecture: aarch64 pulls
`nvcr.io/nvidia/pytorch:25.08-py3` as its base (x86_64 uses `nvidia/cuda`), so
run `docker login nvcr.io` first if that image is gated for your account.

Rebuild/re-tag `alpasim-base` any time the resolved tag changes (e.g. after
bumping the `pyproject.toml` version or switching branches).

### `E: Unable to locate package datacenter-gpu-manager-4-cuda12` during `docker build`

Cause: the amd64 base image (`nvidia/cuda:12.4.1-cudnn-devel-ubuntu22.04`)
ships with NVIDIA's CUDA apt repo pre-configured, so `datacenter-gpu-manager-4-cuda12`
resolves there. The arm64 base (`nvcr.io/nvidia/pytorch:25.08-py3`, Ubuntu
24.04 "noble") does not have that repo configured — only the stock Ubuntu
ports repos — so the package can't be found.

Fix: add NVIDIA's `cuda-keyring` package (which adds the CUDA apt repo) before
the main `apt-get install`, gated to the arm64 build only. Already patched
into this repo's root `Dockerfile` (search for `cuda-keyring`) — if your
checkout predates that fix, either `git pull` if it shares the same remote/
branch, or apply the same edit by hand: insert, right after the
`ARG DEBIAN_FRONTEND=noninteractive` line and before the main `apt-get install`
block:

```dockerfile
ARG TARGETARCH

RUN if [ "$TARGETARCH" = "arm64" ]; then \
        apt-get update && apt-get install -y --no-install-recommends ca-certificates curl \
        && curl -fsSL -o /tmp/cuda-keyring.deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/sbsa/cuda-keyring_1.1-1_all.deb \
        && dpkg -i /tmp/cuda-keyring.deb \
        && rm -f /tmp/cuda-keyring.deb; \
    fi
```

Note: if you have multiple checkouts of this repo on different machines (e.g.
editing on a Windows dev box, building on a separate Linux/aarch64 host),
edits made on one don't appear on the other unless you `git pull`/push
through a shared remote or copy the file over — check `git remote -v` on
both sides before assuming a fix "took."
