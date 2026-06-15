# Kaszinomasina — AI Programming Prompt List

## Hardware Context (include in every prompt)
- **Board:** ESP32-S3-DevKitC-1 N16R8 CP2102, MicroPython with Thonny
- **Logic level:** ESP32-S3 GPIO is 3.3V. Do not connect 5V signals directly to ESP32 GPIO.
- **Display:** SSD1306 OLED, 128×32, I²C, 3.3V. Note: SSD1306 is monochrome; 24-bit BMP images must be converted to 1-bit black/white pixels for display.
- **OLED pins:** SDA=GPIO8, SCL/SCK=GPIO9, VCC=3V3, GND=GND
- **Image SD card reader:** SPI, 3.3V, mounted at `/sd` using `sdcard.py` if available. Pins: SCK=GPIO12, MOSI=GPIO11, MISO=GPIO13, CS=GPIO10
- **Audio:** DFPlayer Mini powered from 5V, controlled by UART1 at 9600 baud. ESP32 TX=GPIO17 → DFPlayer RX, ESP32 RX=GPIO18 ← DFPlayer TX. Speaker connects to DFPlayer SPK_1/SPK_2.
- **Matrix keyboard:** 3×3 matrix, rows GPIO4/GPIO5/GPIO6, columns GPIO7/GPIO15/GPIO16
- **Standalone button A:** GPIO21
- **Avoid/reserved pins:** GPIO0, GPIO19, GPIO20, GPIO35–GPIO48 unless explicitly needed. GPIO19/GPIO20 are USB pins. GPIO0 is boot/strapping.
- **Sound SD card:** Still inside the DFPlayer. Folder structure and MP3 files remain unchanged.
- **Picture SD folder:** `/sd/maps/` contains 128×32 24-bit BMP files: `tanariss.bmp` and `ungoroo.bmp`.
-USE THIS REPO and its tools: https://github.com/espressif/esp-idf

---

## Prompt 1 — Project Skeleton & Hardware Init

```
You are writing MicroPython for an ESP32-S3-DevKitC-1 N16R8 CP2102 using Thonny.

Set up the project skeleton for a game called "Kaszinomasina". Create a single main.py file with the following:

1. Import all necessary MicroPython modules: machine, time, os, random, framebuf, struct if needed.

2. Define all pin constants:
   - OLED I2C: OLED_SDA=8, OLED_SCL=9
   - SD card reader SPI: SD_SCK=12, SD_MOSI=11, SD_MISO=13, SD_CS=10
   - DFPlayer Mini UART1: DF_TX=17, DF_RX=18
   - Matrix keyboard rows: ROW0=4, ROW1=5, ROW2=6
   - Matrix keyboard cols: COL0=7, COL1=15, COL2=16
   - Standalone button A: BTN_A=21

3. Define the button label map for the 3×3 matrix. Layout (row, col) → label:
   (0,0)=G  (0,1)=H  (0,2)=I
   (1,0)=J  (1,1)=B  (1,2)=C
   (2,0)=D  (2,1)=E  (2,2)=F

4. Initialize hardware:
   - I2C for SSD1306 OLED at 400kHz
   - SPI for SD card at low speed first, then allow higher speed after mount
   - UART1 for DFPlayer at 9600 baud
   - Matrix row pins as OUTPUT, default HIGH
   - Matrix col pins as INPUT with pull-up
   - Button A as INPUT with pull-up

5. Add placeholder functions for each major module:
   oled_init(), sd_init(), dfplayer_init(), scan_keys(), main_loop().

6. Add a main() entry point that calls all init functions then calls main_loop().

Do not implement the full game logic yet — just the skeleton, constants, and hardware init.
```

---

## Prompt 2 — SSD1306 OLED Display Driver

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the SSD1306 128×32 I²C display driver.

Important: the SSD1306 OLED is monochrome, not color. Any color arguments from old code must be replaced by simple pixel values: 1=white/on, 0=black/off. The 24-bit BMP files should be converted to monochrome when displayed.

Use either:
- MicroPython's built-in ssd1306.py driver if available, or
- include a minimal SSD1306_I2C class in main.py.

Implement these display helper functions:

1. `oled_init()`
   - Initialize I2C on SDA=GPIO8, SCL=GPIO9.
   - Create a 128×32 SSD1306 object.
   - Clear the display.

2. `oled_clear()`
   - Fill screen black and show.

3. `oled_text(text, x, y, invert=False)`
   - Draw ASCII text using the built-in 8×8 font.
   - If invert=True, draw a white rectangle behind the text and draw text in black if possible.

4. `oled_center(text, y, invert=False)`
   - Center text horizontally.
   - Use 8 pixels per character.

