# Hướng Dẫn Sử Dụng ZMOD4410

## Giới Thiệu Dự Án

Đây là gói phần mềm driver cho cảm biến chất lượng không khí **ZMOD4410** của Renesas, được tích hợp cho hệ thống RT-Thread RTOS.

### ZMOD4410 là gì?

ZMOD4410 là một cảm biến khí số (Digital Gas Sensor) dùng để đo chất lượng không khí trong nhà (Indoor Air Quality - IAQ). Cảm biến này có thể đo:

- **EtOH**: Nồng độ Ethanol (đơn vị: ppm)
- **TVOC**: Tổng hợp chất hữu cơ bay hơi (đơn vị: mg/m³)
- **eCO2**: Nồng độ CO2 ước tính (đơn vị: ppm)
- **IAQ**: Chỉ số chất lượng không khí (1-5, 1 là tốt nhất)

### Thông Số Kỹ Thuật

- **Điện áp hoạt động**: 1.7V ~ 3.6V
- **Giao tiếp**: I2C (địa chỉ mặc định: 0x32)
- **Chế độ hoạt động**: Polling (đọc dữ liệu theo chu kỳ)
- **Thời gian khởi động**: Cần thời gian "warmup" khoảng 60 mẫu đầu tiên

## Cấu Trúc Dự Án

```
zmod4410/
├── README.md                      # Tài liệu chính (tiếng Trung)
├── docs/                          # Tài liệu kỹ thuật PDF
│   ├── ZMOD4410 Programming Manual - Read Me.pdf
│   ├── ZMOD4410-IAQ_2nd_Gen-lib.pdf
│   └── ZMOD4xxx-API.pdf
├── src/                           # Mã nguồn chính
│   ├── demo.c                     # Chương trình demo
│   ├── zmod4410_config_iaq2.h     # Cấu hình cho thuật toán IAQ Gen 2
│   ├── zmod4xxx.c                 # Driver cơ bản ZMOD4xxx
│   └── zmod4xxx.h                 # Header file driver
├── hal/                           # Hardware Abstraction Layer
│   ├── hal_rtthread.c             # HAL cho RT-Thread
│   └── zmod4xxx_hal.h             # Interface HAL
├── ports/                         # Tích hợp với RT-Thread Sensor Framework
│   ├── sensor_renesas_zmod4410.c  # Driver sensor framework
│   └── sensor_renesas_zmod4410.h  # Header file
└── libraries/                     # Thư viện thuật toán IAQ Gen 2
    └── iaq_2nd_gen/               # Thuật toán tính IAQ, TVOC, EtOH, eCO2
```

## Cách Cài Đặt

### Bước 1: Yêu Cầu Hệ Thống

Bạn cần có:
- **RT-Thread 4.0.0** trở lên
- **Sensor Framework** của RT-Thread
- **libc component**
- **I2C driver** được cấu hình đúng trên board của bạn

### Bước 2: Thêm Gói Vào Dự Án

Sử dụng công cụ quản lý gói của RT-Thread:

```
RT-Thread online packages  --->
  peripheral libraries and drivers  --->
    sensors drivers  --->
      [*] zmod4410: Gas Sensor Module ZMOD4410 driver library.
            Version (latest)  --->
```

Kích hoạt libc:

```
-> RT-Thread Components
  -> POSIX layer and C standard library
    [*] Enable libc APIs from the toolchain
```

## Cách Sử Dụng

### Phương Án 1: Sử Dụng với RT-Thread Sensor Framework (Khuyến nghị)

File sử dụng: `ports/sensor_renesas_zmod4410.c`

#### 1. Khởi Tạo Cảm Biến

```c
#include "sensor_renesas_zmod4410.h"

#define ZMOD4410_I2C_BUS "i2c1"  // Thay đổi theo I2C bus của bạn

int rt_hw_zmod4410_port(void)
{
    struct rt_sensor_config cfg;
    cfg.intf.dev_name = ZMOD4410_I2C_BUS;
    
    rt_hw_zmod4410_init("zmod4410", &cfg);
    
    return RT_EOK;
}
INIT_ENV_EXPORT(rt_hw_zmod4410_port);
```

