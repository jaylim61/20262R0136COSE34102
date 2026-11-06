# Project 0: Setting Linux Environment and Compiling Linux Kernel (WSL2)

---
### Custom Kernel Compilation
execute the following command to compile a custom kernel:
```bash
make KCONFIG_CONFIG_Microsoft/config-wsl -j [number of CPUs]
```
- run `lscpu` to get the number of CPUs
- After compilation, `vmlinux` (kernel image) is created under `~/WSL2-Linux_Kernel`
- Copy `vmlinux` to the user folder on the host OS
```bash
cp vmlinux /mnt/c/Users/[host os username]/[file name]
```

- Move to `/mnt/c/Users/[host os username]` and modify `.wslconfig`
```wslconfig
[wsl2]
kernel=C:\\Users\\[host os username]\\[kernel image]
```
- Shut down WSL and reboot host OS
```powershell
PS C:\Windows\system32> wsl --shutdown
```
 - Check kernel version
 ```bash
 uname -r
 ```
 