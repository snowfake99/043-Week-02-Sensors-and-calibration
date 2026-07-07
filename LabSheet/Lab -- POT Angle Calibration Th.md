# **ใบงาน การทดลอง POT วัดมุมด้วยกระดาษ + ESP32**

## **ภาพรวม**

ใบงานนี้เป็นการทดลองเซนเซอร์อนาล็อกอย่างง่ายโดยใช้ **Potentiometer (POT)** เพื่อวัดมุมการหมุน นักศึกษาจะได้เรียนรู้
- การอ่านค่า ADC ด้วย ESP-IDF
- การสร้างหน้าปัดมุมบนกระดาษ
- การเก็บข้อมูลเพื่อทำ Calibration
- การแปลงค่า ADC → มุมจริงด้วยสมการเชิงเส้น
- การตรวจสอบความแม่นยำของเซนเซอร์

# **เป้าหมายการเรียนรู้**

- เข้าใจหลักการทำงานของ POT
- อ่านค่า ADC ด้วย ESP-IDF (ADC driver)
- สร้างหน้าปัดมุมบนกระดาษ
- เก็บข้อมูลเพื่อทำคาลิเบรต
- แปลงค่า ADC → มุมจริง
- ตรวจสอบความแม่นยำของระบบวัดมุม

# **อุปกรณ์ที่ใช้**

- POT 10k
- ESP32
- สาย Jumper
- กระดาษ A4 หรือกระดาษแข็ง
- ปากกา ไม้บรรทัด โปรแทรกเตอร์
- เทปกาว

# **ขั้นตอนที่ 1 สร้างหน้าปัดมุมบนกระดาษ**

1. วาดวงกลมขนาด 10–12 ซม.
2. ทำจุดศูนย์กลาง
3. ใช้โปรแทรกเตอร์แบ่งมุม เช่น 0°, 45°, 90°, 135°, 180°
4. เขียนตัวเลขกำกับ
5. ติดตั้ง POT ให้แกนหมุนตรงศูนย์กลาง
6. ทำเข็มชี้มุมด้วยกระดาษ

# **ขั้นตอนที่ 2 ต่อวงจร POT กับ ESP32**

- ขาซ้าย POT → 3.3V
- ขาขวา POT → GND
- ขากลาง POT → GPIO34 (ADC1_CH6)

# **ขั้นตอนที่ 3 อ่านค่า ADC (ESP-IDF)**


###  โค้ด ESP-IDF สำหรับอ่านค่า ADC

```c
#include "driver/adc.h"
#include "esp_log.h"

void app_main(void)
{
    // กำหนดความละเอียด ADC = 12 bit (0–4095)
    adc1_config_width(ADC_WIDTH_BIT_12);

    // กำหนด channel และ attenuation
    adc1_config_channel_atten(ADC1_CHANNEL_6, ADC_ATTEN_DB_11);

    while (1) {
        int adc = adc1_get_raw(ADC1_CHANNEL_6);
        printf("ADC = %d\n", adc);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

#  **ขั้นตอนที่ 4 เก็บข้อมูลคาลิเบรต**

หมุน POT ให้ตรงกับมุมบนกระดาษ แล้วบันทึกค่า ADC

| มุม (°) | ค่า ADC(คาดการณ์) | ค่า ADC ที่อ่านได้ |
| :-----: | :---------------: | ------------------ |
|    0    |        180        |         110            |
|   45    |       1100        |          662        |
|   90    |       2100        |        1452         |
|   135   |       3000        |         2160        |
|   180   |       3900        |         2879ww       |

จากข้อมูลนี้ให้หาค่า

- **$ADC_{min}$** = ค่า ADC ที่มุม 0°
- **$ADC_{max}$** = ค่า ADC ที่มุมสูงสุด
- **$θ_{max}$** = มุมสูงสุด เช่น 180°
    

# **ขั้นตอนที่ 5 แปลงค่า ADC → มุมจริง**

สมการเชิงเส้น

$$angle= \frac{ADC−ADC_{min}}{ADC_{max}−ADC_{min}}⋅θ_{max}
$$
**ตัวอย่าง**

$ADC_{min} = 180$

$ADC_{max} = 3900$

$θ_{max} = 180°$

$$angle=(ADC−180)⋅\frac{180}{3720}
$$
# **ขั้นตอนที่ 6 แสดงผลมุมแบบเรียลไทม์ (ESP-IDF)**

```c
#include "driver/adc.h"
#include "esp_log.h"

#define ADC_MIN 180
#define ADC_MAX 3900
#define DEG_MAX 180.0

float adc_to_angle(int adc)
{
    return (adc - ADC_MIN) * DEG_MAX / (ADC_MAX - ADC_MIN);
}

void app_main(void)
{
    adc1_config_width(ADC_WIDTH_BIT_12);
    adc1_config_channel_atten(ADC1_CHANNEL_6, ADC_ATTEN_DB_11);

    while (1) {
        int adc = adc1_get_raw(ADC1_CHANNEL_6);
        float angle = adc_to_angle(adc);

        printf("ADC = %d  Angle = %.2f deg\n", adc, angle);
        vTaskDelay(pdMS_TO_TICKS(300));
    }
}
```

# **ขั้นตอนที่ 7 ตรวจสอบความแม่นยำ**

ให้นักศึกษาทดสอบมุมแบบสุ่ม

1. หมุนไปที่มุมที่คาดไว้
2. อ่านค่ามุมจาก ESP32
3. เปรียบเทียบกับมุมบนกระดาษ
4. บันทึก error

ถ้า error มาก

- ตรวจสอบ POT ว่าติดตั้งตรงศูนย์กลางหรือไม่
- ตรวจสอบ ADC_min / ADC_max
- พิจารณาใช้ **piecewise calibration**