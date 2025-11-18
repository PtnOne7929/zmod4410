# ZMOD4410 Usage Guide

## Project Overview

This is a driver software package for the Renesas **ZMOD4410** indoor air quality sensor, integrated for the RT-Thread RTOS system.

### What is ZMOD4410?

ZMOD4410 is a Digital Gas Sensor used to measure Indoor Air Quality (IAQ). This sensor can measure:

- **EtOH**: Ethanol concentration (unit: ppm)
- **TVOC**: Total Volatile Organic Compounds (unit: mg/m³)
- **eCO2**: Estimated CO2 concentration (unit: ppm)
- **IAQ**: Air Quality Index (1-5, where 1 is best)

### Technical Specifications

- **Operating Voltage**: 1.7V ~ 3.6V
- **Communication**: I2C (default address: 0x32)
- **Operating Mode**: Polling (periodic data reading)
- **Warm-up Time**: Requires approximately first 60 samples for stabilization

## Project Structure

```
zmod4410/
├── README.md                      # Main documentation (Chinese)
├── docs/                          # Technical documentation PDFs
│   ├── ZMOD4410 Programming Manual - Read Me.pdf
│   ├── ZMOD4410-IAQ_2nd_Gen-lib.pdf
│   └── ZMOD4xxx-API.pdf
├── src/                           # Main source code
│   ├── demo.c                     # Demo application
│   ├── zmod4410_config_iaq2.h     # Configuration for IAQ Gen 2 algorithm
│   ├── zmod4xxx.c                 # Basic ZMOD4xxx driver
│   └── zmod4xxx.h                 # Driver header file
├── hal/                           # Hardware Abstraction Layer
│   ├── hal_rtthread.c             # HAL for RT-Thread
│   └── zmod4xxx_hal.h             # HAL interface
├── ports/                         # RT-Thread Sensor Framework integration
│   ├── sensor_renesas_zmod4410.c  # Sensor framework driver
│   └── sensor_renesas_zmod4410.h  # Header file
└── libraries/                     # IAQ Gen 2 algorithm library
    └── iaq_2nd_gen/               # Algorithm for IAQ, TVOC, EtOH, eCO2
```

## Installation

### Step 1: System Requirements

You need:
- **RT-Thread 4.0.0** or higher
- RT-Thread **Sensor Framework**
- **libc component**
- Properly configured **I2C driver** on your board

### Step 2: Add Package to Project

Using RT-Thread package manager:

```
RT-Thread online packages  --->
  peripheral libraries and drivers  --->
    sensors drivers  --->
      [*] zmod4410: Gas Sensor Module ZMOD4410 driver library.
            Version (latest)  --->
```

Enable libc:

```
-> RT-Thread Components
  -> POSIX layer and C standard library
    [*] Enable libc APIs from the toolchain
```

## How to Use

### Method 1: Using RT-Thread Sensor Framework (Recommended)

File: `ports/sensor_renesas_zmod4410.c`

#### 1. Initialize Sensor

```c
#include "sensor_renesas_zmod4410.h"

#define ZMOD4410_I2C_BUS "i2c1"  // Change according to your I2C bus

int rt_hw_zmod4410_port(void)
{
    struct rt_sensor_config cfg;
    cfg.intf.dev_name = ZMOD4410_I2C_BUS;
    
    rt_hw_zmod4410_init("zmod4410", &cfg);
    
    return RT_EOK;
}
INIT_ENV_EXPORT(rt_hw_zmod4410_port);
```

#### 2. Read Data

After initialization, you can read data using MSH commands:

```shell
# Read IAQ air quality index
msh> sensor_polling iaq_zmod

# Read TVOC concentration
msh> sensor_polling tvoc_zmo

# Read Ethanol concentration
msh> sensor_polling etoh_zmo

# Read eCO2 concentration
msh> sensor_polling eco2_zmo
```

**Sample output:**
```
msh />sensor_polling iaq_zmod
[309438] I/sensor.zmod4410: Warmup!
[311432] I/sensor.cmd: num:  0, IAQ:    1.0 , timestamp:946684800
```

**Note:** During the first 60 samples, the sensor will be in "Warmup!" mode and values may not be accurate.

### Method 2: Using Direct Demo

File: `src/demo.c`

**Important Note:** `demo.c` and `sensor_renesas_zmod4410.c` should **NOT** be compiled together. Choose only one.

#### Run Demo

```shell
msh> zmod_demo
```

**Sample output:**
```
Evaluate measurements in a loop. Press any key to quit.

*********** Measurements ***********
 Rmox[0] = 0.100 kOhm
 Rmox[1] = 0.100 kOhm
 ...
 Rmox[12] = 0.100 kOhm
 log_Rcda = 0.000 logOhm
 EtOH =  0.008 ppm
 TVOC =  0.016 mg/m^3
 eCO2 =  400 ppm
 IAQ  =  1.0
Warmup!
************************************
```

## Understanding the API

### Core Driver API

Files: `src/zmod4xxx.h`, `src/zmod4xxx.c`

