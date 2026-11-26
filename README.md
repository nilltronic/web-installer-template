# web-installer-template
Template for a web installer for esp8266 projects

# ESP8266 Web Flasher

Flash or update an ESP8266 project straight from your browser. No Arduino IDE, no extra software, just a USB cable.

This repo is a template. It works for **any** ESP8266 project (Wemos/LOLIN D1 mini, NodeMCU, ESP-01 and so on). To use it for your own project you swap in your own `.bin` files and edit one small `manifest.json`.

**Flasher page:** `https://<your-username>.github.io/<repo-name>/install.html`

---

## Part 1: For users (you just want to flash your device)

1. Plug the device into your computer with a **data** USB cable. Some cables only charge and will not work.
2. Open the flasher page in **Chrome or Edge** on a desktop or laptop. Firefox, Safari and phones do not support this.
3. Click the install button.
4. Pick your device's port in the pop-up (often "USB-SERIAL CH340" or "CP210x") and click **Connect**.
5. Wait until it says it has finished. Do not unplug it before then.

If no port shows up, install the CH340 (or CP210x) USB driver for your operating system and try again. If the connection still fails, hold the board's **FLASH** (GPIO0) button while plugging it in, then retry.

---

## Part 2: For maintainers (making the files for your own project)

The flasher needs up to two files, both made in Arduino IDE:

| File | What it is | Always needed? |
|---|---|---|
| `firmware.bin` | Your compiled sketch | Yes |
| `littlefs.bin` | The contents of your sketch's `data` folder | **Only if your sketch has a `data` folder** |

### Quick decision: do I need `littlefs.bin`?

Look inside your sketch folder (the folder with your `.ino` file).

- **There is a folder called `data` with files in it** (web pages, images, config and so on): you need both files. Do Steps 1 to 5.
- **There is no `data` folder**: you only need `firmware.bin`. Do Step 1, then go to Step 6 and use the **firmware only** manifest. Skip everything about LittleFS, including the offset and size.

What happens on the device in the "no data folder" case: flashing firmware alone leaves whatever is already in the device's filesystem untouched. Many sketches create their own files at runtime, so this is fine.

---

### Step 1: Get `firmware.bin`

1. Open your sketch in Arduino IDE.
2. In the **Tools** menu, select your **Board** and your **Flash Size** (for example `4MB (FS:1MB OTA:~1019KB)`). Write the Flash Size down. You will need it in Step 3.
3. Click **Sketch → Export Compiled Binary** and wait for it to finish.
4. Click **Sketch → Show Sketch Folder**.
5. Find the `.bin` file:
   - **Arduino IDE 2.x:** open the `build` folder, then the single folder inside it (named after your board, like `esp8266.esp8266.d1_mini`). The file is `YourSketch.ino.bin`.
   - **Arduino IDE 1.x:** the file sits straight in the sketch folder, named like `YourSketch.ino.d1_mini.bin`.
6. Copy it into this repo and rename it **`firmware.bin`**.

Ignore the `.elf` and `.map` files. They are not needed.

Can't find it? Turn on **File → Preferences → Show verbose output during: compilation**, export again, and read the build path near the end of the output panel.

---

### Step 2: Install the LittleFS tool (data folder projects only)

The `data` folder must be named exactly `data` and sit inside your sketch folder, next to the `.ino` file.

Arduino IDE 2.x has no built-in "make filesystem image" button for ESP8266. The easiest fix is the free **arduino-littlefs-upload** plugin by Earle Philhower:

1. Download the `.vsix` file from the plugin's GitHub releases page: https://github.com/earlephilhower/arduino-littlefs-upload/releases
2. Copy it into the plugins folder (create the folder if it does not exist):
   - **Windows:** `C:\Users\<your-username>\.arduinoIDE\plugins\`
   - **Mac / Linux:** `~/.arduinoIDE/plugins/`
3. Restart Arduino IDE.

It needs **Arduino IDE 2.2.1 or newer**. On Arduino IDE 1.x, or if you'd rather not install a plugin, use the command-line method in Step 4B instead.

---

### Step 3: Find your filesystem offset and size (data folder projects only)

This is the part that trips everyone up. `littlefs.bin` has to be written to one exact place in the chip's flash, and the place depends on the **Flash Size** option you chose in the Tools menu. Get it wrong and the filesystem can overwrite your program.

**Common 4MB boards (most D1 minis and NodeMCUs):**

| Tools → Flash Size | Filesystem size (bytes) | Offset (hex) | Offset (decimal, for `manifest.json`) |
|---|---|---|---|
| `4MB (FS:1MB ...)` | `1024000` | `0x300000` | `3145728` |
| `4MB (FS:2MB ...)` | `2072576` | `0x200000` | `2097152` |
| `4MB (FS:3MB ...)` | `3055616` | `0x100000` | `1048576` |

**Any other board or flash size (1MB, 2MB, 16MB, or you're not sure):** don't guess. Ask the board itself. Add this to your sketch temporarily, upload it, and open the Serial Monitor:

```cpp
extern "C" uint32_t _FS_start;
extern "C" uint32_t _FS_end;

void setup() {
  Serial.begin(115200);
  delay(1000);
  uint32_t offset = (uint32_t)&_FS_start - 0x40200000;
  uint32_t size   = (uint32_t)&_FS_end - (uint32_t)&_FS_start;
  Serial.printf("FS offset: 0x%X (decimal %u)\n", offset, offset);
  Serial.printf("FS size:   %u bytes\n", size);
}

void loop() {}
```

This pattern comes straight from how the ESP8266 core itself finds its filesystem, but run it on your board and check the numbers look sensible (for a 4MB board with `FS:1MB` you should see `0x300000` and `1024000`). Write down both numbers, then remove the temporary code.

**Always use the same Flash Size setting** when you build the firmware, when you build the filesystem image, and when you work out these numbers.

---

### Step 4: Get `littlefs.bin` (data folder projects only)

#### Step 4A: With the plugin (Arduino IDE 2.2.1 or newer)

1. Make sure your board and **Flash Size** are selected in the Tools menu (same as Step 1).
2. Open the command palette: **Ctrl + Shift + P** (Mac: **Cmd + Shift + P**).
3. Type and choose **Build LittleFS image in sketch directory**.
4. The image is saved in your **sketch folder** as **`mklittlefs.bin`**, and its location is printed in the output panel. No digging through temp folders.
5. Copy it into this repo and rename it **`littlefs.bin`**.

Do not choose "Upload LittleFS to Pico/ESP8266/ESP32" for this. That one sends the image to your board but does not leave you a file you can use here.

#### Step 4B: Without the plugin (any Arduino IDE version)

`mklittlefs` is already on your computer. It was installed with the ESP8266 board package.

1. Find it:

| System | Folder |
|---|---|
| Windows | `C:\Users\<your-username>\AppData\Local\Arduino15\packages\esp8266\tools\mklittlefs\<version>\` |
| Mac | `~/Library/Arduino15/packages/esp8266/tools/mklittlefs/<version>/` |
| Linux | `~/.arduino15/packages/esp8266/tools/mklittlefs/<version>/` |

On Windows, `AppData` is hidden. Paste the path into File Explorer's address bar. `<version>` is the one folder inside.

2. Open a terminal in that folder and run this, replacing the data folder path and the `-s` number with your **filesystem size from Step 3**:

Windows:
```
mklittlefs.exe -c "C:\path\to\YourSketch\data" -p 256 -b 8192 -s 1024000 littlefs.bin
```

Mac / Linux:
```
./mklittlefs -c /path/to/YourSketch/data -p 256 -b 8192 -s 1024000 littlefs.bin
```

3. `littlefs.bin` appears in the folder you ran it from. Copy it into this repo.

If the command complains that the files don't fit, your `data` folder is bigger than the filesystem. Shrink it, or pick a Flash Size with a bigger FS.

#### Sanity check before moving on

`littlefs.bin` should be the **same size** as the filesystem size from Step 3 (for example 1,024,000 bytes for `FS:1MB`). If it isn't, something doesn't match: stop and re-check the Flash Size setting rather than flashing it.

---

### Step 5: Put the files in the repo

```
your-repo/
  install.html
  manifest.json
  firmware.bin
  littlefs.bin      <- only if you have a data folder
