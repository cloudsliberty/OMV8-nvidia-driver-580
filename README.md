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


---------------------------------------------------------------
> **Method 2**
---------------------------------------------------------------

Maxwell, Pascal and Volta GPUs on Trixie (D-13)
🔗

580 LTS series of the driver required for Maxwell/Pascal/Volta support (see the details above), is not available in the D-13 repo, but several users reported, that drivers from the Debian-12 repo generally worked well with Debian-13 as well. This section covers all the work-arounds needed for such installation.

Note: so far this method has been tested only on amd64 CPU arch (Intel/AMD), hence that's what the below instructions assume.


Using drivers originally intended for a different release is at least a bit of a dirty hack and borderline "FrankenDebian".

Proceed at your own risk.

MAKE SURE that you've completed mandatory steps described in the prerequisites section.

Since 2026-02-01, Debian-13 rejects signatures using SHA1 used in the NV D-12 repo. Additionally, D-13 migrated signature verification tool from gpgv to sqv that does not accept additional options from apt sources. To workaround it all, you need to:

Revert back to using gpgv by adding the below config option somewhere in /etc/apt/apt.conf.d/ folder, for example in /etc/apt/apt.conf.d/10gpgv file:

APT::Key::GPGVCommand "/usr/bin/gpgv";
Configure apt sources with relaxed verification policy: add the below Nvidia D-12 repo somewhere in /etc/apt/sources.list.d/ folder, for example in /etc/apt/sources.list.d/nvidia-d12.list file:

deb [signed-by=/etc/apt/keyrings/nvidia-d12-archive.key allow-insecure=yes] http://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/ /
Download and extract the original Nvidia D-12 repo definition also containing the key:

$ wget -O /tmp/nv-d12-repo-config.deb https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/cuda-keyring_1.1-1_all.deb
$ dpkg -x /tmp/nv-d12-repo-config.deb /tmp/nv-d12-repo-config
Copy the Nvidia key to the location pointed in the repo definition from the 2nd point, update apt's DB and upgrade the system as the Nvidia repo provides newer versions of some system utilities (most notably DKMS) :

# cp /tmp/nv-d12-repo-config/usr/share/keyrings/cuda-archive-keyring.gpg  /etc/apt/keyrings/nvidia-d12-archive.key
# apt update
# apt dist-upgrade
Install nvidia-driver-pinning-580 package to stay on v580 series of the driver and update apt DB again:

# apt install nvidia-driver-pinning-580
# apt update
Install cuda-drivers meta-package:

# apt install cuda-drivers
DKMS will build the proprietary modules for your system.

Proceed to the post-installation steps.


If you also install CUDA toolkit to develop/build CUDA apps, depending on the toolkit version, you may need to apply a few tiny patches to one of the header files as described in this thread and this thread. Versions >= 13.2 should not require patching anymore.



Post-installation steps for NV DC drivers
🔗

Optionally verify the result of the DKMS module compilation:

# dkms status 'nvidia*'
This should result in 1 or more lines (1 per each installed kernel version) similar to the below: (the exact versions may differ)

nvidia/550.163.01, 6.12.57+deb13-amd64, x86_64: installed
Anything other than installed at the end means there was some problem: see Troubleshooting section in such case.
source: https://wiki.debian.org/NvidiaGraphicsDrivers?pow_referer=https%3A%2F%2Fforums.developer.nvidia.com%2F#data-center-d13





To get GPU access working inside Docker containers on your system (such as an OpenMediaVault / OMV box), follow these clean, step-by-step procedures to install the NVIDIA Container Toolkit separately and configure the Docker daemon.

1
Set up the Repository and GPG Key
2 min
Add the official NVIDIA repository and its GPG key to your system's package manager securely. Run the following command in your terminal:

Bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
Verification: Check that the keyring file was successfully created at /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg.

2
Add the Package Repository Source
1 min
Configure the repository list with proper signature verification by running:

Bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
Verification: Ensure /etc/apt/sources.list.d/nvidia-container-toolkit.list exists and contains the correct repository URL with the signed-by keyring reference.

3
Update Package Lists and Install Toolkit
3 min
Refresh your local package index and install the toolkit package.

Note: As mentioned, avoid running a blind apt upgrade on NVIDIA-related packages if your base driver was installed outside of apt, to prevent version conflicts.

Run the update and install commands:

Bash
sudo apt update
sudo apt install -y nvidia-container-toolkit
Verification: Run nvidia-ctk --version to confirm the toolkit binary is correctly installed and accessible.

4
Configure the Docker Daemon
2 min
Configure Docker to recognize the NVIDIA runtime by running the automatic configuration command provided by the toolkit:

Bash
sudo nvidia-ctk runtime configure --runtime=docker
This updates your /etc/docker/daemon.json file to include the nvidia runtime entry.
Verification: Inspect /etc/docker/daemon.json using cat /etc/docker/daemon.json and verify that "runtimes" includes the nvidia handler.

5
Restart the Docker Service
1 min
Apply the configuration changes by restarting the Docker daemon:

Bash
sudo systemctl restart docker
Verification: Run sudo systemctl status docker to ensure the service restarted successfully without errors.

Test Your Setup
To verify that your containers can successfully access your GPU, run a test container using the official CUDA image:

Bash
sudo docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi
If everything is configured correctly, this command will print out your GPU details and driver version table from inside the test container.
