# ComfyUI Docker

Run [ComfyUI](https://github.com/comfyanonymous/ComfyUI) using Docker Compose with support for three hardware configurations:

- **NVIDIA** – CUDA 13.0
- **AMD** – ROCm
- **CPU** – CPU-only PyTorch

Each backend runs in its own Docker Compose profile. All profiles share the same persistent data directory and Docker network configuration.

## Features

- Three hardware profiles: NVIDIA, AMD, and CPU.
- Separate prebuilt images for each backend.
- Static container IP addressing.
- Custom isolated Docker bridge network.
- Persistent models, workflows, custom nodes, and generated images.
- No published host ports.
- Only one backend needs to run at a time.

## Requirements

### Common requirements

- Linux host
- Docker Engine
- Docker Compose v2
- Internet connection for pulling images and downloading models

### NVIDIA

- NVIDIA GPU
- Compatible NVIDIA drivers
- NVIDIA Container Toolkit

Verify GPU availability:

```bash
nvidia-smi
```

### AMD

- AMD GPU supported by the selected ROCm image
- Linux AMDGPU drivers
- ROCm-compatible kernel and GPU configuration
- Access to `/dev/kfd` and `/dev/dri`

Verify GPU devices:

```bash
ls -l /dev/kfd /dev/dri
```

### CPU

No additional hardware requirements.

## Project Structure

```text
comfyui/
├── docker-compose.yml
├── README.md
└── data/
    ├── models/
    │   ├── checkpoints/
    │   ├── clip/
    │   ├── clip_vision/
    │   ├── configs/
    │   ├── controlnet/
    │   ├── diffusion_models/
    │   ├── embeddings/
    │   ├── loras/
    │   ├── upscale_models/
    │   └── vae/
    ├── custom_nodes/
    ├── input/
    ├── output/
    └── user/
```

The `./data` directory is mounted into the container at `/data`.

Your models and generated files remain available when switching between hardware profiles.

## Docker Compose Profiles

| Profile | Docker image | Backend |
|---|---|---|
| `nvidia` | `ghcr.io/kagurazakanyaa/comfyui:cu130-latest` | CUDA 13.0 |
| `amd` | `ghcr.io/kagurazakanyaa/comfyui:rocm-latest` | ROCm |
| `cpu` | `ghcr.io/kagurazakanyaa/comfyui:cpu-latest` | CPU |

## Installation

Clone or create the project directory:

```bash
mkdir -p comfyui
cd comfyui
```

Create the persistent data directory:

```bash
mkdir -p data
```

Place the Docker Compose configuration in:

```text
docker-compose.yml
```

## Running ComfyUI

Choose the profile matching your hardware.

### NVIDIA

```bash
docker compose --profile nvidia up -d
```

### AMD

```bash
docker compose --profile amd up -d
```

### CPU

```bash
docker compose --profile cpu up -d
```

**Important:** Run only one profile at a time. All three services use the same static IP address, `172.30.10.2`.

## Stopping ComfyUI

Stop and remove the running container:

```bash
docker compose down
```

This does not delete your persistent models, custom nodes, workflows, or generated images.

## Switching Hardware Profiles

To switch from NVIDIA to AMD:

```bash
docker compose down
docker compose --profile amd up -d
```

To switch from AMD to CPU:

```bash
docker compose down
docker compose --profile cpu up -d
```

To switch back to NVIDIA:

```bash
docker compose down
docker compose --profile nvidia up -d
```

## Network Configuration

The project uses a custom Docker bridge network.

| Property | Value |
|---|---|
| Network name | `comfyui` |
| Driver | `bridge` |
| Subnet | `172.30.10.0/30` |
| Gateway | `172.30.10.1` |
| Container IP | `172.30.10.2` |
| ComfyUI port | `8188` |

The `/30` subnet provides two usable addresses: the gateway (`172.30.10.1`) and the container (`172.30.10.2`).

All profiles use the same IP because they are intended to run independently.

### Accessing ComfyUI

No host ports are published.

On a Linux Docker host, access the interface using:

```text
http://172.30.10.2:8188
```

Other containers connected to the `comfyui` network can also access the API using:

```text
http://172.30.10.2:8188
```

To connect another container to this network:

```bash
docker network connect comfyui <container_name>
```

## Managing Containers

### View running containers

```bash
docker compose ps
```

### View logs

```bash
docker compose logs -f
```

### View logs for a specific backend

```bash
docker compose logs -f nvidia
```

```bash
docker compose logs -f amd
```

```bash
docker compose logs -f cpu
```

### Restart a backend

```bash
docker compose restart nvidia
```

Replace `nvidia` with `amd` or `cpu` as required.

### Update images

```bash
docker compose --profile nvidia pull
docker compose --profile nvidia up -d
```

Replace the profile with the backend you use.

## GPU Verification

### NVIDIA

Check GPU visibility inside the container:

```bash
docker exec comfyui-nvidia nvidia-smi
```

### AMD

Check GPU device availability:

```bash
docker exec comfyui-amd ls -l /dev/kfd /dev/dri
```

### CPU

Check PyTorch device availability:

```bash
docker exec comfyui-cpu python -c \
"import torch; print('PyTorch:', torch.__version__); print('CUDA:', torch.cuda.is_available())"
```

## Troubleshooting

### Container cannot access the GPU

For NVIDIA, verify the NVIDIA Container Toolkit installation and Docker GPU support.

For AMD, verify:

```bash
ls -l /dev/kfd /dev/dri
```

Also check the `video` and `render` group permissions.

### Address already allocated

Check whether another container is using the configured network IP:

```bash
docker network inspect comfyui
```

Only one ComfyUI backend should run at a time.

### Port or interface inaccessible

Check the container status:

```bash
docker compose ps
```

Check its logs:

```bash
docker compose logs --tail=100
```

Verify that ComfyUI listens on `0.0.0.0:8188` inside the container.

### Models are missing

Verify the persistent directory:

```bash
ls -lah data/models
```

Ensure that model files are stored in the correct ComfyUI model subdirectories.

## Data Persistence

All persistent data is stored under:

```text
./data
```

This includes:

- Checkpoints
- Diffusion models
- LoRAs
- VAE models
- ControlNet models
- Custom nodes
- Input images
- Generated images
- User workflows and settings

Removing containers does not remove the host data directory.

## Useful Commands

| Task | Command |
|---|---|
| Start NVIDIA | `docker compose --profile nvidia up -d` |
| Start AMD | `docker compose --profile amd up -d` |
| Start CPU | `docker compose --profile cpu up -d` |
| Stop | `docker compose down` |
| Logs | `docker compose logs -f` |
| Status | `docker compose ps` |
| Pull images | `docker compose --profile nvidia pull` |
| Inspect network | `docker network inspect comfyui` |

## References

- [ComfyUI GitHub](https://github.com/comfyanonymous/ComfyUI)
- [ComfyUI Linux Installation Guide](https://comfyui-wiki.com/en/install/install-comfyui/install-comfyui-on-linux)
- [Docker Compose Profiles](https://docs.docker.com/compose/how-tos/profiles/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [AMD ROCm Documentation](https://rocm.docs.amd.com/)
