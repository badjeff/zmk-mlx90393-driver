# MLX90393 input driver for ZMK

This work is based on [te9no/zmk-driver-MLX90393](https://github.com/te9no/zmk-driver-MLX90393).

#### What is different to [te9no/zmk-driver-MLX90393](https://github.com/te9no/zmk-driver-MLX90393)
- SPI Bus Compitable
- INT pin support with Wake-On Change Mode
- Auto-calibration for Wake-On Change threshold
- Auto-upshift to Burst Mode on leaving dead zone
- Auto-downshift to Wake-on Change mode on back to neutral position
- Configurable axis disable
- Fix on-chip sample rate at ~20ms


## How is it auto-calibrated and perform measure mode up/down-shift ?

Default to measure 100 samples on bootup to collect the neutral drifts of each axes. And then, apply the mean of samples (scaled down to 75%) on each axes as the Wake-on Change Threashold. Intrrupts would be reduced on neutral position at Wake-on Change Mode. But, few false-interrupts still occur occasionally due to unstable magnetic field.

On any axes exceed the configurable deadzone, it upshifts to Burst Mode to intrrupt at full resolution. While it detects the magnetic souece back to its neutral position by counting up to 75 times without any axes exceeding the deadzone, it downshifts to Wake-on Change Mode.


## Installation

Include this project on ZMK's west manifest in `config/west.yml`:

```yml
manifest:
  remotes:
    ...
    # START #####
    - name: badjeff
      url-base: https://github.com/badjeff
    # END #######
    ...
  projects:
    ...
    # START #####
    - name: zmk-mlx90393-driver
      remote: badjeff
      revision: main
    # END #######
    ...
  self:
    path: config
```

Update `board.overlay` adding the necessary bits (update the pins for your board accordingly):

```dts
&pinctrl {
    spi2_default: spi2_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                    <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                    <NRF_PSEL(SPIM_MISO, 0, 17)>; /* not a typo, using 3-wire SPI */
        };
    };
    spi2_sleep: spi2_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                    <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                    <NRF_PSEL(SPIM_MISO, 0, 17)>; /* not a typo, using 3-wire SPI */
            low-power-enable;
        };
    };
};

#include <dt-bindings/spi/spi.h>
&spi2 {
    compatible = "nordic,nrf-spim";
    status = "okay";
    pinctrl-0 = <&spi2_default>;
    pinctrl-1 = <&spi2_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio1 4 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
    spi_mlx90393: mlx90393@0 {
        compatible = "melexis,mlx90393";
        status = "okay";
        spi-max-frequency = <1000000>;
        reg = <0>;
        irq-gpios = <&gpio1 6 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>; // INT pin

        /* === Optional properties (uncomment to enable) === */
        // no-x: /* skip measure x axis on chip (default read xyz) */
        // no-y: /* skip measure y axis on chip (default read xyz) */
        // no-z; /* skip measure z axis on chip (default read xyz) */
        // calib-cycle = <100>; /* count of auto calibration cycle */
        // woc-thd-trim-pctg = <75>;
            /* Percentage to trim calibration results before setup wake-on-change threshold.
               Adjust this to filter extreme cases of samples during calibration. */
        // ex-woc-thd-xy = <0>; /* extra delta for XY-axis wake-on-change threshold */
        // ex-woc-thd-z = <0>:  /* extra delta for Z-axis wake-on-change threshold */
        // smooth-len = <5>; /* Number of samples for smoothing */
        // downshift = <25>: /* 
            /* Maximum in-deadzone-interrupts in burst mode before downshift to wake-on change mode.
               Interrupt interval is hardcode to ~20ms, default 25 * ~20ms = 500ms downshift */
        // rpt-dzn-x = <150>; /* x-axis deadzone */
        // rpt-dzn-y = <150>; /* y-axis deadzone */
        // rpt-dzn-z = <150>; /* z-axis deadzone */
    };
};

/ {
  trackball_listener {
    compatible = "zmk,input-listener";
    device = <&spi_mlx90393>;
  };
};
```

Enable the driver config in `<shield>.config` file:

```conf
CONFIG_ZMK_POINTING=y
CONFIG_PINCTRL=y
CONFIG_GPIO=y
CONFIG_SPI=y # if use spi bus
# CONFIG_I2C=y # if use i2c bus
CONFIG_INPUT=y
```