```

---

### Step 6: Edit `manifest.json`

`manifest.json` tells the flasher which files to write and where.

 If you have a data folder, copy manifest-with-filesystem.example.json over manifest.json and set your own offset.

#### A. Firmware only (no `data` folder)

```json
{
  "name": "My Project",
  "version": "1.0.0",
  "new_install_prompt_erase": false,
  "builds": [
    {
      "chipFamily": "ESP8266",
      "parts": [
        { "path": "firmware.bin", "offset": 0 }
      ]
    }
  ]
}
```

#### B. Firmware and filesystem (with a `data` folder)

```json
{
  "name": "My Project",
  "version": "1.0.0",
  "new_install_prompt_erase": false,
  "builds": [
    {
      "chipFamily": "ESP8266",
      "parts": [
        { "path": "firmware.bin", "offset": 0 },
        { "path": "littlefs.bin", "offset": 3145728 }
      ]
    }
  ]
}
```

`3145728` is the decimal offset for `4MB (FS:1MB)`. **Replace it with the decimal number from your Step 3.**

#### What to change for your project

| Field | Change it to |
|---|---|
| `name` | Your project's name |
| `version` | A new number each release |
| `offset` of `littlefs.bin` | Your decimal offset from Step 3 |
| `chipFamily` | Leave as `ESP8266` (ESP8285 boards also use this) |

#### Rules that break things

- **No comma after the last item** in a list. One stray comma makes the whole file invalid and the Install button will fail.
- Offsets must be **plain decimal numbers**. JSON does not accept `0x300000`. Convert hex to decimal with the Windows Calculator in Programmer mode, or use the table in Step 3.
- No filesystem means **delete the whole `littlefs.bin` line** (and the comma on the line above it). Don't leave it pointing at a file that doesn't exist.
- `new_install_prompt_erase` set to `true` offers users a tick-box to wipe the device first. That also deletes saved settings, so most projects leave it `false`.

Paste the finished file into any online "JSON validator" to check it.

#### Also check `install.html`

If you reused this repo from another project, read the text on the page. Remove or reword anything that mentions things your project doesn't have, such as a "built-in frame set", or a filesystem update when you only ship firmware.

---

### Step 7: Publish on GitHub

1. Commit the files and push to `main`.
2. In the repo go to **Settings → Pages**. Set the source to **Deploy from a branch**, branch `main`, folder `/ (root)`, and save.
3. After a minute or two the flasher is live at `https://<your-username>.github.io/<repo-name>/install.html`.

### Releasing an update

1. Repeat Step 1 (and Step 4 if your `data` folder changed).
2. Overwrite `firmware.bin` (and `littlefs.bin`).
3. Raise `version` in `manifest.json`.
4. Commit and push. Hard-refresh the page (Ctrl + Shift + R) because browsers cache the old files.
5. **Test on a real board** before telling anyone.

Tip: flashing `littlefs.bin` replaces the device's whole filesystem, which wipes saved settings such as WiFi details. If you want routine updates to keep user settings, ship a second manifest (for example `manifest-update.json`) with firmware only, and keep the full manifest for first-time installs or when `data` changed.

---

## Troubleshooting

| Problem | Try this |
|---|---|
| "Your browser doesn't support this" | Use desktop Chrome or Edge |
| No port appears | Different USB cable (must carry data), install the CH340 or CP210x driver, try another USB port |
| Install button does nothing or errors | Check `manifest.json` for a trailing comma or a wrong file name |
| Connection fails | Unplug, hold FLASH (GPIO0) while plugging in, retry |
| Flashes fine, but files are missing or the device acts odd | The Flash Size used for the firmware, the filesystem image and the offset don't all match (see Step 3) |
| Can't find the `.bin` after compiling | Use **Export Compiled Binary** and **Show Sketch Folder** (Step 1), and **Build LittleFS image in sketch directory** for the filesystem (Step 4A) |

A failed or interrupted flash will not permanently brick an ESP8266. Reconnect and run it again.

---

## Credits

Built on [ESP Web Tools](https://esphome.github.io/esp-web-tools/). Filesystem images are built with `mklittlefs`, and the optional IDE 2.x plugin is [arduino-littlefs-upload](https://github.com/earlephilhower/arduino-littlefs-upload) by Earle F. Philhower, III.