5. `oled_menu(title, items, selected_index)`
   - Fit the UI into 128×32.
   - Use at most 4 text rows.
   - Main menu must look like:
     > PLAY
       MAP
       MUSIC
   - Do not use Unicode icons; use ASCII only.

6. `oled_status(line1, line2='', line3='', line4='')`
   - Clear and draw up to four 8-pixel-high text lines.

7. `oled_show_bmp_128x32(path)`
   - Load a 128×32 24-bit uncompressed BMP from the image SD card.
   - Parse the BMP header.
   - BMP rows are usually stored bottom-up and padded to 4-byte boundaries.
   - Convert each RGB pixel to brightness using a simple formula: brightness=(r*30 + g*59 + b*11)//100.
   - If brightness > 127, set OLED pixel white, else black.
   - Call display.show() after drawing.
   - Return False if file is missing or not a supported 128×32 24-bit BMP.

Because the OLED framebuffer is only 128×32 monochrome, it is safe to use framebuf/display buffering.
```

---

## Prompt 3 — SD Card & DFPlayer Drivers

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement two driver modules as classes or grouped functions.

### SD Card driver for picture SD
Use `sdcard.py` if available on the ESP32 filesystem. Mount the SPI SD card at `/sd`.

Pins:
- SCK=GPIO12
- MOSI=GPIO11
- MISO=GPIO13
- CS=GPIO10

Implement:
- `sd_init()` — mounts the SD card at `/sd`. Prints error, shows `SD ERROR` on OLED, and returns False if it fails.
- `sd_list_files(folder, extensions)` — returns a sorted list of filenames in `/sd/<folder>/` matching the given extensions, for example ['.bmp'] or ['.mp3']. Return empty list if folder is missing.

Picture SD structure:
- `/sd/maps/tanariss.bmp`
- `/sd/maps/ungoroo.bmp`

These BMP files are 128×32, 24-bit color BMPs, but they must be displayed on the SSD1306 as monochrome.

### DFPlayer Mini driver
Communicate via UART1 with ESP32 TX=GPIO17 and ESP32 RX=GPIO18 at 9600 baud.

Implement a DFPlayer class with:
- `__init__()` — init UART, wait 1000ms, call reset(), then select_sd().
- `_send(cmd, p1, p2)` — send a 10-byte DFPlayer command frame with checksum.
- `reset()` — command 0x0C, wait 2000ms.
- `select_sd()` — command 0x09, p2=2, wait 500ms.
- `set_volume(v)` — set volume 0–30, clamp value. Command 0x06.
- `play_folder_file(folder, file)` — command 0x0F, p1=folder number, p2=file number.
- `stop()` — command 0x16.
- `loop_track(folder, file)` — play a track and set single-repeat/loop behaviour.

DFPlayer sound SD folder structure remains unchanged:
- Folder 01 = background music tracks, files 0001.mp3, 0002.mp3, etc.
- Folder 02 = funny soundboard clips
- Folder 03 = laugh soundboard clips
- Folder 04 = sad soundboard clips
- Folder 05 = meme soundboard clips
```

---

## Prompt 4 — Keyboard Scanner & Button Debounce

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the keyboard input system.

### 3×3 Matrix scanner
Rows are OUTPUT pins:
- ROW0=GPIO4
- ROW1=GPIO5
- ROW2=GPIO6

Columns are INPUT pins with pull-up:
- COL0=GPIO7
- COL1=GPIO15
- COL2=GPIO16

Scanning: pull one row LOW at a time, read all 3 columns. A pressed key reads LOW on its column.

Button label map by (row, col):
  (0,0)=G  (0,1)=H  (0,2)=I
  (1,0)=J  (1,1)=B  (1,2)=C
  (2,0)=D  (2,1)=E  (2,2)=F

Button A is a standalone INPUT with pull-up on GPIO21. It reads LOW when pressed.

Implement:
- `scan_keys()` — scans the full matrix plus button A. Returns the label of the first pressed key ('A','B','C','D','E','F','G','H','I','J'), or None if nothing is pressed.
- `wait_key(allowed=None)` — blocks until one of the allowed key labels is pressed, or any key if allowed=None. Include 30ms debounce and wait for key release before returning.
- `wait_key_timed(allowed=None, timeout_ms=5000)` — same as wait_key but returns None if timeout_ms elapses with no press. Used for skill check timing.

