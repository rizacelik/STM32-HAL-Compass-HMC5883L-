# STM32 HAL Compass HMC5883L

## Below is an example of a (power-on) initialization process for “continuous-measurement mode:

1. Write CRA (**00**) – **send 0x3C 0x00 0x70** (***8-average**, **15 Hz** default, normal measurement*)
2. Write CRB (**01**) – **send 0x3C 0x01 0xA0** (***Gain=5**, or any other desired gain, **scale = 2.56***)
3. Write Mode (**02**) – **send 0x3C 0x02 0x00** (*Continuous-measurement mode*)
4. Wait *6 ms* or monitor status register or **DRDY** hardware interrupt pin
5. **Loop**

**Send 0x3D>>1 0x06** (*Read all 6 bytes*. If gain is changed then this data set is using previous gain)
Convert three **16-bit** 2’s compliment hex values to decimal values and assign to **X**, **Z**, **Y**, respectively.

**Send 0x3C>>1 0x03** (point to first data register **03**)
Wait about **67 ms** (if **15 Hz** rate) or monitor status register or **DRDY** hardware interrupt pin

**End_loop**

[3-Axis Digital Compass IC HMC5883L DATASHEETS](https://cdn-shop.adafruit.com/datasheets/HMC5883L_3-Axis_Digital_Compass_IC.pdf)
```C++
  #define M_PI 3.14159265359  
  //   Christchurch, 23° 35' EAST  
  int declination_degs = 23;  
  int declination_mins = 35;  
  float declination_offset_radians = ( declination_degs + (1/60 * declination_mins)) * (M_PI / 180);  
  float scale = 2.56;  
  uint8_t data;
  data = 0x70;
1.  HAL_I2C_Mem_Write(&hi2c1, 0x3C>>1, 0x00, 1, &data, 1, HAL_MAX_DELAY);
  data = 0xA0;
2.  HAL_I2C_Mem_Write(&hi2c1, 0x3C>>1, 0x01, 1, &data, 1, HAL_MAX_DELAY);
  data = 0x00;
3.  HAL_I2C_Mem_Write(&hi2c1, 0x3C>>1, 0x02, 1, &data, 1, HAL_MAX_DELAY);
4.  HAL_Delay(6);

5. Loop
  uint8_t buffer[6];
  HAL_I2C_Mem_Read(&hi2c1, 0x3D>>1, 0x06, I2C_MEMADD_SIZE_8BIT, (uint8_t *)&buffer, 6, HAL_MAX_DELAY);
  int16_t x = ((buffer[0] << 8) | buffer[1]) * scale;
  int16_t y = ((buffer[2] << 8) | buffer[3]) * scale;
  int16_t z = ((buffer[4] << 8) | buffer[5]) * scale;

  float heading = atan2(x, y);
  heading += declination_offset_radians;

  // Correct for when signs are reversed.
  if (heading < 0)
    heading += 2 * M_PI;

  // Check for wrap due to addition of declination.
  if (heading > 2 * M_PI)
    heading -= 2 * M_PI;

  // Convert radians to degrees for readability.
  return heading * 180 / M_PI;
  
  HAL_Delay(67);
End_loop
```


# What is Magnetic Declination?
Did you know that magnetic compass does not always point to North? Actually, there are only a few locations on Earth where it points exactly to the True (geographic) North. The direction in which the compass needle points is known as Magnetic North, and the angle between Magnetic North and the True North direction is called magnetic declination.

If the compass at your place is pointing clockwise with respect to the True North, declination is positive or EAST
If the compass at your place is pointing counter-clockwise with respect to the True North, declination is negative or WEST

![image](https://user-images.githubusercontent.com/19993109/153770299-f81771e1-8781-4f3b-abfd-1c82487a5e74.png)
![image](https://user-images.githubusercontent.com/19993109/153770313-06bb0adc-2fb6-4ae8-855d-6f9b4a937f44.png)

Magnetic declination must be compensated for by adding the declination to the compass bearing if it is negative or subtracting if it is positive.

# STM32 HAL Example Code

```C++
#define HMC5883L_ADDRESS  0x1E

#define COMPASS_CONFIG_REGISTER_A 0x00
#define COMPASS_CONFIG_REGISTER_B 0x01
#define COMPASS_MODE_REGISTER     0x02
#define COMPASS_DATA_REGISTER     0x03

#define Data_Output_X_MSB 0x03
#define Data_Output_X_LSB 0x04
#define Data_Output_Z_MSB 0x05
#define Data_Output_Z_LSB 0x06
#define Data_Output_Y_MSB 0x07
#define Data_Output_Y_LSB 0x08

#define COMPASS_SAMPLE1 0x00 // 1-average(Default)
#define COMPASS_SAMPLE2 0x20 // 2-average
#define COMPASS_SAMPLE4 0x40 // 4-average
#define COMPASS_SAMPLE8 0x60 // 8-average

#define COMPASS_RATE0_75 0x00 // 0.75Hz
#define COMPASS_RATE1_5  0x04 // 1.5Hz
#define COMPASS_RATE3    0x08 // 3Hz
#define COMPASS_RATE7_5  0x0C // 7.5Hz
#define COMPASS_RATE15   0x10 // 15Hz (Default)
#define COMPASS_RATE30   0x14 // 30Hz
#define COMPASS_RATE75   0x18 // 75Hz

#define COMPASS_MEASURE_NORMAL   0x00 // Normal measurement configuration (Default)
#define COMPASS_MEASURE_POSITIVE 0x01 // Positive bias configuration for X, Y, and Z axes.
#define COMPASS_MEASURE_NEGATIVE 0x02 // Negative bias configuration for X, Y and Z axes.

#define COMPASS_SCALE_088  0x0  //0.88 Ga
#define COMPASS_SCALE_130  0x20 //1.3 Ga (Default)
#define COMPASS_SCALE_190  0x40 //1.9 Ga
#define COMPASS_SCALE_250  0x60 //2.5 Ga
#define COMPASS_SCALE_400  0x80 //4.0 Ga
#define COMPASS_SCALE_470  0xA0 //4.7 Ga
#define COMPASS_SCALE_560  0xC0 //5.6 Ga
#define COMPASS_SCALE_810  0xE0 //8.1 Ga

#define COMPASS_CONTINUOUS 0x00  //Continuous-Measurement Mode.
#define COMPASS_SINGLE     0x01  //Single-Measurement Mode (Default). 
#define COMPASS_IDLE       0x02  //Idle Mode. Device is placed in idle mode

float scale;
float declination_offset_radians;


// Magnetic Declination is the correction applied according to your present location
// in order to get True North from Magnetic North, it varies from place to place.
//
// The declination for your area can be obtained from http://www.magnetic-declination.com/
// Take the "Magnetic Declination" line that it gives you in the information,
//
// Examples:
//   Christchurch, 23° 35' EAST
//   Wellington  , 22° 14' EAST
//   Dunedin     , 25° 8'  EAST
//   Auckland    , 19° 30' EAS
//   SANFINA     , 4°  33' WEST

SetDeclination( int declination_degs , int declination_mins, char declination_dir )
{
  // Convert declination to decimal degrees
  switch (declination_dir)
  {
    // North and East are positive
    case 'E':
      declination_offset_radians = 0 - ( declination_degs + (1 / 60 * declination_mins)) * (M_PI / 180);
      break;

    // South and West are negative
    case 'W':
      declination_offset_radians =  (( declination_degs + (1 / 60 * declination_mins) ) * (M_PI / 180));
      break;
  }
}


SetDeclination(23, 35, 'E');


void SetSamplingMode( uint16_t sampling_mode , uint16_t rate, uint16_t measure)
{
  uint8_t data = (sampling_mode | rate | measure);
  HAL_I2C_Mem_Write(&hi2c1, HMC5883L_ADDRESS, COMPASS_CONFIG_REGISTER_A, 1, &data, 1, HAL_MAX_DELAY);
}

SetSamplingMode(COMPASS_SAMPLE8 , COMPASS_RATE15, COMPASS_MEASURE_NORMAL);


void SetScaleMode(uint8_t ScaleMode)
{
  switch (ScaleMode)
    ​ {
    case COMPASS_SCALE_088:
      scale = 0.73;
      break;
    case COMPASS_SCALE_130:
      scale = 0.92;
      break;
    case COMPASS_SCALE_190:
      scale = 1.22;
      break;
    case COMPASS_SCALE_250:
      scale = 1.52;
      break;
    case COMPASS_SCALE_400:
      scale = 2.27;
      break;
    case COMPASS_SCALE_470:
      scale = 2.56;
      break;
    case COMPASS_SCALE_560:
      scale = 3.03;
      break;
    case COMPASS_SCALE_810:
      scale = 4.35;
      break;
    default:
      scale = 0.92;
      ScaleMode = COMPASS_SCALE_130;
  }

  HAL_I2C_Mem_Write(&hi2c1, HMC5883L_ADDRESS, COMPASS_CONFIG_REGISTER_B, 1, &ScaleMode, 1, HAL_MAX_DELAY);
}

SetScaleMode( COMPASS_SCALE_130);


void SetMeasureMode( uint8_t Measure)
{
  HAL_I2C_Mem_Write(&hi2c1, HMC5883L_ADDRESS, COMPASS_MODE_REGISTER, 1, &Measure, 1, HAL_MAX_DELAY);
}

SetMeasureMode(COMPASS_CONTINUOUS);


uint16_t CompasReadAxis(uint16_t register) {
  uint16_t buffer[1];
  HAL_I2C_Mem_Read(&hi2c1, HMC5883L_ADDRESS, register, I2C_MEMADD_SIZE_8BIT, (uint8_t *)&buffer, 2, HAL_MAX_DELAY);
  int16_t value = buffer[0] << 8 | buffer[1];
  return value;
}

float x = CompasReadAxis(Data_Output_X_MSB) * scale;
float y = CompasReadAxis(Data_Output_Y_MSB) * scale;
float z = CompasReadAxis(Data_Output_Z_MSB) * scale;

float heading = atan2(x, y);
heading += declination_offset_radians;

// Correct for when signs are reversed.
if (heading < 0)
  heading += 2 * M_PI;

// Check for wrap due to addition of declination.
if (heading > 2 * M_PI)
  heading -= 2 * M_PI;

// Convert radians to degrees for readability.
return heading * 180 / M_PI;

float CompasRead6Axis(void)
{
  uint8_t buffer[6];
  HAL_I2C_Mem_Read(&hi2c1, HMC5883L_ADDRESS, COMPASS_DATA_REGISTER, I2C_MEMADD_SIZE_8BIT, (uint8_t *)&buffer, 6, HAL_MAX_DELAY);
  int16_t x = ((buffer[0] << 8) | buffer[1]) * scale;
  int16_t y = ((buffer[2] << 8) | buffer[3]) * scale;
  int16_t z = ((buffer[4] << 8) | buffer[5]) * scale;

  float heading = atan2(x, y);
  heading += declination_offset_radians;

  // Correct for when signs are reversed.
  if (heading < 0)
    heading += 2 * M_PI;

  // Check for wrap due to addition of declination.
  if (heading > 2 * M_PI)
    heading -= 2 * M_PI;

  // Convert radians to degrees for readability.
  return heading * 180 / M_PI;
}

float degrees = CompasRead6Axis();


// To compensate a compass for Tilt sensor and Compass
float compensate(float compass_X, float compass_Y, float compass_Z, float pitch, float roll) {

  float IMU_roll = pitch * (M_PI / 180);
  float IMU_pitch = roll * (M_PI / 180);

  float XH = compass_X * cos(IMU_pitch) + compass_Y * sin(IMU_roll) * sin(IMU_pitch) - compass_Z * cos(IMU_roll) * sin(IMU_pitch);
  float YH = compass_Y * cos(IMU_roll) + compass_Z * sin(IMU_roll)
       // Azimuth = atan2(YH / XH)
  float Azimuth = atan2(YH / XH) * 180 / M_PI;  
  Azimuth += declination_offset_radians; // see https://www.magnetic-declination.com/

  if (Azimuth < 0) {
    Azimuth += 360;
  }
  else if (Azimuth >= 360) {
    Azimuth -= 360;
  }

  return Azimuth;
}


```

