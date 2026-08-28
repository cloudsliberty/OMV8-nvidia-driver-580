# Installing NVIDIA 580 Driver on OMV 8 / Debian 13 (GTX 1060)

**Date:** August 29, 2026
**System:** OpenMediaVault 8, Debian 13 (Trixie), GeForce GTX 1060
**Goal:** Install NVIDIA driver 580.x instead of 590+ (590+ dropped support for
several older/compute functions needed for this GPU).

## Why this was tricky

- Debian 13's own `non-free` repo only ships driver **550.163.01** — too old.
- NVIDIA's official CUDA repo for Debian 13 only offers **590/595+** — 580
  isn't published there because it doesn't build cleanly against Trixie's
  newer glibc.
- Debian 12's CUDA repo *does* have 580, but its packages are signed with
  SHA-1, which Debian 13 has rejected since Feb 1, 2026. Various
  `trusted=yes` / `allow-insecure=yes` workarounds were attempted but apt's
  signature verification kept resurfacing.
- **Final working method:** skip apt entirely and use NVIDIA's official
  `.run` installer, built with DKMS so it survives kernel updates.

## What was done (final working steps)

1. **Removed the problematic APT repo** for Debian 12 CUDA packages —
   abandoned the apt-based approach.

2. **Purged old NVIDIA packages:**
   ```
   sudo apt purge -y 'nvidia-*' 'libnvidia-*' 'firmware-nvidia-*'
   sudo dkms status   # confirmed empty
   ```

3. **Installed build prerequisites:**
   ```
   sudo apt install -y build-essential dkms linux-headers-$(uname -r) pkg-config libglvnd-dev
   ```

4. **Blacklisted the nouveau driver:**
   ```
   echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
   sudo update-initramfs -u
   ```

5. **Downloaded and ran the official `.run` installer:**
   ```
   cd /tmp
   wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.173.02/NVIDIA-Linux-x86_64-580.173.02.run
   chmod +x NVIDIA-Linux-x86_64-580.173.02.run
   sudo ./NVIDIA-Linux-x86_64-580.173.02.run --dkms --no-x-check --no-nouveau-check --silent
   ```
   - `--dkms` registers the module with DKMS so it rebuilds automatically on
     future kernel upgrades.
   - `--no-x-check` / `--no-nouveau-check` skip desktop/X.Org checks, which
     don't apply on a headless server.
   - A warning about missing 32-bit compatibility libraries appeared and was
     safely ignored (not needed for headless compute/transcode use).

6. **Rebooted and verified:**
   ```
   sudo reboot
   nvidia-smi
   ```
   Confirmed driver **580.173.02** loaded correctly against the GTX 1060.

## Maintenance notes for the future

- Because the module was installed with `--dkms`, it should automatically
  rebuild after kernel updates. After any kernel upgrade, it's worth
  double-checking with:
  ```
  sudo dkms status
  nvidia-smi
  ```
- If a future kernel update breaks the DKMS build (e.g. incompatible kernel
  version), you may need to re-run the `.run` installer against the new
  kernel headers, or reconfigure with:
  ```
  sudo dkms autoinstall
  ```
- If GPU passthrough into Docker containers (e.g. for Jellyfin/Plex hardware
  transcoding) is needed next, install `nvidia-container-toolkit` separately
  and configure the Docker daemon to use the `nvidia` runtime.
- Avoid running `apt upgrade` blindly on NVIDIA-related packages — since the
  driver was installed outside apt, apt has no record of it and won't
  interfere, but keep an eye out in case a future kernel package pulls in
  nouveau again.


Here's how to get GPU access working inside Docker containers on your OMV box.
1. Install the NVIDIA Container Toolkit
Code curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update
sudo apt install -y nvidia-container-toolkit

3. Configure Docker to use the NVIDIA runtime
Code
This adds an nvidia runtime entry to /etc/docker/daemon.json.
4. Verify Docker can see the GPU
Code
If this prints your GTX 1060, Docker-level GPU passthrough is working.
5. Add GPU access to Emby
If you're managing it via docker run or a compose file, add:
Yaml
Then inside Emby's web UI: Settings → Transcoding → set Hardware acceleration to NVIDIA NVENC, and enable it for the codecs you need (H.264/HEVC decode+encode).
If you're using OMV-Extras' Docker/Compose GUI or Portainer instead of raw compose files, the equivalent is: add --runtime=nvidia (or select "nvidia" under GPU/runtime options if the GUI exposes it) plus those two environment variables.
6. Add GPU access to Nextcloud Memories
Memories uses go-vod for video transcoding, which also needs NVENC. In your Nextcloud (or dedicated go-vod) container:
Yaml
Then in Nextcloud: Memories app settings → Transcoding → enable hardware acceleration and select NVIDIA/NVENC. If you're running go-vod as a separate sidecar container (common in the Memories docs), apply the same runtime: nvidia + env vars to that container instead of/alongside Nextcloud itself.
A note on concurrent transcodes: consumer GPUs like the 1060 are limited by Nvidia's driver to a small number of simultaneous NVENC sessions (historically capped at 2–3 unless patched). If you'll run Emby and Nextcloud Memories transcoding at the same time, you may hit that limit — there's an unofficial patch for the session cap if it becomes an issue, happy to cover that if you run into it.
Want me to help you find/edit your actual docker-compose.yml files for these two, if you're running them via compose?
