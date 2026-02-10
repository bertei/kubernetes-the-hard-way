# Vagrant Commands Reference

**Goal:** Quick reference for common Vagrant commands used in Kubernetes The Hard Way

**⚠️ IMPORTANT:** All `vagrant` commands must be run from the **Vagrantfile directory**:
```powershell
cd D:\Personal-Projects\k8s-hard-way\kubernetes-the-hard-way\vagrant
```

---

## Essential Commands

### Starting & Stopping VMs

#### `vagrant up`
**Starts all VMs defined in Vagrantfile**

```powershell
vagrant up
```

**What it does:**
1. Downloads Ubuntu image (first time only)
2. Creates 5 VMs (controlplane01, controlplane02, loadbalancer, node01, node02)
3. Allocates CPU/RAM based on your system
4. Configures networking (NAT + Host-only)
5. Runs provisioning scripts (kernel setup, DNS, SSH)
6. Sets up port forwarding (localhost:2711 → controlplane01, etc.)

**Output:**
```
Bringing machine 'controlplane01' up with 'virtualbox' provider...
==> controlplane01: Importing base box 'ubuntu/jammy64'...
==> controlplane01: Matching MAC address for NAT networking...
...
```

**Time:** 10-20 minutes (first run), 2-3 minutes (subsequent runs)

**Useful flags:**
```powershell
vagrant up --no-provision        # Start VMs but skip provisioning scripts
vagrant up controlplane01        # Start only one VM
vagrant up controlplane01 node01 # Start specific VMs
```

---

#### `vagrant halt`
**Gracefully shuts down all VMs (like pressing power button)**

```powershell
vagrant halt
```

**What it does:**
- Sends shutdown signal to all running VMs
- Saves VM state
- VMs can be restarted with `vagrant up`

**Output:**
```
==> controlplane01: Attempting graceful shutdown of VM...
==> controlplane02: Attempting graceful shutdown of VM...
...
```

**Useful flags:**
```powershell
vagrant halt controlplane01      # Halt only one VM
vagrant halt --force             # Forcefully shutdown (not graceful)
```

**When to use:**
- Pausing work (VMs saved to disk)
- Low on RAM (VMs not running)
- Ending the day (safer than destroy)

---

#### `vagrant destroy`
**Completely removes VMs and associated files**

```powershell
vagrant destroy
```

**What it does:**
1. Asks for confirmation (unless `-f` used)
2. Stops all running VMs
3. Deletes VM files
4. Unregisters from VirtualBox
5. Clears `.vagrant/` state (but keeps Vagrantfile)

**Output:**
```
==> controlplane01: Destroying VM and associated drives...
==> controlplane02: Destroying VM and associated drives...
...
```

**Useful flags:**
```powershell
vagrant destroy -f               # Force (no confirmation)
vagrant destroy controlplane01   # Destroy only one VM
```

**When to use:**
- Starting fresh (fresh certificates, fresh etcd, etc.)
- Cleaning up old VMs (like you did!)
- Freeing up disk space
- Testing the setup from scratch

**⚠️ WARNING:** This DELETES everything! Make sure you don't need the data.

---

#### `vagrant suspend`
**Pauses VMs (like hibernate)**

```powershell
vagrant suspend
```

**What it does:**
- Saves exact VM state (RAM, processes, everything) to disk
- VMs are frozen, not shut down
- Resume with `vagrant resume`

**Output:**
```
==> controlplane01: Saving VM state and suspending execution...
==> controlplane02: Saving VM state and suspending execution...
```

**Useful flags:**
```powershell
vagrant suspend controlplane01   # Suspend only one VM
```

**When to use:**
- Quick pause (faster than halt + up)
- Keep everything running state intact
- Coming back in 5 minutes

**Difference from `halt`:**
- `halt` = graceful shutdown (cleaner)
- `suspend` = freeze (faster restart, but less clean)

---

#### `vagrant resume`
**Resumes suspended VMs**

```powershell
vagrant resume
```

