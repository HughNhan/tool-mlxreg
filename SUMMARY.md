# tool-mlxreg - Implementation Summary

## Overview
Created a new Crucible tool for collecting BF-3 DPU power telemetry via mlxreg (MFT tools).

## Files Created

### Core Scripts
- **mlxreg-start** - Parse parameters, validate device access, launch collector
- **mlxreg-collect** - Main collection loop querying MVCR sensors
- **mlxreg-stop** - Stop collector and compress CSV output
- **mlxreg-post-process** - Summary statistics (placeholder for future metrics)

### Configuration
- **rickshaw.json** - Worker node deployment configuration
- **workshop.json** - Container userenv specification

### Documentation
- **README.md** - Comprehensive usage guide
- **LICENSE** - Apache 2.0 license

## Key Features

### Multi-Device Support
- Accepts comma-separated PCI BDF addresses via `--devices` parameter
- Creates separate CSV file per device
- Collects from all devices each interval

### Configurable Sensors
- Default: sensors 1, 2, 6 (Vr0Pwr, Vr1Pwr, PwrEnv)
- Customizable via `--sensors` parameter
- Auto-decodes sensor names from register values

### CSV Output Format
```csv
timestamp,date,device,sensor_index,sensor_name,power_watts,status
1715180000.123456,2026-05-08 12:00:00,0000:b5:00.0,1,Vr0Pwr,18.0,OK
```

### Data Processing
- Converts power values from 1/10 watt to watts automatically
- Decodes ASCII sensor names from hex fields
- Includes error handling and status reporting
- Compresses output with xz

## Usage Example

### Rickshaw Configuration
```json
{
  "tool-params": [{
    "tool": "mlxreg",
    "params": [
      {"arg": "devices", "val": "0000:b5:00.0"},
      {"arg": "interval", "val": "2"},
      {"arg": "sensors", "val": "1,2,6"}
    ]
  }]
}
```

### Output
- Files: `mlxreg-data/<device-bdf>.csv.xz`
- Format: Timestamped CSV with power readings per sensor

## Architecture

### Deployment
- Runs on **worker nodes** (where DPU hardware is present)
- Executes in **privileged container** with MFT tools
- Similar to tool-procstat deployment model

### Container Requirements
- Base: UBI9
- Packages: MFT 4.35.0 (mft, mft-mlx5)
- Privileges: --privileged (for PCI access)
- Example image: quay.io/hnhan/mft-tools:4.35.0

## Technical Details

### MVCR Register Query
```bash
mlxreg --device 0000:b5:00.0 --reg_name MVCR --indexes "sensor_index=2" --get
```

### Power Calculation
- Raw value: `power_sensor_value` in hex (e.g., 0x118 = 280)
- Conversion: Divide by 10 → 28.0 watts
- Total power = Vr0Pwr + Vr1Pwr

### Sensor Name Decoding
- Fields: `sensor_name_hi` + `sensor_name_lo`
- Format: ASCII bytes in hex (e.g., 0x56723150 + 0x77720000 = "Vr1Pwr")
- Python conversion strips null bytes

## Next Steps

1. **Testing**: Test on actual worker node with BF-3 DPU
2. **Post-processing**: Implement metric generation for OpenSearch
3. **Integration**: Add to benchmark run configurations
4. **Validation**: Correlate with Redfish readings from tool-power
5. **Documentation**: Add to MLX-CONFIG.md

## Differences from tool-power

| Aspect | tool-power | tool-mlxreg |
|--------|-----------|-------------|
| Deployment | Profiler node | Worker nodes |
| Access method | Network/Redfish API | Local PCIe registers |
| Requires | BMC network access | Privileged container |
| Works when | BMC initialized | Always (direct HW access) |
| Scope | Remote endpoints | Local DPU hardware |

## Benefits

✅ Works when BMC sensors show "Absent"
✅ No IPMB dependency
✅ Direct hardware access
✅ Per-voltage-regulator breakdown
✅ Lower latency (no network overhead)
✅ Multi-device support