```c
// Read sensor information
int8_t zmod4xxx_read_sensor_info(zmod4xxx_dev_t *dev);

// Prepare sensor
int8_t zmod4xxx_prepare_sensor(zmod4xxx_dev_t *dev);

// Start measurement
int8_t zmod4xxx_start_measurement(zmod4xxx_dev_t *dev);

// Read status
int8_t zmod4xxx_read_status(zmod4xxx_dev_t *dev, uint8_t *status);

// Read ADC results
int8_t zmod4xxx_read_adc_result(zmod4xxx_dev_t *dev, uint8_t *adc_result);
```

### IAQ Gen 2 Algorithm API

File: `libraries/iaq_2nd_gen/.../iaq_2nd_gen.h`

```c
// Initialize algorithm
int8_t init_iaq_2nd_gen(iaq_2nd_gen_handle_t *handle);

// Calculate results
int8_t calc_iaq_2nd_gen(iaq_2nd_gen_handle_t *handle, 
                        zmod4xxx_dev_t *dev,
                        const uint8_t *sensor_results_table,
                        iaq_2nd_gen_results_t *results);
```

### Result Data Structure

```c
typedef struct {
    float rmox[13];     // MOx resistance (13 values)
    float log_rcda;     // log10 of CDA resistance
    float iaq;          // IAQ index (1.0 - 5.0)
    float tvoc;         // TVOC (mg/m³)
    float etoh;         // Ethanol (ppm)
    float eco2;         // eCO2 (ppm)
} iaq_2nd_gen_results_t;
```

## Measurement Flow

```
1. Initialize hardware (I2C, GPIO)
2. Read sensor information (PID, Product Data)
3. Prepare sensor (prepare_sensor)
4. Initialize IAQ algorithm
5. Start measurement (start_measurement)
6. LOOP:
   a. Wait for measurement completion (poll status register)
   b. Read ADC results
   c. Calculate IAQ algorithm
   d. Display results
   e. Wait 1.99 seconds
   f. Start new measurement
7. Stop and deinitialize
```

## Understanding Output Values

### IAQ (Indoor Air Quality Index)
- **1.0**: Excellent
- **2.0**: Good
- **3.0**: Moderate
- **4.0**: Poor
- **5.0**: Unhealthy

### TVOC (Total Volatile Organic Compounds)
- Unit: mg/m³
- Low values (<0.3 mg/m³): Clean air
- High values (>10 mg/m³): Ventilation needed

### eCO2 (equivalent CO2)
- Unit: ppm
- 400-1000 ppm: Normal
- >1000 ppm: Ventilation recommended

### EtOH (Ethanol)
- Unit: ppm
- Commonly found in air from alcohol, solvents

## Important Notes

1. **Warm-up Time**: The sensor needs the first 60 samples (about 2 minutes) to "warm up". During this time, returned values may not be accurate.

2. **Measurement Cycle**: Wait about 2 seconds between measurements for optimal sensor performance.

3. **Mode Selection**: Choose only one:
   - `sensor_renesas_zmod4410.c` (integrated with Sensor Framework)
   - `demo.c` (standalone demo)

4. **I2C Configuration**: Ensure I2C bus is properly configured with appropriate speed.

5. **Cleaning Procedure**: A cleaning procedure can be run to clean the metal oxide surface after assembly (only once in product lifetime, takes 10 minutes).

## Documentation References

The `docs/` directory contains detailed technical documentation:

1. **ZMOD4410 Programming Manual - Read Me.pdf**: Programming guide
2. **ZMOD4410-IAQ_2nd_Gen-lib.pdf**: IAQ Gen 2 algorithm documentation
3. **ZMOD4xxx-API.pdf**: Complete API reference

Official Datasheet: https://www2.renesas.cn/cn/zh/products/sensor-products/environmental-sensors/digital-gas-sensors/zmod4410-indoor-air-quality-sensor-platform

## Contact and Support

- **Maintainer**: Sherman (shaopengyu@rt-thread.com)
- **Original Repository**: https://github.com/ShermanShao/zmod4410
- **License**: Renesas Software License (see `Renesas_License_for_Code_Software.pdf`)

## Complete Code Examples

### Example 1: Using Sensor Framework

```c
#include <rtthread.h>
#include "sensor.h"
#include "sensor_renesas_zmod4410.h"

#define ZMOD4410_I2C_BUS "i2c1"

// Initialize sensor
int rt_hw_zmod4410_port(void)
{
    struct rt_sensor_config cfg;
    cfg.intf.dev_name = ZMOD4410_I2C_BUS;
    rt_hw_zmod4410_init("zmod4410", &cfg);
    return RT_EOK;
}
INIT_ENV_EXPORT(rt_hw_zmod4410_port);

// Example: Read IAQ value
void read_iaq_example(void)
{
    rt_device_t sensor;
    struct rt_sensor_data data;
    
    // Open IAQ sensor device
    sensor = rt_device_find("iaq_zmod");
    if (sensor == RT_NULL) {
        rt_kprintf("Can't find IAQ sensor!\n");
        return;
    }
    
    rt_device_open(sensor, RT_DEVICE_FLAG_RDONLY);
    
    // Read value
    if (rt_device_read(sensor, 0, &data, 1) == 1) {
        rt_kprintf("IAQ: %d.%d\n", 
                   data.data.iaq / 10, 
                   data.data.iaq % 10);
    }
    
    rt_device_close(sensor);
}
```

### Example 2: Using Direct Demo API

See `src/demo.c` for a complete example of using the low-level API.

---

**Happy coding!** 🎉
