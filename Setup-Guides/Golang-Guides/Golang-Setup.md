## Installing Go

###  1. Download Go and Install

[https://go.dev/dl/](https://go.dev/dl/)

#### On macOS (with pkg):

```bash
# Run the downloaded .pkg file
# Follow the installation steps
```

##### NOTE
You can also use homebrew to install it using the following command

```bash
brew install go
```

#### On Linux (manual method):

```bash
# Remove any previous Go installation
sudo rm -rf /usr/local/go

# Extract the archive to /usr/local
sudo tar -C /usr/local -xzf go1.xx.x.linux-amd64.tar.gz

# Add Go to PATH (add this to ~/.bashrc or ~/.zshrc)
export PATH=$PATH:/usr/local/go/bin
```

Replace `go1.xx.x.linux-amd64.tar.gz` with the actual filename.

##### NOTE
You can install the linux version using the package manager of the linux distro.

Example: `sudo pacman -S go`

#### On Windows:

* Run the `.msi` installer
* The installer will set up everything automatically, including the `PATH`

---

### 2. Verify

After installing, open a terminal and run:

```bash
go version
```

Expected output:

```bash
go version go1.xx.x linux/amd64
```