**What it does:**
- Unfreezes suspended VMs
- Restores exact state before suspension
- All processes continue where they left off

**Output:**
```
==> controlplane01: Resuming suspended execution...
==> controlplane02: Resuming suspended execution...
```

---

### Checking Status

#### `vagrant status`
**Shows status of all VMs in current Vagrantfile**

```powershell
vagrant status
```

**Output:**
```
Current machine states:

controlplane01               running (virtualbox)
controlplane02               running (virtualbox)
loadbalancer                 running (virtualbox)
node01                       running (virtualbox)
node02                       running (virtualbox)

This environment represents multiple machines. The machines are all
based on the same Vagrantfile, and they share dependencies. To interact
with one of them, specify the VM name as an argument, for example:
"vagrant up controlplane01"
```

**Possible states:**
- `running` = VM is on
- `not created` = VM doesn't exist (need `vagrant up`)
- `poweroff` = VM is shut down (halted)
- `saved` = VM is suspended
- `aborted` = Something went wrong

---

#### `vagrant global-status`
**Shows status of ALL Vagrant VMs on your machine (all projects)**

```powershell
vagrant global-status
```

**Output:**
```
id       name           provider   state   directory
------------------------------------------------------------------------------------------------
00b6947  controlplane01 virtualbox running D:/Personal-Projects/k8s-hard-way/.../vagrant
70e3380  controlplane02 virtualbox running D:/Personal-Projects/k8s-hard-way/.../vagrant
e1bcfd2  loadbalancer   virtualbox running D:/Personal-Projects/k8s-hard-way/.../vagrant
6ddb0c2  node01         virtualbox running D:/Personal-Projects/k8s-hard-way/.../vagrant
f9g3h9k  node02         virtualbox running D:/Personal-Projects/k8s-hard-way/.../vagrant

The above shows information about all known Vagrant environments
on this machine. This data is cached and may not be up-to-date. Run
"vagrant global-status --prune" to remove stale entries.
```

**Key columns:**
- `id` = Machine ID (use with `vagrant ssh <id>`)
- `name` = VM name
- `state` = running/poweroff/saved
- `directory` = Where Vagrantfile is

**Useful flags:**
```powershell
vagrant global-status --prune    # Remove stale/deleted VMs from cache
```

**When to use:**
- See all VMs across all projects
- Check what's using RAM/CPU
- Use ID when `vagrant ssh` doesn't work (wrong directory)

---

### SSH & Connectivity

#### `vagrant ssh <vm-name>`
**SSH into a VM without needing password or key path**

```powershell
vagrant ssh controlplane01
vagrant ssh controlplane02
vagrant ssh node01
```

**What it does:**
1. Finds private key in `.vagrant/machines/<vm>/virtualbox/private_key`
2. Uses port forwarding (`localhost:2711` → VM:22)
3. Connects to VM as `vagrant` user
4. No password needed!

**Output (inside VM):**
```
vagrant@controlplane01:~$  # You're now inside the VM!
```

**To exit VM:**
```bash
exit
# Or Ctrl+D
```

**Useful flags:**
```powershell
vagrant ssh controlplane01 -c "hostname"  # Run command inside VM
vagrant ssh controlplane01 -c "ip addr show enp0s8"
vagrant ssh controlplane01 -c "whoami"    # Output: vagrant
```

**If wrong directory:**
```powershell
# If you get "Vagrant environment required" error:
vagrant ssh 00b6947  # Use ID from vagrant global-status
```

---

#### `vagrant ssh-config`
**Shows SSH configuration for all VMs**

```powershell
vagrant ssh-config
```

**Output:**
```
Host controlplane01
  HostName 127.0.0.1
  User vagrant
  Port 2711
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile C:/path/to/.vagrant/machines/controlplane01/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL

Host controlplane02
  HostName 127.0.0.1
  User vagrant
  Port 2712
  ...
```

