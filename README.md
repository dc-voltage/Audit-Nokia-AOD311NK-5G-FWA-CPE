<img width="870" height="334" alt="WhatsApp Image 2026-10-31 at 7 47 20 PM" src="https://github.com/user-attachments/assets/0ae87056-09fa-472d-846f-5583c28ff0ca" />
Security & Architecture Audit: Nokia AOD311NK 5G FWA CPE
                                                                                                                  
  Chip-inte top-il oru dot (Pin 1 marker) kaanam. Pin numbering thazhe parayum poleyanu:                            
                                                                                                                    
                   Top View (WSON8)                                                                                 
                  ┌───┬───────┬───┐                                                                                 
         /CS (1) ─┤ o │       │   ├─ (8) VCC (1.8V)                                                                 
      DO/IO1 (2) ─┤   │       │   ├─ (7) /HOLD (IO3)                                                                
     /WP/IO2 (3) ─┤   │       │   ├─ (6) CLK                                                                        
         GND (4) ─┤   │       │   ├─ (5) DI/IO0                                                                     
                  └───┴───────┴───┘                                                                                 
                                                                                                                    
             Pin           │ Name                   │ Programmer Connection
  ─────────────────────────┼────────────────────────┼───────────────────────────────────────────────────────────────
              1            │ /CS                    │ Programmer Chip Select (CS / SS)
              2            │ DO (IO1)               │ Programmer MISO (Data Output)
              3            │ /WP (IO2)              │ 1.8V VCC-ilekku pull cheyyuka (Write Protect disable cheyyan)
              4            │ GND                    │ Programmer Ground (GND)
              5            │ DI (IO0)               │ Programmer MOSI (Data Input)
              6            │ CLK                    │ Programmer Serial Clock (SCK / CLK)
              7            │ /HOLD (IO3)            │ 1.8V VCC-ilekku pull cheyyuka
              8            │ VCC                    │ 1.8V Power Rail ONLY
  ──────                                                                                         
