# RStudio Open OnDemand App

This repository contains a Tufts Open OnDemand Batch Connect app for launching
RStudio Server on the `pax` Slurm cluster.

The app uses a shared resource discovery partial to build resource controls from
Slurm at form-render time. It is configured as a CPU-only app, so users only see
CPU partitions and CPU-relevant resource widgets.

## Features

- Launch RStudio Server through Open OnDemand.
- Let users choose an R module version from the form.
- Discover user-accessible Slurm partitions dynamically.
- Filter the partition list to CPU-only partitions.
- Set dynamic maximum values for CPU cores, memory, and wall time.
- Avoid caching resource-selection widgets so users make fresh choices for each job.
- Load dynamic JavaScript from the app-local `form.js`.

## App Layout

- `manifest.yml`: Open OnDemand app metadata.
- `form.yml.erb`: Batch Connect form definition and CPU partition filter.
- `/var/www/ood/apps/sys/ood-resource-discovery/resource_discovery.erb`: shared Slurm discovery and generated resource widgets.
- `form.js`: Dynamic browser behavior for partition, CPU, memory, and wall-time fields.
- `submit.yml.erb`: Slurm submit arguments.
- `template/before.sh.erb`: RStudio connection setup.
- `template/script.sh.erb`: Runtime environment setup and RStudio Server launch.
- `template/after.sh.erb`: Cleanup hook.
- `template/bin/auth`: RStudio PAM auth helper.

## CPU Partition Filtering

Partition filtering is controlled in `form.yml.erb` before
the shared site-level `resource_discovery.erb` is evaluated:

```erb
<%
@partition_filter = :cpu

_partial_path = '/var/www/ood/apps/sys/ood-resource-discovery/resource_discovery.erb'
_gpu_result = ERB.new(File.read(_partial_path)).result(binding)
%>
```

Available filters:

- `:cpu`: show only CPU-only partitions.
- `:gpu`: show only partitions that contain GPU nodes.
- `:all`: show all user-accessible partitions.

This RStudio app should use:

```erb
@partition_filter = :cpu
```

The generated helpers use `@partition_filter` by default, so the CPU-only form
can render only the relevant widgets:

```erb
<%= generate_partition_field.indent(2) %>
<%= generate_num_cores_field.indent(2) %>
<%= generate_num_memory_field.indent(2) %>
<%= generate_num_hours_field.indent(2) %>
```

Do not render `generate_gpu_type_field` or `generate_num_gpus_field` in this app
unless you intentionally convert it into a GPU-capable RStudio app.

## Resource Discovery

The app uses Slurm commands at form-render time:

- `sinfo -o "%P %l"` for user-accessible partitions and wall-time limits.
- `scontrol show partition --oneliner` for partition-level CPU and memory caps.
- `scontrol show node --oneliner` for node CPU count, memory, features, GRES, and
  node state.

For CPU-only partitions, the dynamic JavaScript treats the internal GPU type as
`none` and updates CPU cores, memory, and wall time when the selected partition
changes.

## Deployment

Install this directory as a Batch Connect app under the Open OnDemand app
location for your site, then review these site-specific settings:

- `cluster: "pax"` in `form.yml.erb`.
- `@partition_filter = :cpu` in `form.yml.erb`.
- Available R module options in `form.yml.erb`.
- Slurm native arguments in `submit.yml.erb`.
- RStudio launch options and bind paths in `template/script.sh.erb`.
- Connection/auth behavior in `template/before.sh.erb` and `template/bin/auth`.

## Development Checks

Useful syntax checks:

```bash
erb -x -T '-' form.yml.erb | ruby -c
erb -x -T '-' form.yml.erb | ruby -c
node --check form.js
```

## Related App

The same shared `resource_discovery.erb` and `form.js` pattern is also used by
the Tufts Jupyter Open OnDemand app. Apps can select CPU-only, GPU-only, or mixed
partition behavior by setting `@partition_filter` in their own `form.yml.erb`.
