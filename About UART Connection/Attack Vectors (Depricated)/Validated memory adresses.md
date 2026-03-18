

- - -
# Power Management Controller (PMC)

### `0x7000E400` — PMC Base Address

 **Use:** The starting point for all Power Management operations.

### `0x7000E450` — PMC_SCRATCH0

- **Use:** undefined but i used it as a Data Safe.
    
- **Example:** When the USB connection dies, store 4 bytes of  data here.
    
- **C Code exampple:**
    
 ```c
    // Store a 4-byte value from IRAM into the scratch register
    uint32_t leaked_data = *(volatile uint32_t *)0x40001000;
    *(volatile uint32_t *)0x7000E450 = leaked_data;
 ```
    

### `0x7000E414` — PMC_CNTRL (Reset Control)

- **Use:** The Reboot Trigger.
    
- **Bit 4:** Setting this bit triggers a Warm Reset.
    
- **Example:** use this immediately after writing to `SCRATCH0` to force Jibo back into RCM mode.
    
- **C Code Implementation:**

```c
    // Trigger Warm Reset to return to RCM mode
    *(volatile uint32_t *)0x7000E414 |= (1 << 4);
```

# Timer and Watchdog (TMR/WDT)

These addresses allow us to cut the system’s security features from what i can tell so they don't interfere with the payload.

### `0x6000501c` — TMR_WDT_RESTART

- **Use:** The Watchdog cut.
    
- **Mechanism:** The Tegra K1 has a timer that reboots the device if it doesn't receive a clock signal within a few seconds.
    
- **Example:** We write the magic value `0xcafe` here in every loop to prevent Jibo from panic-resetting during a dump.
    
- **C Code Implementation:**

```c
    while(1) {
        // (reset) 
        *(volatile uint32_t *)0x6000501c = 0xcafe; 
    
        // ... rest of the exploit ...
    }    
    
```

# Internal RAM (IRAM/SRAM)

This is the part where the BootROM is.

### `0x40000000` — IRAM Base (Start)

- **Use:** The beginning of the 256KB internal memory.
    
- **Significance:** This is where the payload is injected. If u dump from here, u get the "Leftovers" of the boot process (keys, signatures, and config data) from what i could tell.
    
### `0x4003FFFF` — IRAM End

- **Use:** The boundary of the volatile memory. (not 100% Sure but pretty certain)
    
- **Constraint:** If we try to read past this address, the system will hard-crash because we are hitting undefined memory space.

# BootROM Service Functions

The BootROM (at `0x00000000`) is read-only memory etched into the silicon. It contains pre-written functions for USB communication that we can "borrow" so we don't have to write a full USB driver from scratch.

### `0x00003551` — usb_send (Standard)

- **Use:** Sends a structured data packet over the RCM USB connection.
    
- **Significance:** This is the basically the proper way i could get it to talk. It handles some of the USB protocol handshaking automatically.
    
- **Example:**

```C
    // Define the function signature based on BootROM
    typedef int (*usb_send_ptr)(void *buffer, unsigned int length);
    usb_send_ptr usb_send = (usb_send_ptr)0x00003551;
    
    //push 64 bytes of IRAM to the PC
    usb_send((void *)0x40000000, 64);
```


### `0x000035e5` — usb_send_raw (Bulk)

- **Use:** A lower-level function that bypasses some of the standard RCM state checks i think.
    
- **Significance:** This is how i got those first **16 bytes** out. It is open data pushing.
    
- **Example:** I used this when the standard `usb_send` was stalling, as it forces the hardware to dump the buffer to the endpoint immediately.

	```c
// Define the raw function pointer
typedef void (*usb_send_raw_ptr)(void *buffer, unsigned int length);
usb_send_raw_ptr usb_send_raw = (usb_send_raw_ptr)0x000035e5;

//Force dump a 16-byte chunk of IRAM regardless of USB state
void dump_raw_chunk() {
    uint8_t buffer[16];
    // Copy the data from a target address (e.g.sstart of IRAM)
    memcpy(buffer, (void*)0x40000000, 16);
    
    // Push it out. On the PC side
    usb_send_raw(buffer, 16);
}
	```

# Hardware Fuse Registers 

The Fuses are permanent oly set at the factory so we cant change these

### `0x7000F800` — FUSE_BASE

- **Use:** The starting point for reading the hardware id & status.
    
- **Significance:** Reading from here tells us if Jibo is in "Production" mode (Locked) or "Development" mode (Unlocked).
    

### `0x7000F800 + 0x60` — Security Config Fuse

- **Use:** Determines if **Secure Boot** is active.
    
- **Discovery:** In my tests, reading this offset returned random data.
    
- **The Verdict:** This confirmed that Jibo is a **Secure Boot** device. He will only boot software that has been digitally signed by Jibo Inc.’s private keys.
    

### `0x7000F900` — FUSE_RESERVED_ODM (The SBK/SSK Area)

- **Use:** This is where the **Secure Boot Key (SBK)** usually is.
    
- **The Goal:** If we can successfully read this entire block, we might find the AES keys needed to decrypt the Jibo OS files on the EMMC.

# The Boot Configuration Table (BCT)

The BCT is a piece of data usually found at the very start of the boot process (and often cached in IRAM).

### `0x400000f0` — BCT Pointer / Entry Point

- **Use:** I identified this address in on of the `hexdumps` i sent on discord.
    
- **Significance:** This is where the BootROM hands over control to the next stage of the bootloader.
    
- **The Hack:** By targeting our leaker near this address, we can see how the Tegra sets up its memory controllers before it realizes we've hijacked the process.

```C
	void inspect_bct_handoff() {
    // Read the instruction at the BCT entry point
    uint32_t entry_instruction = *(volatile uint32_t *)0x400000f0;
    
    // Often, this is a 'Branch' (B) instruction in ARM hex: 0xEAXXXXXX
    // If 0xEA, we know Jibo is ready run the bootloader
    if ((entry_instruction >> 24) == 0xEA) {
        // Log it for the PC to see
        *(volatile uint32_t *)0x7000E450 = 0xBC7DE4D; // "BCT DEAD" hex flag
    }
}
```