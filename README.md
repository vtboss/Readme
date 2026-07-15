# Readme

# EDA Tools Setup (Windows + WSL2 + Ubuntu 24.04)

## Prerequisites

- Windows 11
- WSL2 installed
- Ubuntu 24.04 installed
- Docker Desktop installed
- Docker WSL Integration enabled

---

## 1. Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 2. Install Required Packages

```bash
sudo apt install git dos2unix curl wget build-essential python3-pip -y
```

---

## 3. Install Latest CMake

Ubuntu's default CMake is too old.

```bash
pip install --user cmake
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Verify:

```bash
cmake --version
```

---

## 4. Clone Repository

```bash
git clone https://github.com/NielitCoeNoida/EDA-Tools-Installer.git
cd EDA-Tools-Installer
```

If GitHub gives:

```
Recv failure: Connection reset by peer
```

just run the clone command again.

---

## 5. Convert Scripts

```bash
dos2unix *.sh
chmod +x *.sh
```

---

## 6. Install OpenLane

```bash
./openlaneinstall.sh
```

Wait until it finishes.

---

## 7. Install Sky130 PDK

```bash
./pdk.sh
```

Wait until it finishes.

---

## 8. Remove Old PDK (Only if Instructor Says)

```bash
sudo rm -rf /labroot/pdk
```

---

## 9. Run Final Installer

```bash
./installfinalv2.sh
```

---

## If You Get

```
fatal: destination path 'verilator' already exists
```

Run

```bash
rm -rf ~/EDA-Tools-Installer/verilator
```

Then run

```bash
./installfinalv2.sh
```

again.

---

## Verify Installation

```bash
verilator --version
yosys -V
magic --version
klayout -v
```

---

## Open OpenLane

```bash
openlane
```

Inside the container verify

```bash
openroad -version
sta -version
```

---

## Test Everything

Inside the container

```bash
flow.tcl -design spm
```

If installation is successful you'll see

```
[SUCCESS]: Flow complete.
```

---

## Exit OpenLane Container

```bash
exit
```

or press

```
Ctrl + D
```

---

## Notes

- `openroad` and `sta` work **only inside the OpenLane container**.
- Keep Docker Desktop running.
- Do not close the terminal while installation is running.