#### 2. Đọc Dữ Liệu

Sau khi khởi tạo, bạn có thể đọc dữ liệu bằng các lệnh MSH:

```shell
# Đọc chỉ số chất lượng không khí IAQ
msh> sensor_polling iaq_zmod

# Đọc nồng độ TVOC
msh> sensor_polling tvoc_zmo

# Đọc nồng độ Ethanol
msh> sensor_polling etoh_zmo

# Đọc nồng độ eCO2
msh> sensor_polling eco2_zmo
```

**Kết quả mẫu:**
```
msh />sensor_polling iaq_zmod
[309438] I/sensor.zmod4410: Warmup!
[311432] I/sensor.cmd: num:  0, IAQ:    1.0 , timestamp:946684800
```

**Lưu ý:** Trong khoảng 60 mẫu đầu tiên, cảm biến sẽ ở chế độ "Warmup!" và giá trị chưa chính xác.

### Phương Án 2: Sử Dụng Demo Trực Tiếp

File sử dụng: `src/demo.c`

**Chú ý quan trọng:** `demo.c` và `sensor_renesas_zmod4410.c` **không được** biên dịch cùng lúc. Chỉ chọn một trong hai.

#### Chạy Demo

```shell
msh> zmod_demo
```

**Kết quả mẫu:**
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

## Hiểu Về API

### API Chính - Driver Cơ Bản

File: `src/zmod4xxx.h`, `src/zmod4xxx.c`

```c
// Đọc thông tin cảm biến
int8_t zmod4xxx_read_sensor_info(zmod4xxx_dev_t *dev);

// Chuẩn bị cảm biến
int8_t zmod4xxx_prepare_sensor(zmod4xxx_dev_t *dev);

// Bắt đầu đo
int8_t zmod4xxx_start_measurement(zmod4xxx_dev_t *dev);

// Đọc trạng thái
int8_t zmod4xxx_read_status(zmod4xxx_dev_t *dev, uint8_t *status);

// Đọc kết quả ADC
int8_t zmod4xxx_read_adc_result(zmod4xxx_dev_t *dev, uint8_t *adc_result);
```

### API Thuật Toán IAQ Gen 2

File: `libraries/iaq_2nd_gen/.../iaq_2nd_gen.h`

```c
// Khởi tạo thuật toán
int8_t init_iaq_2nd_gen(iaq_2nd_gen_handle_t *handle);

// Tính toán kết quả
int8_t calc_iaq_2nd_gen(iaq_2nd_gen_handle_t *handle, 
                        zmod4xxx_dev_t *dev,
                        const uint8_t *sensor_results_table,
                        iaq_2nd_gen_results_t *results);
```

### Cấu Trúc Dữ Liệu Kết Quả

```c
typedef struct {
    float rmox[13];     // Điện trở MOx (13 giá trị)
    float log_rcda;     // log10 của điện trở CDA
    float iaq;          // Chỉ số IAQ (1.0 - 5.0)
    float tvoc;         // TVOC (mg/m³)
    float etoh;         // Ethanol (ppm)
    float eco2;         // eCO2 (ppm)
} iaq_2nd_gen_results_t;
```

## Quy Trình Đo Đạc

```
1. Khởi tạo hardware (I2C, GPIO)
2. Đọc thông tin cảm biến (PID, Product Data)
3. Chuẩn bị cảm biến (prepare_sensor)
4. Khởi tạo thuật toán IAQ
5. Bắt đầu đo (start_measurement)
6. LOOP:
   a. Đợi đo xong (polling status register)
   b. Đọc kết quả ADC
   c. Tính toán thuật toán IAQ
   d. Hiển thị kết quả
   e. Đợi 1.99 giây
   f. Bắt đầu đo mới
7. Dừng và deinitialize
```

## Hiểu Các Giá Trị Đầu Ra

### IAQ (Indoor Air Quality Index)
- **1.0**: Rất tốt (Excellent)
- **2.0**: Tốt (Good)
- **3.0**: Trung bình (Moderate)
- **4.0**: Kém (Poor)
- **5.0**: Rất kém (Unhealthy)

