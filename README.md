Installing NVIDIA 580 Driver on OMV 8 / Debian 13 (GTX 1060)
Date: August 29, 2026
System: OpenMediaVault 8, Debian 13 (Trixie), GeForce GTX 1060
Goal: Install NVIDIA driver 580.x instead of 590+ (590+ dropped support for
several older/compute functions needed for this GPU).
Why this was tricky
Debian 13's own non-free repo only ships driver 550.163.01 — too old.
NVIDIA's official CUDA repo for Debian 13 only offers 590/595+ — 580
isn't published there because it doesn't build cleanly against Trixie's
newer glibc.
Debian 12's CUDA repo does have 580, but its packages are signed with
SHA-1, which Debian 13 has rejected since Feb 1, 2026. Various
trusted=yes / allow-insecure=yes workarounds were attempted but apt's
signature verification kept resurfacing.
Final working method: skip apt entirely and use NVIDIA's official
.run installer, built with DKMS so it survives kernel updates.
What was done (final working steps)
Removed the problematic APT repo for Debian 12 CUDA packages —
abandoned the apt-based approach.
Purged old NVIDIA packages:
Code
Installed build prerequisites:
Code
Blacklisted the nouveau driver:
Code
Downloaded and ran the official .run installer:
Code
--dkms registers the module with DKMS so it rebuilds automatically on
future kernel upgrades.
--no-x-check / --no-nouveau-check skip desktop/X.Org checks, which
don't apply on a headless server.
A warning about missing 32-bit compatibility libraries appeared and was
safely ignored (not needed for headless compute/transcode use).
Rebooted and verified:
Code
Confirmed driver 580.173.02 loaded correctly against the GTX 1060.
Maintenance notes for the future
Because the module was installed with --dkms, it should automatically
rebuild after kernel updates. After any kernel upgrade, it's worth
double-checking with:
Code
If a future kernel update breaks the DKMS build (e.g. incompatible kernel
version), you may need to re-run the .run installer against the new
kernel headers, or reconfigure with:
Code
If GPU passthrough into Docker containers (e.g. for Jellyfin/Plex hardware
transcoding) is needed next, install nvidia-container-toolkit separately
and configure the Docker daemon to use the nvidia runtime.
Avoid running apt upgrade blindly on NVIDIA-related packages — since the
driver was installed outside apt, apt has no record of it and won't
interfere, but keep an eye out in case a future kernel package pulls in
nouveau again.
