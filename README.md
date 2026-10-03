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
| `sev.type`    | String  | `none`  | Y        | Guest type (`sev`, `es`, `snp`, `none`)  |
| `sev.props`   | Table   | `{}`    | Y        | Properties of the guest object           |
| `sudo`        | Boolean |         | Y        | Start guest with `sudo`                  |
| `extras`      | Array   | `[]`    | Y        | Extras to be added as-is to command line |

`sudo` defaults to `true` when `sev.type` is set and `false` otherwise.

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

Confidential guests are selected with `sev.type`. Everything under `sev.props`
is passed to the QEMU object, so any property that the configured QEMU build
understands can be set there.

```
[foo.mods.sev]
sev.type = "sev"

[foo.mods.sev-es]
sev.type = "es"

[foo.mods.sev-snp]
sev.type = "snp"
sev.props.kernel-hashes = true

[foo.mods.sev-snp-debug]
sev.type = "snp"
sev.props.policy = 0xB0000
```

`cbitpos` and `reduced-phys-bits` are read from the host, so they do not have
to be set. Setting either one under `sev.props` overrides the host value.

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
