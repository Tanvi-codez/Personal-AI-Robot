# Personal-AI-Robot
import RPi.GPIO as GPIO
from time import sleep
from PIL import Image, ImageDraw
from luma.core.interface.serial import i2c
from luma.oled.device import ssd1306
import speech_recognition as sr

# ===============================
# MOTOR SETUP
# ===============================
STBY = 17
AIN1, AIN2, PWMA = 27, 22, 18
BIN1, BIN2, PWMB = 23, 24, 13

GPIO.setmode(GPIO.BCM)
for pin in [STBY, AIN1, AIN2, PWMA, BIN1, BIN2, PWMB]:
    GPIO.setup(pin, GPIO.OUT)

left_pwm = GPIO.PWM(PWMA, 1000)
right_pwm = GPIO.PWM(PWMB, 1000)
left_pwm.start(0)
right_pwm.start(0)

def standby(on=True):
    GPIO.output(STBY, GPIO.HIGH if on else GPIO.LOW)

def set_motor(left_speed, right_speed):
    standby(True)
    GPIO.output(AIN1, left_speed > 0)
    GPIO.output(AIN2, left_speed < 0)
    GPIO.output(BIN1, right_speed > 0)
    GPIO.output(BIN2, right_speed < 0)
    left_pwm.ChangeDutyCycle(abs(left_speed))
    right_pwm.ChangeDutyCycle(abs(right_speed))

def stop():
    set_motor(0, 0)
    standby(False)

def forward(speed=70):
    set_motor(speed, speed)

def backward(speed=70):
    set_motor(-speed, -speed)

def left_turn(speed=70):
    set_motor(-speed, speed)

def right_turn(speed=70):
    set_motor(speed, -speed)

# ===============================
# OLED SETUP (Eyes)
# ===============================
serial = i2c(port=1, address=0x3C)
device = ssd1306(serial)
width, height = device.width, device.height

def draw_eyes(mode="neutral"):
    image = Image.new("1", (width, height))
    draw = ImageDraw.Draw(image)
    eye_radius = 12
    x_left, x_right = 25, 90
    y_center = 32

    if mode == "neutral":
        draw.ellipse((x_left-eye_radius, y_center-eye_radius, x_left+eye_radius, y_center+eye_radius), fill=255)
        draw.ellipse((x_right-eye_radius, y_center-eye_radius, x_right+eye_radius, y_center+eye_radius), fill=255)
    elif mode == "left":
        draw.ellipse((x_left-eye_radius-5, y_center-eye_radius, x_left+eye_radius-5, y_center+eye_radius), fill=255)
        draw.ellipse((x_right-eye_radius-5, y_center-eye_radius, x_right+eye_radius-5, y_center+eye_radius), fill=255)
    elif mode == "right":
        draw.ellipse((x_left-eye_radius+5, y_center-eye_radius, x_left+eye_radius+5, y_center+eye_radius), fill=255)
        draw.ellipse((x_right-eye_radius+5, y_center-eye_radius, x_right+eye_radius+5, y_center+eye_radius), fill=255)
    elif mode == "surprised":
        draw.rectangle((x_left-eye_radius, y_center-eye_radius, x_left+eye_radius, y_center+eye_radius), outline=255)
        draw.rectangle((x_right-eye_radius, y_center-eye_radius, x_right+eye_radius, y_center+eye_radius), outline=255)
    elif mode == "sleep":
        draw.line((x_left-eye_radius, y_center, x_left+eye_radius, y_center), fill=255, width=3)
        draw.line((x_right-eye_radius, y_center, x_right+eye_radius, y_center), fill=255, width=3)
    device.display(image)

draw_eyes("neutral")

# ===============================
# VOICE SETUP
# ===============================
recognizer = sr.Recognizer()
mic = sr.Microphone()

def listen_command():
    with mic as source:
        print("🎤 Listening...")
        recognizer.adjust_for_ambient_noise(source)
        audio = recognizer.listen(source)
    try:
        command = recognizer.recognize_google(audio).lower()
        print(f"You said: {command}")
        return command
    except sr.UnknownValueError:
        print("❓ Couldn't understand.")
        return ""
    except sr.RequestError:
        print("⚠️ Speech service issue.")
        return ""

# ===============================
# MAIN LOOP
# ===============================
try:
    print("Voice + eyes + motors active! 🎧🤖")
    draw_eyes("neutral")
    while True:
        cmd = listen_command()

        if "forward" in cmd:
            draw_eyes("neutral")
            forward()
            print("➡️ Moving forward")
        elif "back" in cmd or "reverse" in cmd:
            draw_eyes("surprised")
            backward()
            print("⬅️ Moving backward")
        elif "left" in cmd:
            draw_eyes("left")
            left_turn()
            print("↩️ Turning left")
        elif "right" in cmd:
            draw_eyes("right")
            right_turn()
            print("↪️ Turning right")
        elif "stop" in cmd or "sleep" in cmd:
            draw_eyes("sleep")
            stop()
            print("😴 Stopped")
        sleep(0.5)

except KeyboardInterrupt:
    print("\nExiting...")
finally:
    stop()
    left_pwm.stop()
    right_pwm.stop()
    GPIO.cleanup()
    draw_eyes("sleep")