### TVOC (Total Volatile Organic Compounds)
- Đơn vị: mg/m³
- Giá trị thấp (<0.3 mg/m³): Không khí sạch
- Giá trị cao (>10 mg/m³): Cần thông gió

### eCO2 (equivalent CO2)
- Đơn vị: ppm
- 400-1000 ppm: Bình thường
- >1000 ppm: Cần thông gió

### EtOH (Ethanol)
- Đơn vị: ppm
- Thường thấy trong không khí từ rượu, dung môi

## Lưu Ý Quan Trọng

1. **Thời gian khởi động**: Cảm biến cần 60 mẫu đầu tiên (khoảng 2 phút) để "warmup". Trong thời gian này, giá trị trả về chưa chính xác.

2. **Chu kỳ đo**: Nên đợi khoảng 2 giây giữa các lần đo để cảm biến hoạt động tốt.

3. **Chọn mode**: Chỉ chọn một trong hai:
   - `sensor_renesas_zmod4410.c` (tích hợp với Sensor Framework)
   - `demo.c` (standalone demo)

4. **I2C Configuration**: Đảm bảo I2C bus được cấu hình đúng với tốc độ phù hợp.

5. **Cleaning Procedure**: Có thể chạy quy trình làm sạch bề mặt kim loại oxide sau khi lắp ráp (chỉ chạy 1 lần trong vòng đời sản phẩm, mất 10 phút).

## Tài Liệu Tham Khảo

Trong thư mục `docs/` có các tài liệu kỹ thuật chi tiết:

1. **ZMOD4410 Programming Manual - Read Me.pdf**: Hướng dẫn lập trình
2. **ZMOD4410-IAQ_2nd_Gen-lib.pdf**: Tài liệu thuật toán IAQ Gen 2
3. **ZMOD4xxx-API.pdf**: API reference đầy đủ

Datasheet chính thức: https://www2.renesas.cn/cn/zh/products/sensor-products/environmental-sensors/digital-gas-sensors/zmod4410-indoor-air-quality-sensor-platform

## Liên Hệ và Hỗ Trợ

- **Người bảo trì**: Sherman (shaopengyu@rt-thread.com)
- **Repository gốc**: https://github.com/ShermanShao/zmod4410
- **Giấy phép**: Renesas Software License (xem file `Renesas_License_for_Code_Software.pdf`)

## Ví Dụ Code Hoàn Chỉnh

### Ví dụ 1: Sử dụng Sensor Framework

```c
#include <rtthread.h>
#include "sensor.h"
#include "sensor_renesas_zmod4410.h"

#define ZMOD4410_I2C_BUS "i2c1"

// Khởi tạo cảm biến
int rt_hw_zmod4410_port(void)
{
    struct rt_sensor_config cfg;
    cfg.intf.dev_name = ZMOD4410_I2C_BUS;
    rt_hw_zmod4410_init("zmod4410", &cfg);
    return RT_EOK;
}
INIT_ENV_EXPORT(rt_hw_zmod4410_port);

// Ví dụ đọc giá trị IAQ
void read_iaq_example(void)
{
    rt_device_t sensor;
    struct rt_sensor_data data;
    
    // Mở thiết bị sensor IAQ
    sensor = rt_device_find("iaq_zmod");
    if (sensor == RT_NULL) {
        rt_kprintf("Can't find IAQ sensor!\n");
        return;
    }
    
    rt_device_open(sensor, RT_DEVICE_FLAG_RDONLY);
    
    // Đọc giá trị
    if (rt_device_read(sensor, 0, &data, 1) == 1) {
        rt_kprintf("IAQ: %d.%d\n", 
                   data.data.iaq / 10, 
                   data.data.iaq % 10);
    }
    
    rt_device_close(sensor);
}
```

### Ví dụ 2: Sử dụng Demo API Trực Tiếp

Xem file `src/demo.c` để có ví dụ đầy đủ về cách sử dụng API cấp thấp.

---

**Chúc bạn sử dụng thành công!** 🎉
