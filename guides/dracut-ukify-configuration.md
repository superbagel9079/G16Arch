## Install dracut-ukify

```bash
yay -S dracut-ukify
```

### Configure dracut-ukify

```bash
# Configuration file for dracut-ukify package

# Should dracut-ukify colorize its output?
# Can be auto, true or false
colorize=auto

# Kernel package to be set as default in systemd-boot
# eg. setting this to 'linux' is equivalent of calling
# 'bootctl set-default ENTRY_ID_FOR_LINUX' after each upgrade of corresponding package
#default_kernel_package='linux'

# Add global ukify flags to each invocation
# See '/usr/lib/systemd/ukify --help' for an available flags
# 1. Specify cmdline
#    Alternatively, cmdline can be specified in /etc/kernel/cmdline
#    If no cmdline was found, /proc/cmdline will be used
ukify_global_args+=(--cmdline "root=/dev/sda1 quiet")
# 2. Sign UKI image for use with UEFI Secure Boot
ukify_global_args+=(--secureboot-private-key /usr/share/secureboot/keys/db/db.key --secureboot-certificate /usr/share/secureboot/keys/db/db.pem)
# 3. Add splash image (only BMP supported!)
#ukify_global_args+=(--splash /usr/share/systemd/bootctl/splash-arch.bmp)
# 4. Use systemd-sbsign to sign binaries for secure boot, requires systemd>=257
#    --(no-)sign-kernel required for now, as it forces to skip verify phase, which is not supported by systemd-sbsign
#ukify_global_args+=(--signtool=systemd-sbsign --no-sign-kernel)

# Build variants can be declared here
# ukify_variants is are associative array where the key is variant name and value is dracut options to pass during generation
# Note the "default" key is special - it will be omitted in the resulting image name
# It can be used to create fallback images, for example:
#ukify_variants=(
#  [default]="--hostonly"
#  [fallback]="--no-hostonly"
#)

# Override UKI image path for each variant
# Available variables:
# ${name} - package name (linux, linux-lts, linux-zen, etc)
# ${version} - package version
# ${machine_id} - machine id (taken from /etc/machine-id)
# ${build_id} - build id (taken from /etc/os-release, for ArchLinux it's always 'rolling')
# ${id} - os id (taken from /etc/os-release, for ArchLinux it's always 'arch')
# ${esp} - ESP partition mount path (e.g. /efi or /boot/efi)
# ${boot} - XBOOTLDR partition mount path, or ESP if there is none
# Note: that's not real shell variable expansion, it's just string substitution, so the parentheses are required
# Note 2: unless you're using only one kernel package, you must provide unique paths for each package,
#         so either ${name} or ${version} is strongly recommended to use here
# Note 3: you can use absolute or relative paths here, in the latter case ESP mount path will be prepended automatically.
#ukify_install_path=(
#  [default]='EFI/Linux/linux-${version}-${machine_id}-${build_id}.efi'
#  [fallback]='${boot}/EFI/Linux/linux-${version}-${machine_id}-${build_id}-fallback.efi'
#)

# Override kernel cmdline per variant
# It can be used to define each variant it's own cmdline, for example:
#ukify_cmdline=(
#  [default]='root=/dev/sda1 quiet'
#  [fallback]='root=/dev/sda1'
#)
```
