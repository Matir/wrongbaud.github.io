# Notes  for cabinet research -- disclaimer these will be _AWFUL_

Nametable for boot screen starts at 0x7D149

Bootloader is at 0x7C000 (?)

test menu at 0x74000 (?)

CHR table at 0x70000 (?)

0x40000 seems to be image size?

0x70000 --> This  is the CHR table for the main bootloader! Modifying this changes the loading screen


| Start Addr | End Addr | Content | 
| ---------- | -------- | ------- | 
| 0 | 0x30000 | Rampage Rom | 
| 0x30000 | 0x40000 | Mirror of 0x20000:0x30000 |
| 0x40000 | 0x60000 | (???) Removing this didn't seem to change anything |
| 0x60000 | 0x70000 | CHR Table for first stage loader / test menu (?) | 
| 0x70000 | 0x74000 | Also CHR table! | 
| 0x74000 | 0x78000 | Unknown (possibly dev menu) |
| 0x7C000 | 0x7FFFF | Initial loader? | 

So it looks like we have two 16k program areas ...  

