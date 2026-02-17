# wee

A _wee_ tool to launch basic x86-64 KVM guests meant for development use.

## Configuration

TOML is used as the file format for configuration. Guests are defined in
`~/.config/wee/guests.toml` as tables. Each guest uses the following keys as
options.

| Name          | Type    | Default | Optional | Description                              |
|---------------|---------|---------|----------|------------------------------------------|
| `cpu.model`   | String  | `host`  | Y        | CPU model                                |
| `cpu.flags`   | Array   | `[]`    | Y        | CPU flags                                |
| `smp`         | Integer |         | N        | CPU count                                |
| `mem.size`    | Integer |         | N        | RAM size                                 |
| `mem.unit`    | String  | `G`     | Y        | RAM size unit (`M` for MB or `G` for GB) |
| `bios`        | String  |         | Y        | BIOS path                                |
| `disk`        | String  |         | N        | Disk path                                |
| `kernel`      | String  |         | Y        | Kernel path or URL (use PXE boot kernel) |
| `initrd`      | String  |         | Y        | Initrd path or URL (use PXE boot initrd) |
| `append`      | String  |         | Y        | Kernel command line                      |
| `qemu`        | String  |         | Y        | QEMU path                                |
| `sudo`        | Boolean | `false` | Y        | Start guest with `sudo`                  |

## Usage

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
wee foo
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
wee --mods no-pmu foo
wee --mods install-fedora,small foo
wee --mods no-pmu,large foo
```
