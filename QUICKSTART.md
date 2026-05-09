# tool-mlxreg Quick Start Guide

## Prerequisites

✅ Image pushed to quay.io: `quay.io/hnhan/mft-tools:4.35.0`
✅ Tool registered in Crucible: `/opt/crucible/subprojects/tools/mlxreg/`

## Usage in Benchmark Configuration

### Option 1: Use with Custom MFT Tools Image (Recommended)

Add to your rickshaw/crucible configuration:

```json
{
  "endpoints": [{
    "type": "k8s",
    "k8s": [{
      "host": "your-k8s-api-server",
      "user": "your-user",
      "client": {
        "userenv": "quay.io/hnhan/mft-tools:4.35.0"
      },
      "server": {
        "userenv": "quay.io/hnhan/mft-tools:4.35.0"
      }
    }]
  }],
  "tool-params": [{
    "tool": "mlxreg",
    "params": [
      {
        "arg": "devices",
        "val": "0000:b5:00.0"
      },
      {
        "arg": "interval",
        "val": "2"
      },
      {
        "arg": "sensors",
        "val": "1,2,6"
      }
    ]
  }]
}
```

### Option 2: Use with Standard Userenv (Requires Manual MFT Installation)

If using `fedora-latest` or `rhubi9-latest`, you must manually install MFT tools in your userenv.

## Finding Your Device BDF Address

On the worker node where the BF-3 DPU is installed:

```bash
# Method 1: Via lspci
lspci | grep -i mellanox

# Method 2: Via OpenShift (if using k8s)
oc debug node/<worker-node-name>
chroot /host
lspci | grep -i mellanox
```

Example output:
```
0000:b5:00.0 Ethernet controller: Mellanox Technologies MT2894 Family [ConnectX-6 Lx]
0000:b5:00.2 System peripheral: Mellanox Technologies MT2894 Family [BlueField-3 SoC Management Interface]
```

Use the first address (e.g., `0000:b5:00.0`) for the `--devices` parameter.

## Multi-Device Example

If your worker node has multiple BF-3 DPUs:

```json
{
  "tool": "mlxreg",
  "params": [
    {
      "arg": "devices",
      "val": "0000:b5:00.0,0000:c3:00.0"
    },
    {
      "arg": "interval",
      "val": "1"
    },
    {
      "arg": "sensors",
      "val": "1,2,6"
    }
  ]
}
```

## Sensor Selection

Default sensors (1, 2, 6) are recommended:

| Sensor | Name | Description | Typical Value |
|--------|------|-------------|---------------|
| 1 | Vr0Pwr | Voltage Regulator 0 power | ~18W |
| 2 | Vr1Pwr | Voltage Regulator 1 power | ~28W |
| 6 | PwrEnv | Power envelope/budget | ~124W |

**Total DPU power consumption = Vr0Pwr + Vr1Pwr**

To collect different sensors:
```json
{
  "arg": "sensors",
  "val": "1,2,3,4,5,6"
}
```

Valid sensor indices: 1-21 (per MVCAP register sensor_map)

## Output Files

After the benchmark completes, look for:

```
<run-dir>/worker-<N>/tools-<N>/mlxreg/mlxreg-data/<device-bdf>.csv.xz
```

Example:
```
mlxreg-data/0000b50000.csv.xz
```

Decompress and view:
```bash
xzcat mlxreg-data/0000b50000.csv.xz | head -20
```

Expected format:
```csv
timestamp,date,device,sensor_index,sensor_name,power_watts,status
1715180000.123456,2026-05-08 12:00:00,0000:b5:00.0,1,Vr0Pwr,18.0,OK
1715180000.234567,2026-05-08 12:00:00,0000:b5:00.0,2,Vr1Pwr,28.0,OK
1715180000.345678,2026-05-08 12:00:00,0000:b5:00.0,6,PwrEnv,124.0,OK
```

## Troubleshooting

### Error: "Cannot access device"

**Check 1:** Verify container is privileged
```bash
# In your rickshaw config, ensure privileged mode
```

**Check 2:** Verify device exists
```bash
oc exec -it <worker-pod> -- lspci | grep -i mellanox
```

**Check 3:** Test mlxreg manually
```bash
oc exec -it <worker-pod> -- mlxreg --device 0000:b5:00.0 --reg_name MVCAP --get
```

### Error: "mlxreg command not found"

You're not using the correct container image. Ensure:
- Using `quay.io/hnhan/mft-tools:4.35.0`
- OR MFT tools are manually installed in your userenv

### No output files

Check the tool logs:
```bash
<run-dir>/worker-<N>/tools-<N>/mlxreg/mlxreg-start-stderrout.txt
<run-dir>/worker-<N>/tools-<N>/mlxreg/mlxreg-stop-stderrout.txt
```

## Example Integration with Regulus

For your NVD DPU lab setup, add to your run configuration:

```bash
# In your run.sh or config generation
tool_params+=" --tool-param mlxreg,devices:0000:b5:00.0"
tool_params+=" --tool-param mlxreg,interval:2"
tool_params+=" --tool-param mlxreg,sensors:1,2,6"

# Specify the MFT tools userenv for worker nodes
endpoint_opt+=" --endpoint k8s,user:kubeadmin,host:api.cluster.example.com,worker:1,userenv:quay.io/hnhan/mft-tools:4.35.0"
```

## See Also

- [README.md](README.md) - Full documentation
- [MLX-CONFIG.md](../tool-power/MLX-CONFIG.md) - Technical details on mlxreg registers
- [tool-power](../tool-power/) - Complementary Redfish-based power monitoring