**What it shows:**
- Hostname/IP for SSH connection
- Port (port forwarding)
- User (vagrant)
- Private key location
- All SSH settings

**When to use:**
- Manually SSH (advanced): `ssh -i <key> vagrant@localhost -p 2711`
- Troubleshoot SSH issues
- Configure SSH clients

---

### Provisioning

#### `vagrant provision`
**Re-runs provisioning scripts WITHOUT destroying VMs**

```powershell
vagrant provision
```

**What it does:**
- Runs all provisioning scripts defined in Vagrantfile
- Useful if provisioning failed earlier
- Keeps VM data intact

**Scripts that run:**
```ruby
# From Vagrantfile:
node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/vagrant/setup-hosts.sh"
node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"
node.vm.provision "setup-kernel", :type => "shell", :path => "ubuntu/setup-kernel.sh"
```

**Useful flags:**
```powershell
vagrant provision controlplane01         # Provision only one VM
vagrant provision --no-parallel          # Run provisioning sequentially
```

**When to use:**
- Script failed during initial `vagrant up`
- Changed provisioning scripts, need to re-run
- Don't want to destroy and recreate VMs

---

### Information & Debugging

#### `vagrant box list`
**Shows all downloaded Vagrant box images**

```powershell
vagrant box list
```

**Output:**
```
ubuntu/jammy64 (virtualbox, 20250101.0.0)
```

**What it shows:**
- Box name: `ubuntu/jammy64` (Ubuntu 22.04 Jammy)
- Provider: `virtualbox`
- Version: `20250101.0.0`

**When to use:**
- Check if image already downloaded (faster `vagrant up`)
- Clean up old images

---

#### `vagrant box remove <box-name>`
**Removes downloaded box image to free disk space**

```powershell
vagrant box remove ubuntu/jammy64
```

**What it does:**
- Deletes the box image
- Next `vagrant up` will re-download (slower)

**⚠️ WARNING:** Only do this if you're NOT using this box anywhere else!

---

### Reload & Update

#### `vagrant reload`
**Restarts VMs (halt + up)**

```powershell
vagrant reload
```

**What it does:**
1. `halt` all VMs
2. Re-reads Vagrantfile
3. `up` all VMs (with provisioning if specified)

**Useful flags:**
```powershell
vagrant reload --provision        # Also re-run provisioning scripts
vagrant reload controlplane01     # Reload only one VM
```

**When to use:**
- Changed Vagrantfile settings
- Need fresh boot
- Network issues (try reload)

---

#### `vagrant reload --provision`
**Restart VMs AND re-run provisioning scripts**

```powershell
vagrant reload --provision
```

**When to use:**
- Changed provisioning scripts
- Provisioning didn't run properly
- Need clean configuration reset

---

### Validation

#### `vagrant validate`
**Checks if Vagrantfile is valid syntax**

```powershell
vagrant validate
```

**Output (if valid):**
```
Vagrantfile validated successfully.
```

**Output (if invalid):**
```
There are errors in the configuration of this machine. Please fix
the following errors and try again:

<errors>
```

**When to use:**
- After editing Vagrantfile
- Before `vagrant up` if having issues

---

## Quick Reference Table

| Command | What It Does | Time | Reversible? |
|---------|--------------|------|-------------|
| `vagrant up` | Create & start all VMs | 10-20 min | Yes (destroy) |
| `vagrant halt` | Graceful shutdown | 1 min | Yes (up) |
| `vagrant destroy` | Delete all VMs | 2 min | Yes (up again) |
| `vagrant suspend` | Freeze VMs | 10 sec | Yes (resume) |
| `vagrant resume` | Unfreeze VMs | 10 sec | Yes (suspend) |
| `vagrant status` | Check VM states | Instant | - |
| `vagrant ssh <vm>` | SSH into VM | Instant | - |
| `vagrant provision` | Re-run setup scripts | 2-5 min | - |
| `vagrant reload` | Restart VMs | 5 min | - |
| `vagrant validate` | Check Vagrantfile | Instant | - |

---

