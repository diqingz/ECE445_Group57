### EMG Electrodes Design
- https://www.nature.com/articles/s41597-022-01484-2 (HD-sEMG, very expensive setup but promising technique)

Buffer amplifier to boost EMG source signal without distortion.
- https://www.ti.com/product/INA321
- [OPA333](https://www.ti.com/product/OPA333)
- [INA128](https://www.ti.com/product/INA128/part-details/INA128UA)

ADC:
- 16bit: ADS1198
- 24bit: ADS1299
### IMU Design
- https://invensense.tdk.com/products/motion-tracking/6-axis/icm-42670-p/
- https://invensense.tdk.com/products/motion-tracking/6-axis/mpu-6050/
- BNO085 https://www.ceva-ip.com/wp-content/uploads/BNO080_085-Datasheet.pdf

Choosing between 6-DOF Accelerometer + Gyro accompanied with 3-DOF Magnetometer VS. 9-DOF full package:
- Magnetometer: LIS3MDL (STMicroelectronics), [IST8306 Magnetometer](https://isentek.com/products_view.php?PID=10&sn=18)
- 6-DOF Accelerometer: LSM6DSOX, ICM-42670
- 9-DOF IMU: BNO085

### Spandex Sleeve Design
- Include thumb hole for easier placement tracking
