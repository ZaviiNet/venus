## Venus OS: the Victron Energy Unix like distro with a linux kernel

The problematic part with this name is that it is from the Roman
mythology and not, as most of our products, from the Greek. Phoenix
is already taken though by a charger...

This readme documents how to compile and build Venus OS from source.
It also covers the **Home Assistant Add-on** (Docker container) that runs the
key Venus OS services inside Home Assistant OS without compiling anything.

First, make sure that that is really what you want or need. It takes
several hours to compile, lots of diskspace and results in an
image and sdk which are both already available for download as binaries
[swu](http://updates.victronenergy.com/feeds/venus/release/images/) and
[sdk](http://updates.victronenergy.com/feeds/venus/release/sdk/).

Even when you are developing on one of the parts of Venus OS, for example
one of its drivers, or the gui, its still not necessary to build the
full Venus OS from source.

Make sure to read the [Venus OS wiki](https://github.com/victronenergy/venus/wiki)
first.

---

## Home Assistant Add-on (Docker container)

> **New!** You no longer need to build Venus OS from source to use its core
> services in Home Assistant. The `addon/` directory in this repository ships a
> fully-featured HA Add-on that runs inside Docker on any Home Assistant OS
> installation.

### What the add-on provides

| Service | Role |
|---|---|
| **dbus-daemon** | Private D-Bus system bus inside the container |
| **mosquitto** | MQTT broker — HA connects on port **1883** |
| **dbus-systemcalc-py** | Aggregates device values (SOC, power, voltage, …) |
| **dbus-mqtt** | Bridges D-Bus → MQTT with VRM-compatible `N/<portalid>/#` topics |

Home Assistant's built-in **MQTT integration** picks up all Victron entities
automatically via MQTT auto-discovery.

### Supported architectures

`aarch64` · `amd64` · `armhf` · `armv7`

(Raspberry Pi 3/4/5, Intel NUC, ODROID, and most other HA-supported hardware.)

### Installing the add-on in Home Assistant

1. **Add this repository to Home Assistant:**
   - Go to **Settings → Add-ons → Add-on Store**.
   - Click the ⋮ menu (top-right) → **Repositories**.
   - Paste `https://github.com/ZaviiNet/venus` and click **Add**.

2. **Install the "Venus OS" add-on** that now appears in the store.

3. **Configure the add-on** (Settings → Add-ons → Venus OS → Configuration):

   | Option | Description | Default |
   |---|---|---|
   | `vrm_id` | Your 12-char GX device VRM Portal ID (leave blank to auto-generate) | _(auto)_ |
   | `mqtt_username` | MQTT broker username | _(anonymous)_ |
   | `mqtt_password` | MQTT broker password | _(anonymous)_ |
   | `vedirect_port` | Serial device for a VE.Direct adapter, e.g. `/dev/ttyUSB0` | _(disabled)_ |
   | `vedirect_baud` | Baud rate for the VE.Direct port | `19200` |
   | `vecan_interface` | Linux CAN interface, e.g. `can0` | _(disabled)_ |
   | `vrm_token` | VRM Portal API token for cloud connectivity | _(disabled)_ |
   | `modbus_enabled` | Start the Modbus TCP server on port 502 | `false` |
   | `log_level` | Verbosity (`debug`/`info`/`warning`/`error`) | `info` |

4. **Start the add-on.**

5. **Connect the MQTT integration** (if not already set up):
   - Go to **Settings → Devices & Services → Add Integration → MQTT**.
   - Broker: `localhost` (or your HA host IP), Port: `1883`.
   - Supply the same credentials you configured in step 3.
   - Victron entities appear automatically under *Devices* once the add-on is running.

### Hardware wiring

**VE.Direct (USB adapter)**
Plug a [VE.Direct to USB interface](https://www.victronenergy.com/accessories/ve-direct-to-usb-interface)
into the HA host (typically `/dev/ttyUSB0`), then set `vedirect_port: /dev/ttyUSB0`
and enable **UART** access in the add-on's *Devices* tab.

**VE.Can (CAN hat / USB adapter)**
Load the kernel module on the HA OS host (e.g. `mcp251x`, `gs_usb`), verify
`ip link show can0` works, then set `vecan_interface: can0`. The add-on brings
the interface up at 250 kbps automatically.

**VE.Bus / MultiPlus / Quattro**
Connect a [MK3-USB interface](https://www.victronenergy.com/accessories/mk3-usb).
The `mk2dbus` daemon is proprietary and not included; see
[Venus OS Large](https://github.com/victronenergy/venus/wiki/venus-os-large) for
installing it separately.

### Running with plain Docker (without Home Assistant)

The image can also be run standalone for development or testing:

```bash
docker build \
    --build-arg BUILD_FROM=debian:bookworm-slim \
    -t venus-os-addon \
    ./addon

docker run --rm \
    --privileged \
    -v "$(pwd)/data:/data" \
    -p 1883:1883 \
    -e SUPERVISOR_TOKEN="" \
    venus-os-addon
```

> **Note:** `--privileged` is required to allow D-Bus and CAN interface
> operations. In production, use the HA Add-on which applies a scoped AppArmor
> profile instead.

### Add-on internals

```
addon/
├── Dockerfile          # Multi-stage build (builder → HA base image)
├── build.yaml          # Per-arch base images (ghcr.io/home-assistant/…-base-debian:bookworm)
├── config.yaml         # HA add-on manifest (ports, options schema, capabilities)
├── apparmor.txt        # AppArmor security profile
├── DOCS.md             # Full user documentation
├── CHANGELOG.md        # Release history
└── rootfs/
    ├── run.sh                           # Init script: reads options.json, writes runtime config
    └── etc/
        ├── dbus-1/                      # D-Bus policy (permissive for container use)
        └── s6-overlay/s6-rc.d/
            ├── init-addon/              # Oneshot: runs run.sh before services start
            ├── dbus/                    # dbus-daemon longrun
            ├── mosquitto/               # Mosquitto MQTT broker longrun
            ├── dbus-systemcalc-py/      # Venus system calculator longrun
            └── dbus-mqtt/               # D-Bus ↔ MQTT bridge longrun
```

For full configuration details, known limitations, and troubleshooting see
[`addon/DOCS.md`](addon/DOCS.md).

---

## Building Venus OS from source

So, if you insist: this repo is the starting point to build Venus.
It contains wrapper functions around bitbake and git to fetch, and
compile sources.

For a complete build you need to have access to private repros of Victron
Energy. Building only opensource packages is also possible (but not checked
automatically at the moment).

Venus uses [OpenEmbedded](https://www.openembedded.org/) as build system.

### Getting started
Building Venus requires a Linux. At Victron we use Ubuntu for this.

```
# clone this repository
git clone https://github.com/victronenergy/venus.git
cd venus

# install host packages (Debian based)
sudo make prereq

# fetch needed subtrees
# use make fetch-all instead, if you have access to all the private repos.
make fetch
```

That last fetch command has cloned several things into the `./sources/`
directory. First of all there is [bitbake](https://github.com/openembedded/bitbake),
which is a make-like build tool part of OpenEmbedded. Besides that, you'll find
[openembedded-core](https://github.com/openembedded/openembedded-core/) and
various other layers containing recipes and other metadata defining Venus.

Now its time to actually start building (which can take many hours). Select one
of below example commands:
```
# build all, this will take a while though... it builds for all MACHINES as found
# in conf/machines.
make venus-images

# build for a specific machine
make ccgx-venus-image
make beaglebone-venus-image

# build the swu file only
make ccgx-swu

# build from within the bitbake shell.
# this will have the same end result as make ccgx-swu
make ccgx-bb
bitbake venus-swu
```

### Configs
Above Getting Started instructions will automatically select the config that is
used for Venus OS as distributed. Alternative setups can also be used, e.g. to
build for a newer OE version:

    make CONFIG=rocko fetch-all

To see which config your checkout is using, look at the ./conf symlink. It
will link to one of the configs in the ./configs directories.

For each config there are a few files:

- `repos.conf` contains the repositories which need to be checked out. It can be
rebuild with `make update-repos.conf`.
- `metas.whitelist` contains the meta directory which will be added to
bblayers.conf, but only if they are actually present.
- `machines` contains a list of machines that can be build in this config

To add a new repository, put it in sources, then checkout the branch you want and
set an upstream branch. The result can be made permanent with: `make repos.conf`.

Don't forget to add the directories you want to use from the new repository to
metas.whitelist.

### Using the `repos` command
Repos is just like git submodule foreach -q git, but shorter,
so you can do:

./repos push origin
./repos tag xyz

It will push all, tag all etc. Likewise you can revert to a certain
revision with:

./repos checkout tagname

### managing git remotes and branches

```
# patches not in upstream yet
./repos cherry -v

# local changes with respect to upstream
./repos diff @{u}

# local changes with respect to the push branch
./repos diff 'origin/`git rev-parse --abbrev-ref HEAD`'
or if you have git 2.5+ ./repos diff @{push}

./repos log @{u}..upstream/`git rev-parse --abbrev-ref @{u} | grep -o "[a-Z0-9]*$"` --oneline

# rebase your local checkout branches on upstream master
./repos fetch origin
./repos rebase 'origin/$checkout_branch'

# checkout the branches as per used config
./repos checkout '$checkout_branch'
````

### Releasing

```
# tag & push venus repo as well as all repos.

git tag v2.21
git push origin v2.21

./repos tag v2.21
./repos push origin v2.21
```

### Maintenance releases

#### How to create a new maintenance branch
The base branch on which the maintenance releases will be based is to be
prefixed with a `b`.

This example shows how to create a new maintenance branch. The context is that
master is already working on v2.30. Latest official release was v2.20. So we
make a branch named b2.20 in which the first release will be v2.21; later if
another maintenance release is necessary v2.22 is pushed on top; and so forth.

```
# clone & make a branch in the venus repo
git clone git@github.com:victronenergy/venus.git venus-b2.20
cd venus-b2.20
git checkout v2.20
git checkout -b b2.20

# fetch all the meta repos
make fetch-all

# clone, prep and push them
./repos checkout v2.20
./repos checkout -b b2.20
./repos push --set-upstream origin b2.20

# update the used config to the new branch
make update-repos.conf
git commit -a -m "pin dunfell branches to b2.20"

# update the raspbian config to the new branch
[
  Now manually update the raspbian config file, and commit that as well.
  See some earlier branch for example.
]

git commit -a -m "pin raspbian branches to b2.20"

# Update gitlab-ci.yml
[
  Now, modify .gitlab-ci.yml. See a previous maintenance branch for
  how that is done.
]

git commit -a -m "Don't touch SSTATE cache & build from b2.20"

# Push the new branch and changes to the venus repo
# Note that this causes a (useless) CI build to start on the builder once
# it syncs. Easily cancelled in the gitlab ui.

git push --set-upstream origin b2.20

```

Now you're all set; and ready to start cherry-picking.


#### Full cherry-picks vs backporting patches
Be aware that there are two ways to backport a change. One is to take
a complete commit from the meta repositories; and the other one is to
add patches from the source repository. Where you can, apply method
one. But in case the repository, for example mk2-dbus or the gui, has
had lots of commits out of which you need only one; then you have to
take just the patch.


#### The master rule when deciding against- or for inclusion

Changes need to be either really small, well tested or very important


#### The eight golden rules of maintaining maintenance branches

1. only take changes from master: cherry-picking
2. don't add changes or new versions that are not in master yet
3. `git cherry-pick -x` appends a nice (cherry-picked from [ref]) line to the commit message
4. add and/or increase the PR when adding patches
5. drop the PR again when going to a clean version
6. when adding patches; add a `backported from` note just like
   [this one](https://github.com/victronenergy/meta-victronenergy-private/commit/22ac88f61cc6f13cce1d2fc5455248e066e7a835)
   to the commit message
7. go through the [todo](https://github.com/victronenergy/venus-private/wiki/todo)
   where the team is working on master, and add `(**backported to v2.22**)` or where
   applicable `(**backported to v2.22 as a patch**)` to each and every patch and version
   thats been backported.
8. double verify everything by cross referencing the todo, the commits logs from
   master as well as your own commit log.


#### Building a maintenance release

To build, create a pipeline on the mirrors/venus repo, and run it for the
maintenance branch. No variables needed.


### Various notes

#### 1. Linux update
If you encounter problems like this:
 * Solver encountered 1 problem(s):
 * Problem 1/1:
 *   - nothing provides kernel-image-4.14.67 needed by packagegroup-machine-base-1.0-r83.einstein

if can be fixed with:
  make einstein-bb
  bitbake -c cleanall packagegroup-machine-base

and thereafter try again

#### 2. Rust crates
If there are errors about crates missing, cleanall rust recipes:

it can be fixed with:
  bitbake -c cleanall python3-cryptography python3-orjson python3-bcrypt
