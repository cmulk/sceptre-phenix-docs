# Bridge Mode

phēnix uses [Open vSwitch (OVS)](https://www.openvswitch.org/) bridges managed by
minimega to connect VMs to virtual networks. The **bridge mode** setting controls
how phēnix names the default OVS bridge assigned to each experiment.

## Modes

### `manual` (Default)

In `manual` mode, the bridge name for an experiment comes from the
`default-bridge` value specified in the experiment's topology configuration.
If no bridge name is provided, phēnix falls back to the shared default bridge
named **`phenix`**.

This means that unless you explicitly assign a unique bridge name when creating
an experiment, all experiments using `manual` mode that have no custom bridge
name configured will share the same `phenix` bridge.

### `auto`

In `auto` mode, phēnix automatically assigns each experiment's own name as its
default bridge name. This guarantees that every experiment gets a unique,
isolated bridge without any manual configuration.

!!! warning
    Because OVS bridge names are limited to **15 characters**, experiment names
    must be 15 characters or fewer when using `auto` bridge mode. phēnix will
    return an error if you attempt to create or update an experiment with a
    longer name while `auto` mode is active.

## Why Bridge Mode Matters

Giving each experiment a unique bridge name has two important benefits:

1. **Network isolation** — Traffic from one experiment cannot leak onto
   another experiment's virtual network, even if both are running on the same
   host.

2. **Netflow capture** — phēnix's [netflow](netflow.md) feature only works for
   experiments that have a bridge name other than the default `phenix` bridge.
   Using `auto` mode (or manually specifying a unique bridge name per
   experiment) is therefore a prerequisite for per-experiment netflow capture.

## Configuring Bridge Mode

Bridge mode is a **global** setting applied at the phēnix server level. It can
be set in three ways, listed from highest to lowest precedence:

### 1. Command Line Flag

Pass `--bridge-mode` to any phēnix subcommand:

```bash
phenix ui --bridge-mode auto
```

```bash
phenix exp start my-experiment --bridge-mode auto
```

### 2. Configuration File

Add `bridge-mode` to the phēnix configuration file (see
[Settings & Configuration](settings.md) for file locations):

```yaml
bridge-mode: auto
```

### 3. Environment Variable

```bash
export PHENIX_BRIDGE_MODE=auto
phenix ui
```

## Server vs. Client Interaction

When phēnix is run as a UI server, the bridge mode setting is published to
connected CLI clients via the internal `/api/v1/options` endpoint. A CLI client
that connects to the server via the Unix socket will adopt the server's bridge
mode with one exception: if `auto` mode is set on the server, the client will
always honour it, even if `manual` was passed locally. This ensures the server's
isolation policy is consistently enforced.

## Example Workflow

The following shows how to start the phēnix UI server in `auto` mode so that
all experiments automatically receive unique bridges:

```bash
phenix ui --bridge-mode auto
```

With the server running in `auto` mode, creating a new experiment requires the
name to be 15 characters or fewer:

```bash
# Works — name is within the 15-character limit
phenix experiment create my-exp -t my-topology

# Fails — name is too long for auto bridge mode
phenix experiment create my-very-long-experiment-name -t my-topology
```

## Relationship to Netflow

The [netflow](netflow.md) feature captures network flows from the OVS bridge
used by an experiment, but it explicitly excludes the shared default `phenix`
bridge to prevent flows from multiple experiments being mixed together.

Setting bridge mode to `auto` is the simplest way to satisfy this requirement
across all experiments. Alternatively, in `manual` mode you can specify a
unique `--default-bridge` name per experiment at creation time:

```bash
phenix experiment create my-exp -t my-topology --default-bridge my-exp-br
```

See the [Netflow](netflow.md) page for more details on enabling and using
netflow capture.

## Settings Reference

| Setting Key | Environment Variable | Default | Description |
| :--- | :--- | :--- | :--- |
| `bridge-mode` | `PHENIX_BRIDGE_MODE` | `manual` | Bridge naming mode for experiments. `auto` uses the experiment name as the bridge name; `manual` uses the user-specified bridge name, or `phenix` if not specified. |
