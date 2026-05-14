<img width="5712" height="2022" alt="IMG_6045" src="https://github.com/user-attachments/assets/35ce1594-2782-470f-95b6-669a7984a359" />

## How to change the image on the right hand peripheral nice!view screen

Using replacing the image on the nice!view of the right half of Corne 42 as an example.

### Process the image

1. First, choose an image and crop it to a 1:2 aspect ratio.
2. Use [this tool](https://observablehq.com/@rockysweet/convert-image-to-one-bit) to convert the
image to 1-bit. First, we set the image width to the maximum, which will improve the final image
quality. Then choose an appropriate dithering method and save the resulting 1-bit image.
3. Rotate the 1-bit image 90 degree clockwise, then resize it to 140x68 pixels.
4. Finally, use the [LVGL Image Converter](https://lvgl.github.io/lv_img_conv/) to convert it into
a C array. Set the "Color format" to `CF_INDEXED_1_BIT` and the "File name(s)" should be a valid
C identifier as it will be used as the variable name for this array.

[ResizePixel](https://www.resizepixel.com/) is a good option to crop, rotate and resize images.

### Create the ZMK module

1. First, use [ZMK module template](https://github.com/zmkfirmware/zmk-module-template) to create
a module. Give the repository a name, for example, zmk-nice-view-custom
2. Next, copy the nice_view shield (`zmk/app/boards/shields/nice_view`) from ZMK (tag v0.3, the current
release version) to `zmk-nice-view-custom/boards/shields/`.
3. Configure Zephyr module file (`zephyr/module.yml`). For detailed explanations of each property,
refer to the [ZMK doc](https://zmk.dev/docs/development/module-creation#zephyr-module-file).
4. Rename the shield. For example, if we want to rename it to nice_view_custom, the following parts
need to be modified:

    * The folder name `boards/shields/nice_view` -> `boards/shields/nice_view_custom`.
    * These files names inside that folder:

        `nice_view.conf` -> `nice_view_custom.conf`
        `nice_view.overlay` -> `nice_view_custom.overlay`
        `nice_view.zmk.yml` -> `nice_view_custom.zmk.yml`

    * Inside `nice_view_custom.zmk.yml`, update the `id` and perhaps the `name` (though not
       strictly necessary).
    * Inside `Kconfig.shield`, update line 4 and line 5, then whatever we change the config name for
       line 4 to be, we want line 4 of `Kconfig.defconfig` to match it.

        Kconfig.shield
        ```
        # Copyright (c) 2022 The ZMK Contributors
        # SPDX-License-Identifier: MIT

        config SHIELD_NICE_VIEW_CUSTOM   # line 4
            def_bool $(shields_list_contains,nice_view_custom)   # line 5
        ```

        Kconfig.defconfig
        ```
        # Copyright (c) 2023 The ZMK Contributors
        # SPDX-License-Identifier: MIT

            if SHIELD_NICE_VIEW_CUSTOM   # line 4
        ```

5. Finally, add the image. Open the converted C array file and copy everything **AFTER** this block
(shown below) and paste it at the end of the `boards/shields/nice_view_custom/widgets/art.c` file.

    ```c
    #ifndef LV_ATTRIBUTE_MEM_ALIGN
    #define LV_ATTRIBUTE_MEM_ALIGN
    #endif
    ```

    Then in the array `<image file name>_map[] = { ... }`, replace the two color index lines at the
    very top with the following copied from "mountain or "balloon" images

    ```c
    #if CONFIG_NICE_VIEW_WIDGET_INVERTED
        0xff, 0xff, 0xff, 0xff, /*Color of index 0*/
        0x00, 0x00, 0x00, 0xff, /*Color of index 1*/
    #else
        0x00, 0x00, 0x00, 0xff, /*Color of index 0*/
        0xff, 0xff, 0xff, 0xff, /*Color of index 1*/
    #endif
    ```

    Next, open `boards/shields/nice_view_custom/widgets/peripheral_status.c` file and add a new line
    near the top to declare the image

    ```c
    LV_IMG_DECLARE(balloon);
    LV_IMG_DECLARE(mountain);
    LV_IMG_DECLARE(<image file name>); // new line
    ```

    Finally, replace this line `lv_img_set_src(art, random ? &balloon : &mountain);` with
    `lv_img_set_src(art, &<image file name>);`

👉 **For a concreat example, refer to [my ZMK module](https://github.com/rockyzhang24/zmk-nice-view-zelda) that contains a custom shield to display a Zelda image on the peripheral.**

### Build and flash the firmware

Open the `config/west.yml` file in the ZMK config and add the module to it.

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: rockyzhang24                        # new entry
      url-base: https://github.com/rockyzhang24 #new entry
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.3
      import: app/west.yml
    - name: zmk-nice-view-custom                # new entry, the repository name of the module
      remote: rockyzhang24                      # new entry
      revision: main                            # new entry
  self:
    path: config
```

Modify `build.yaml` to display the image on the peripheral

```yaml
---
include:
  - board: nice_nano_v2
    shield: corne_left nice_view_adapter nice_view
    snippet: studio-rpc-usb-uart
  - board: nice_nano_v2
    shield: corne_right nice_view_adapter nice_view_custom  # custom shield
```

Rebuild the firmware, reflash it, and ENJOY!

## References:

* https://www.reddit.com/r/ErgoMechKeyboards/comments/15t3o6k/custom_art_on_niceview_displays/
* https://github.com/GPeye/urchin-peripheral-animation