Button functions:
- A = main action / roll / confirm
- B = navigate up / increase value
- C = navigate down / decrease value
- D = volume up (+3 out of 30)
- E = volume down (-3 out of 30)
- F = back / cancel
- G = soundboard funny
- H = soundboard laugh
- I = soundboard sad
- J = soundboard meme
```

---

## Prompt 5 — Main Menu for 128×32 OLED

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the main menu screen for the 128×32 OLED.

The main menu has 3 options:
1. PLAY
2. MAP
3. MUSIC

Display requirements:
- The OLED has only 4 text rows of 8 pixels each.
- Use ASCII only.
- The menu should look like this:
  > PLAY
    MAP
    MUSIC
- Do not use large titles on this screen because space is limited.
- Highlight the selected item using the `>` cursor. Optionally invert the selected row.
- Selected index starts at 0, PLAY.

Controls:
- B = move selection up, wraps around
- C = move selection down, wraps around
- A = confirm selection and return selected index
- D/E = volume up/down
- G/H/I/J = soundboard

Implement as a function `show_main_menu()` that loops until A is pressed, then returns the selected index 0, 1, or 2.

Implement helpers:
- `handle_volume(key)` — D/E adjust global current_volume, clamp 0–30, step 3, call dfplayer.set_volume(), briefly show `VOL <value>`.
- `handle_soundboard(key)` — G/H/I/J play random files from DFPlayer folders 02/03/04/05. Use hardcoded file counts if reading the DFPlayer SD is not possible.
```

---

## Prompt 6 — Music Select Screen

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the Music Select screen for the 128×32 OLED.

Behaviour:
- The DFPlayer SD is not mounted as a filesystem. Playback uses DFPlayer folder 01 track numbers.
- For display, either:
  1. scan `/sd/music/` on the picture SD for `.mp3` filenames if that mirror folder exists, or
  2. if no display list exists, create generic names: TRACK 1, TRACK 2, TRACK 3, etc.
- Show up to 3 tracks at a time because the screen is tiny.
- Use `>` for the selected item.
- Use ASCII only.

Example screen:
  MUSIC
  > TRACK 1
    TRACK 2
    TRACK 3

Controls:
- B = move selection up
- C = move selection down
- A = confirm: store selected track index, start looping via dfplayer.loop_track(1, track_number), briefly show `PLAYING` and the track name, then return to main menu.
- F = back to main menu without changing track.
- D/E = volume
- G/H/I/J = soundboard

Implement as `show_music_select()`.
```

---

## Prompt 7 — Map Select Screen

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the Map Select screen for the 128×32 OLED.

Behaviour:
- Scan `/sd/maps/` for `.bmp` files using sd_list_files().
- Expected files: `tanariss.bmp` and `ungoroo.bmp`.
- Display a compact list using friendly names without extension, uppercase if needed.
- Show up to 3 map options at a time.

Example screen:
  MAP
  > TANARISS
    UNGOROO

Controls:
- B = move selection up
- C = move selection down
- A = confirm: store the selected filename in global selected_map. Call `oled_show_bmp_128x32('/sd/maps/' + selected_map)` to preview it for 1500ms, then return to main menu.
- F = back to main menu without changing map.
- D/E = volume
- G/H/I/J = soundboard

The selected_map persists as the background image for game screens until changed here.

Implement as `show_map_select()`.
```

---

## Prompt 8 — Game Setup: Player Count & Game Value

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the two game setup screens shown after PLAY is selected from the main menu.

Because the OLED is only 128×32, each screen must use short text.

### Screen 1: Player Count
Display:
  PLAYERS
  <current value>
  B+ C-
  A=OK

Default value: 2. Minimum: 2. Maximum: 10.
- B increases by 1
- C decreases by 1
- A confirms and proceeds
- D/E = volume
- G/H/I/J = soundboard

### Screen 2: Game Value
Display:
  VALUE
  <current value>
  B+500 C-500
  A=OK

Default value: 1000. Minimum: 500. Maximum: 20000. Step: 500.
- B increases by 500
- C decreases by 500
- A confirms and calls start_game(player_count, game_value)
- D/E = volume
- G/H/I/J = soundboard

Implement:
- `setup_player_count()` → returns int
- `setup_game_value()` → returns int
```

---

## Prompt 9 — Skill Check for 128×32 OLED

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the skill check mini-game as `run_skill_check(current_max)`.

Trigger condition: skill check only runs when current_max <= 10.

The old colored-circle design does not work on the SSD1306 monochrome OLED. Use text and a simple bar instead.

Behaviour:
1. Clear the OLED and show:
   WAIT
   [    ]
   DO NOT
   PRESS

2. Wait a random time between 1000ms and 3000ms. During this time, poll scan_keys(). If A is pressed early:
   - Show:
     EARLY!
     NO BONUS
   - Wait 800ms.
   - Return current_max unchanged.

3. Show:
   READY
   [####]
   PRESS A

4. Start timer immediately when READY appears.
   Use wait_key_timed(allowed=['A'], timeout_ms=5000).

5. If A is pressed in less than 300ms:
   - new_max = ceil(current_max * 1.3), minimum 1.
   - Show:
     FAST!
     MAX <new_max>
   - Wait 1000ms.
   - Return new_max.

6. If A is pressed between 300ms and 700ms:
   - Show:
     OK
     NO BONUS
   - Wait 800ms.
   - Return current_max unchanged.

7. If no press within 5000ms or press is later than 700ms:
   - Show:
     SLOW
     NO BONUS
   - Wait 800ms.
   - Return current_max unchanged.

Return value: the possibly modified max value to use for this single roll only.
```

