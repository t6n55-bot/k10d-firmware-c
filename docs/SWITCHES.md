# K10D emulator: SWITCH TEST map and emulator setup (2026-11-21)

Emulator: the qemu-frn fork (Penth4x/qemu, branch `frn-publish`). Needed one fix to build:
`configs/devices/frn-softmmu/default.mak` was missing and was added.
Run: `qemu-system-frn -M k10d -bios FWDC162B.BIN -display sdl` (about 3 s to boot; Language -> Initial Setting -> Date Adjust).

## Reaching the firmware's debug menu in the emulator
1. SD image: a FAT16 256 MB image; a root-dir file can be written into it without mounting.
2. Put `MODSET.TXT` with `[OPEN_DEBUG_MENU]` on it. Boot with `FRN_SD=<img>` and hold MENU from power-on (QEMU monitor `sendkey m 9000`).
   -> works in the emulator on the stock 1.31 image. Per the qemu-frn author this only works because the emulator models the SD card door as open (a peripheral quirk); whether it works on real hardware with a closed door is unverified.
3. Debug menu pages: 1 DEBUG MODE/CARDDOOR/WRITE CARD VNDR/SCRIPT EN/USR DAT CLEAR/DUMMY RELEASE (OK writes flash!),
   2 SWITCH TEST/RELEASE AGING/LENS COM CHECK/AF PINT DISP/DSP ADJ DAT READ+WRITE, 3 EEPROM LOAD/SAVE/READ-WRITE, CPU ROM SAVE, DSP ROM SAVE.
   Keys must be held ~400 ms and spaced ~1.5 s (`sendkey down 400`), or the firmware drops them.

## 'A' button record (9 bytes, active-low). Found by clearing each bit and diffing the SWITCH TEST screen.
| byte | bit | switch | key in the emulator |
|---|---|---|---|
| 0 | 0x01 | SWS shutter half-press | S |
| 0 | 0x02 | SWR shutter full press | (not mapped: wedges, see below) |
| 0 | 0x08 | AFSTART | A |
| 0 | 0x20 | CARDDET | - |
| 2 | 0x01 | AEL | E |
| 2 | 0x02 | GREEN | G |
| 2 | 0x04 | PRV | W |
| 2 | 0x08 | DOOR (card door) | C |
| 2 | 0x10 | INFO | I |
| 2 | 0x20 | DISP (the old "V" guess) | D / V |
| 2 | 0x80 | XV or AVSET (both light together) | Z |
| 3 | 0x01/02/04/08/10 | 4WR/4WL/4WD/4WU/4WOK | arrows, Enter |
| 3 | 0x80 | MENU | M, Esc |
| 4 | 0x01 | FLPOP (flash pop-up) | U |
| 4 | 0x80 | AEB | - |
| 5 | 0x01 | RAW | R |
| 5 | 0x04 | SR | X |
| 5 | 0x08 | FUNC | F |
| 5 | 0x40 | ERASE | Backspace/Delete |
Value-type fields (not switches): byte 1 -> TILT; bytes 3 (0x20-0x40), 4 (0x02-0x20), 5 (0x10-0x20) -> AFMD / MODE / AEMD; byte 6 (0x10,0x20) -> AC; bytes 7-8 -> FDL/RDL dial counters.
Not yet sorted out: exact bit meaning of those value fields (mode dial, AF mode, drive mode, dials).

## Behaviour notes
- SWITCH TEST shows a press one record late; green = pressed at some point since entering, blue = pressed now.
- Pressing SWR (full release) leaves the firmware wedged (SWR stays "pressed", later records ignored). Likely it waits for the
  control CPU to finish a release sequence, which this emulator does not model.
- `FRN_REC_FILE=<file>` (added): file holding 18 hex digits is read every 100 ms and posted as the 'A' record. Used for the bit scan.

