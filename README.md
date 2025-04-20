from machine import Pin, I2C
from time import sleep

# === LCD Setup ===
i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=400000)
LCD_ADDR = 0x27
LCD_BACKLIGHT = 0x08
ENABLE = 0b00000100
CMD_MODE = 0
CHAR_MODE = 1

def lcd_strobe(data):
    i2c.writeto(LCD_ADDR, bytearray([data | ENABLE | LCD_BACKLIGHT]))
    sleep(0.001)
    i2c.writeto(LCD_ADDR, bytearray([(data & ~ENABLE) | LCD_BACKLIGHT]))
    sleep(0.001)

def lcd_write_four_bits(data):
    i2c.writeto(LCD_ADDR, bytearray([data | LCD_BACKLIGHT]))
    lcd_strobe(data)

def lcd_write(cmd, mode=CMD_MODE):
    high = mode | (cmd & 0xF0)
    low = mode | ((cmd << 4) & 0xF0)
    lcd_write_four_bits(high)
    lcd_write_four_bits(low)

def lcd_clear():
    lcd_write(0x01)
    sleep(0.002)

def lcd_init():
    sleep(0.05)
    lcd_write_four_bits(0x30)
    sleep(0.005)
    lcd_write_four_bits(0x30)
    sleep(0.001)
    lcd_write_four_bits(0x30)
    lcd_write_four_bits(0x20)
    lcd_write(0x28)  # Function set: 4-bit, 2-line
    lcd_write(0x0C)  # Display ON, cursor OFF
    lcd_write(0x06)  # Entry mode set
    lcd_clear()

def lcd_message(message, line=0):
    line_addr = [0x80, 0xC0]
    lcd_write(line_addr[line])
    for char in message:
        lcd_write(ord(char), CHAR_MODE)

# === Keypad Setup ===
keypad_rows = [Pin(x, Pin.OUT) for x in [32, 33, 25, 26]]
keypad_cols = [Pin(x, Pin.IN, Pin.PULL_DOWN) for x in [27, 14, 12, 13]]

keypad_map = [
    ["1", "2", "3", "A"],
    ["4", "5", "6", "B"],
    ["7", "8", "9", "C"],
    ["*", "0", "#", "D"]
]

def scan_keypad():
    for row_num, row_pin in enumerate(keypad_rows):
        row_pin.on()
        for col_num, col_pin in enumerate(keypad_cols):
            if col_pin.value():
                row_pin.off()
                return keypad_map[row_num][col_num]
        row_pin.off()
    return None

# === Palindrome Checker ===
def is_palindrome(s):
    reversed_s = ''.join(reversed(s))  # Manual reverse
    return s == reversed_s

# === Main Program ===
lcd_init()
lcd_message("Enter number:")
input_str = ""

while True:
    key = scan_keypad()
    if key:
        if key == "#":
            lcd_clear()
            lcd_message(input_str[:16], line=0)
            if is_palindrome(input_str):
                lcd_message("Palindrome :)", line=1)
            else:
                lcd_message("Not Palindrome", line=1)
            sleep(3)
            lcd_clear()
            lcd_message("Enter number:")
            input_str = ""
        elif key == "*":
            input_str = ""
            lcd_clear()
            lcd_message("Enter number:")
        elif key in "0123456789":
            if len(input_str) < 16:
                input_str += key
                lcd_message(input_str, line=1)
        sleep(0.3)  # Debounce delay# palindrome-checker-number