---

## Prompt 10 — Core Game Loop

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the core game loop as `start_game(player_count, game_value)`.

State:
- players = list of player numbers, e.g. [1, 2, 3, ...]
- original_value = game_value
- current_max = game_value
- current_player_index = 0

Each turn:
1. Draw the selected map background if selected_map is set:
   `oled_show_bmp_128x32('/sd/maps/' + selected_map)`
   Otherwise clear the OLED.

2. Draw compact overlay text. Since the screen is tiny, use this format:
   P<N> TURN
   MAX <current_max>
   A=ROLL

If drawing text on top of the BMP is unreadable, clear the OLED instead of using the image background for text-heavy screens.

3. Wait for A button press. While waiting, D/E volume and G/H/I/J soundboard must work.

4. If current_max <= 10, call run_skill_check(current_max). Use the returned value as the effective max for this roll only. Do not update current_max yet.

5. Roll:
   result = random.randint(1, effective_max)

6. Update:
   current_max = result

7. Display roll result:
   P<N> ROLLED
   <result>
   Wait 2000ms.

8. If result == 1:
   - Show:
     P<N> OUT
   - Wait 2500ms.
   - Remove player N from the players list.
   - If one player remains:
     Show:
       WINNER
       P<N>
       A=MENU
     Wait for A, then return to main navigation.
   - If more than one player remains:
     Reset current_max to original_value.
     Continue with remaining players.

9. If result != 1:
   - Advance current_player_index to the next player, wrapping around.
   - Continue the loop.
```

---

## Prompt 11 — Wiring It All Together & Startup

```
Continue building main.py for Kaszinomasina on ESP32-S3 MicroPython.

Implement the top-level startup sequence and wire all screens together.

Global state variables:
- selected_map = None
- selected_music_track = None
- current_volume = 15
- dfplayer = None
- oled = None

Startup sequence in main():
1. Call oled_init(). Clear display and show:
   KASZINO
   LOADING

2. Call sd_init(). If it fails, show:
   SD ERROR
   CHECK CARD
   then halt.

3. Call dfplayer_init(). Set volume to current_volume.

4. Show splash for a total of about 2000ms, then transition to main menu.

Main navigation loop:
- call show_main_menu(), which returns 0, 1, or 2.
  - 0 → setup_player_count() → setup_game_value() → start_game(player_count, game_value)
  - 1 → show_map_select()
  - 2 → show_music_select()
- After any sub-screen returns, loop back to show_main_menu().

Entry point at the bottom:
    try:
        main()
    except Exception as e:
        print("CRASH:", e)
        try:
            oled_status("CRASH", str(e)[:16])
        except:
            pass
```

---

## Notes for the AI programmer

- This project is now for ESP32-S3 MicroPython in Thonny, not Raspberry Pi Pico W.
- The SSD1306 OLED is 128×32 and monochrome. There are only four 8-pixel text rows, so keep all UI text very short.
- Use ASCII only. Do not use emoji or Unicode icons.
- The menu format should be compact, for example:
  > PLAY
    MAP
    MUSIC
- 24-bit BMP map files must be converted to monochrome when displayed on the SSD1306.
- The picture SD card is mounted at `/sd` and contains `/sd/maps/tanariss.bmp` and `/sd/maps/ungoroo.bmp`.
- The DFPlayer has its own separate sound SD card. It is not mounted by the ESP32.
- DFPlayer folder/file numbers are 1-based.
- The sound SD remains unchanged:
  - 01 = background music
  - 02 = funny
  - 03 = laugh
  - 04 = sad
  - 05 = meme
- No async/threading is needed. Use polling loops.
- D/E volume and G/H/I/J soundboard should work from menu screens and during the roll wait screen.
- Skill check only triggers when current_max <= 10 and modifies only the effective max for that single roll.
- Avoid GPIO0, GPIO19, GPIO20, GPIO35–GPIO48 unless the hardware design is intentionally changed.