## Common Workflows

### Starting Fresh Setup

```powershell
# You just cloned the repo or want to start over

cd D:\Personal-Projects\k8s-hard-way\kubernetes-the-hard-way\vagrant

# Option 1: Clean start (delete everything)
vagrant destroy -f
vagrant up

# Option 2: Safe start (keep old VMs around just in case)
vagrant up
```

---

### Pausing Work

```powershell
# End of day, want to continue tomorrow

# Option 1: Halt (graceful, safer)
vagrant halt
# Tomorrow: vagrant up

# Option 2: Suspend (faster resume, but less clean)
vagrant suspend
# Tomorrow: vagrant resume
```

---

### Debugging Issues

```powershell
# VMs are acting weird

# Step 1: Check status
vagrant status

# Step 2: Restart
vagrant reload

# Step 3: Re-run provisioning
vagrant reload --provision

# Step 4: Nuclear option
vagrant destroy -f
vagrant up
```

---

### SSH into All VMs

```powershell
# Test connectivity to all VMs

cd D:\Personal-Projects\k8s-hard-way\kubernetes-the-hard-way\vagrant

vagrant ssh controlplane01 -c "hostname"
vagrant ssh controlplane02 -c "hostname"
vagrant ssh node01 -c "hostname"
vagrant ssh node02 -c "hostname"
vagrant ssh loadbalancer -c "hostname"
```

---

### Cleaning Up Everything

```powershell
# Completely remove K8s VMs (but keep Vagrantfile)

cd D:\Personal-Projects\k8s-hard-way\kubernetes-the-hard-way\vagrant

vagrant destroy -f      # Delete all VMs
rm -r .vagrant          # Delete Vagrant state (optional)
vagrant global-status --prune  # Clean up cache

# To restart:
vagrant up
```

---

## Troubleshooting

### "Command not found: vagrant"

**Solution:** Install Vagrant or add to PATH
```powershell
# Check installation:
vagrant --version

# If not found, download from: https://www.vagrantup.com/downloads
```

---

### "A Vagrant environment or target machine is required"

**Solution:** You're not in the right directory!
```powershell
# ❌ Wrong:
cd D:\Personal-Projects\k8s-hard-way
vagrant ssh controlplane01

# ✅ Right:
cd D:\Personal-Projects\k8s-hard-way\kubernetes-the-hard-way\vagrant
vagrant ssh controlplane01

# ✅ Or use machine ID:
vagrant ssh 00b6947
```

---

### "VirtualBox machine with name already exists"

**Solution:** Old VMs are still registered
```powershell
# Use the cleanup script:
powershell -ExecutionPolicy Bypass -File ..\cleanup-vms.ps1

# Or manually:
vagrant destroy -f
rm -r .vagrant
vagrant up
```

---

### VMs won't start (VirtualBox error)

**Solution:** Reload configuration
```powershell
vagrant reload --provision

# If that fails:
vagrant destroy -f
vagrant up
```

---

### Lost SSH connection during provisioning

**Solution:** Re-run provisioning
```powershell
vagrant provision

# Or full reload:
vagrant reload --provision
```

---

## Notes for Future You

- **Always `cd` to vagrant directory first** - This is the #1 mistake
- **`vagrant up` first time is slow** - It downloads the Ubuntu image (~1GB)
- **Port forwarding is set up automatically** - Don't need to do anything
- **`.vagrant/` is local state** - Don't commit to git
- **`vagrant destroy` is safe** - You can always `vagrant up` again
- **SSH is passwordless** - Vagrant handles key setup automatically

---

## When to Use Each Shutdown Command

```
Development:       vagrant halt      (safe, quick)
End of day:        vagrant halt      (saves state)
Testing again:     vagrant destroy   (fresh start)
Quick pause:       vagrant suspend   (fastest resume)
```

---

**Last Updated:** Feb 2026  
**Lab:** 03 (Client Tools)  
**Reference:** Kelsey Hightower's K8s The Hard Way