## Fake control-CPU investigation (2026-11-21): why SWR (shutter full press) wedges SWITCH TEST
Setup: nav to SWITCH TEST, `FRN_REC_FILE` record `fd ff..` (SWR) then release; test = do later key presses and see if the screen follows
(note: the screen shows each press one record late, so always add a trailing press). Stub log: `FRN_DSP_LOG=1`.
- On SWR the main CPU sends CPUCOM `Cmd:02` payload `01 00` (twice). `Cmd:02` payload `20 00` / `00 00` is the power-off/save path.
  Stack at send (`FRN_DSP_STACK=1`): FUN_10465958 (SPI byte transfer) <- FUN_1046650e (CPUCOM task, per request) <- FUN_104664a8 (task loop).
  The message builder for Cmd 2 is FUN_10463d82 (payload = 2 bytes from a vtable getter, 0x1070001 event tag).
- Task 35 (debug UI) then sits in FUN_1046f9e4 (the firmware's dummy-release / RELEASE TIMES routine): loop
  `read status record; if FUN_104603b2(rec,4) != 0 break; sleep(10)`. Item 4 = record byte0 & 0x80 (table 0x105a6638, 8-byte entries).
  The record is the object at settings-manager +0x80 (class 0x105a6eac), filled by status letter **'@' (4-byte payload)**
  (dispatcher FUN_1045e83a, case 0x0f). The stub never sent '@'. `FRN_DSP_BOOT=HB@` delivers it (default payload ff) and the task then
  leaves that loop (a new `Cmd:79` appears), but the screen still does not follow later keys even 150 s later, so more waits remain.
- Tried without effect: reply byte for Cmd 2 (`FRN_DSP_R02`=07/01/00), queued status letters after the request (`FRN_DSP_ON02`=B/X/C), first byte of H/X payloads.
- RETRACTED (2026-11-21): an earlier note said a "post-exposure status buffer" at 0x801EDB46 needs bytes +1/+3/+5 = 1.
  Wrong: that test pressed record `de..` (clears 0x01 SWS **and 0x20 CARDDET**), not SWR (0x02). What followed was
  card-removal handling (unmount via 0x1047bd92, "Card not formatted"). 0x801EDB46 is the storage object's mount
  state (0x80532F20+0x48..). Use `fe..` (S1) then `fc..` (S1+S2) for a real full press.
- Real S2 in shooting mode (`fc..`): the main CPU sends no CPUCOM command at all (S1 alone sends Cmd 05/2f/03).
  None of the 21 other status letters with 0xff payloads (injected with `FRN_MSG_FILE`) changed that.
- `FRN_MSG_FILE=<path>` (added): when the file changes, each line `<letter> [hex payload]` is queued as a status
  message (unset bytes 0xff). Runtime listener table (settings manager 0x8054d7bc+0x1ac): only C, I -> 0x805329c0;
  H, B -> mode ctrl 0x804a7e24; X -> 0x804a9828; A -> screen mgr 0x804a9940. The other letters are stored as
  records and polled (e.g. '@' by 0x104f2482).
- Tools: full decompile of the main image: an ELF32 wrapper around the main image (file 0 -> 0x10000000) then `REA_GHIDRA_LANGUAGE_ID=fr60:BE:16:default rea decompile-all X.elf --output NEWDIR`
  (output dir must not exist). `FRN_TASKS=<file>` + monitor `quit` writes per-task parking spots (task 35 normal park 0x10459736 rp 104861e8).
## Shutter release handshake (found 2026-11-21, emulator)
The main CPU does not start a capture on its own; the control CPU signals it over port 0x50000106 + external INT:
1. S1 (`fe..`) arms external INT ch0 (ENIR 0x39: 0x02 -> 0x03). INT ch k = vector 30+k (ch1 = vec31 = CPUCOM attention).
2. Release = control CPU raises input line 0x01 of 0x50000106 and pulses INT0 (vec30). ISR -> event 0x40001 -> capture
   task (task 7, object 0x8052c328) flag bit 0x20 -> it checks the line (`call_with_second_field`, must read 1) ->
   event 0xd0000f -> `execute_capture_sequence` 0x1041bd7e. Line low at INT0 = abort (0xd0000d).
3. It runs the drive sequencer 0x1040fbc8 (19 reload-timer ticks on ch2, GPIO flag 0x44), then waits on
   0x805329bc+0x1c bit 1: set by status letter 'C' (0xa0001). Then it waits for line 0x01 to drop, then for bit 2
   of the same flag: status letter 'I' (0xa0010).
4. Then: configure_exposure_state (0xd00002), parameters over "ch2" (sensor timing generator serial link),
   `measure_and_calibrate_sensor_timing`, `configure_sensor_readout_channels` (4 DMA channels at capture obj
   +0xd4/+0xd8/+0xdc/+0xe0), and waits 500 ms twice (`wait_on_object_with_timeout`, flag +0xcc) for IRQ source
   0x8000001a = **vector 32** (hw_irq_source_demux case 0x10). Not modelled yet -> times out, no file written.
Driving it: `FRN_MSG_FILE` lines `+01` / `-01` (hold / release 0x50000106 input bits), `!30` (raise vector once,
sets EIRR bit for 30..37), letters as before. Sequence: `+01`,`!30` ; `C` ; `-01` ; `I`. Script: a small shell wrapper around these steps.
Emulator fixes needed on the way (op_helper.c): `frn_ic` only counted under FRN_FT (reload timer never expired
without tracing); reload timer TMCSR bit0 is TRG (write-only trigger), the old model treated it as a run level so
every UF-clear write retriggered the one-shot (`FRN_TMR_OLD=1` = old rule); FRN_FT_GO traced from boot.

### After readout: pipeline kick, pre-process, develop (2026-11-21, emulator)
- First-run setup: 3x OK (Language, Initial Setting, Date Adjust) reaches shooting mode (three Enter presses).
- Capture events on the mode controller: d00002, d00003, d00004, **d00006**, d0000c, d00001, d0000e (d00006 is the normal
  end of `execute_capture_sequence`, not an abort).
- Image pipeline = task 21 `dispatch_event_queue_items` 0x105143f8 (object 0x804aa2ec, eventflag +0x34c). Bits:
  0x01 kick (`set_flag_at_offset_0x34c`, from `configure_exposure_state`), 0x08 gate, 0x10 init, 0x20/0x40 replies to 0x800003/4.
  The kick happens at the START of a capture, before the ring entry (`alloc_first_ring_entry`) exists, so shot N is
  processed when shot N+1 starts (or on mode exit, `release_mode_entry_objects` -> 0x200 -> `process_mode_event_flags_10418c04`).
  In the emulator: fire the release sequence twice.
- Pre-process engine 0x400d0000: src 0x400d0014 = sensor buffer A (0x090d1000), 0x400d0010/12 = 0x520 x 0x7a0,
  pitch 0x400d0020 = 0xf60 (3936), dst 0x400d0024/28/2c, 0x48..4e = 0x100 (gains). Start = 0x400d0004 bit15.
  IRQ vec42 (`check_and_acknowledge_irq_source_104fae56`): 0x400d0006 bit15 (W1C) and 0x400d0004 bit15 -> 0x80000014.
- imgproc op C (existing model) then runs 3904x2624 (0xf40 x 0xa40) from 0x090d1000 to 0x0eacee00.
- Develop = task 10 (unit 257/258, `build_and_send_struct` -> `configure_sensor_readout_params_104f3de0`): loads tables
  into 0x40103000-0x4010cfff (3x2048, 256, 4096, 4096 entries), start = 0x40100000 = 1 + 0x40128100 bit14.
  Done IRQ = vec34 (demux case 0x12 `clock_tick_post_a`): status 0x40128004 & 0x301 (W1C) & enable 0x40128002.
  Handler `set_two_flags_on_object_104f3d8e`: bit 0x100 -> 0x200001(1), 0x200 -> 0x200001(0). **Use 0x100**; 0x200 makes
  task 10 restart develop forever.
- After develop: rc0/rc1 resize converters make 640x428 and 320x428 (review/thumbnail). No JPEG/card write yet.
- Emulator: hw/frn/k10d.c `frn_engine_done` (FRN_ENGINE_DONE=0 off, FRN_ENG34_BITS=mask, default 0x100) sets the done
  status and raises vec42/vec34 at the start writes; vec32 (readout done) still needs FRN_FIRE_VECS=32.

- Idle in shooting mode: S1/S2 send NO new CPUCOM commands; the wedge is specific to the debug screens' dummy release.
Experiment knobs added in k10d_cpucom.c: FRN_DSP_R02, FRN_DSP_ON02; in op_helper.c: FRN_DSP_STACK (stack dump on Cmd 2).
