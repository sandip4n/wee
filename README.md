# wee

A _wee_ tool to launch basic x86-64 KVM guests meant for development use.

## Configuration

TOML is used as the file format for configuration. Guests are defined in
`~/.config/wee/guests.toml` as tables. Each guest uses the following keys as
options.

| Name            | Type    | Default | Optional | Description                              |
|-----------------|---------|---------|----------|------------------------------------------|
| `cpu.model`     | String  | `host`  | Y        | CPU model                                |
| `cpu.flags`     | Array   | `[]`    | Y        | CPU flags                                |
| `smp`           | Integer |         | N        | CPU count                                |
| `mem.size`      | Integer |         | N        | RAM size                                 |
| `mem.unit`      | String  | `G`     | Y        | RAM size unit (`M` for MB or `G` for GB) |
| `bios`          | String  |         | Y        | Firmware image                           |
| `bios.code`     | String  |         | Y        | Firmware code (with `bios.vars`)         |
| `bios.vars`     | String  |         | Y        | Firmware variables of the guest          |
| `disk`          | String  |         | N        | Disk path                                |
| `dev.blk`       | String  |         | Y        | Disk connection (`virtio` or `ahci`)     |
| `dev.net`       | String  |         | Y        | Network card (`virtio` or `e1000e`)      |
| `kernel`        | String  |         | Y        | Kernel path or URL (use PXE boot kernel) |
| `initrd`        | String  |         | Y        | Initrd path or URL (use PXE boot initrd) |
| `append`        | String  |         | Y        | Kernel command line                      |
| `qemu`          | String  |         | Y        | QEMU path                                |
| `sev.type`      | String  | `none`  | Y        | Guest type (`sev`, `es`, `snp`, `none`)  |
| `sev.props`     | Table   | `{}`    | Y        | Properties of the guest object           |
| `sudo`          | Boolean |         | Y        | Start guest with `sudo`                  |
| `conn.vnc`      | Boolean | `false` | Y        | Enable VNC display                       |
| `conn.port.vnc` | Integer | `5900`  | Y        | VNC port (`5900` or above)               |
| `conn.ssh`      | Boolean | `false` | Y        | Enable SSH forwarding                    |
| `conn.port.ssh` | Integer | `2222`  | Y        | SSH port                                 |
| `extras`        | Array   | `[]`    | Y        | Extras to be added as-is to command line |

`sudo` defaults to `true` when `sev.type` is set and `false` otherwise.

Guests run on QEMU's `q35` machine. A guest that needs the older `pc` machine
can select it with `extras = [ "-machine", "pc" ]`.

## Usage

The following commands are available.

| Command            | Description                     |
|--------------------|---------------------------------|
| `list`             | List guests                     |
| `mods GUEST`       | List mods of a guest            |
| `edit`             | Edit guests                     |
| `exec GUEST`       | Start a guest                   |

Running `wee` with no command shows the same help as `wee --help`.

An example of a simple guest is shown below.

```
[foo]
cpu.model = "host"
cpu.flags = [ "+pmu" ]
smp = 8
mem.size = 8
mem.unit = "G"
bios = "/usr/share/qemu/OVMF.fd"
disk = "~/foo-disk.qcow2"
```

Once a guest is defined, it can be launched as shown below.

```
wee exec foo
```

Each guest can also have a set of mods. They contain overrides for some of the
options in the base configuration. Multiple mods can be stacked on top of the
base configuration. This is done in the same order in which they are specified.
Hence, in case of multiple mods with overlapping changes to options, the final
value of an option is determined by the last mod that overrides it. An example
of a mod for installing Fedora 43 over the network is shown below.

```
[foo.mods.install-fedora]
kernel = "https://download.fedoraproject.org/pub/fedora/linux/releases/43/Server/x86_64/os/images/pxeboot/vmlinuz"
initrd = "https://download.fedoraproject.org/pub/fedora/linux/releases/43/Server/x86_64/os/images/pxeboot/initrd.img"
append = "inst.stage2=https://download.fedoraproject.org/pub/fedora/linux/releases/43/Server/x86_64/os/ ip=dhcp console=ttyS0,115200"
```

Mods are primarily meant to be used as a means for generating different test
configurations with slight changes between them as shown below.

```
[foo.mods.no-pmu]
cpu.flags = [ "-pmu" ]

[foo.mods.small]
smp = 2
mem.size = 2

[foo.mods.large]
smp = 64
mem.size = 64
```

A guest can be launched with one or more mods as shown below.

```
wee exec --mods no-pmu foo
wee exec --mods install-fedora,small foo
wee exec --mods no-pmu,large foo
```

Unless `dev.blk` and `dev.net` are set, QEMU decides how the disk is connected
and which network card the guest gets. Setting them picks one explicitly:
virtio is fast but needs drivers in the guest, while AHCI and e1000e are
supported by nearly every system.

```
[foo]
dev.blk = "ahci"
dev.net = "e1000e"
```

Firmware can be a single image, as in the example above, or split into code
and variables. Split firmware keeps the changes made to its variables, such as
boot entries, across restarts. The variables are written to, so each guest
needs its own copy of the variables template.

```
[foo]
bios.code = "/usr/share/OVMF/OVMF_CODE_4M.fd"
bios.vars = "~/foo_VARS.fd"
```

```
cp /usr/share/OVMF/OVMF_VARS_4M.fd ~/foo_VARS.fd
```

Confidential guests are selected with `sev.type`. Everything under `sev.props`
is passed to the QEMU object, so any property that the configured QEMU build
understands can be set there.

```
[foo.mods.sev]
sev.type = "sev"

[foo.mods.sev-es]
sev.type = "es"
bios = "/usr/share/ovmf/OVMF.amdsev.fd"

[foo.mods.sev-snp]
sev.type = "snp"
bios = "/usr/share/ovmf/OVMF.amdsev.fd"
sev.props.kernel-hashes = true

[foo.mods.sev-snp-debug]
sev.type = "snp"
bios = "/usr/share/ovmf/OVMF.amdsev.fd"
sev.props.policy = 0xB0000
```

`cbitpos` and `reduced-phys-bits` are read from the host, so they do not have
to be set. Setting either one under `sev.props` overrides the host value.

SEV-ES and SEV-SNP guests need firmware as a single image, since they cannot
use split firmware. `OVMF.amdsev.fd` is built for them, and it is the image
that enforces `kernel-hashes` rather than only measuring them.

A guest can be given a display reachable over VNC. It listens on the local
host only, and the serial console stays in the terminal.

```
[foo.mods.vnc]
conn.vnc = true
conn.port.vnc = 5903
```

```
vncviewer localhost:5903
```

A port on the local host can be forwarded to the SSH server in the guest.

```
[foo.mods.ssh]
conn.ssh = true
conn.port.ssh = 2222
```

```
ssh -p 2222 user@localhost
```

The defined guests can be listed, as can the mods of a guest along with the
options each one overrides.

```
wee list
wee mods foo
```

The definitions can be opened in the editor named by `VISUAL` or `EDITOR`.

```
wee edit
```

The command line for a guest can be shown without starting it.

```
wee exec --comm --mods no-pmu foo
```

More examples can be found [here](examples/)
