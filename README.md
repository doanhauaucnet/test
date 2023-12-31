import configparser
import psutil
import os
import subprocess
import pyautogui
import time
import sys
import platform
import socket
import requests

# Config wait time
loop_time_for_response = 300  # How many loop will run to try detect text before stop
wait_time_for_each_loop = 0.5  # How many second each loop will wait for the screen to update
wait_time_for_keyboard_enter = 0.2 # How many second each keyboard operation will wait befor continue
wait_time_for_mouse_click = 0.2 # How many second each keyboard operation will wait befor continue
basic_confidence = 0.9 # How many second each keyboard operation will wait befor continue
basic_scroll_amout = -10 # How many second each keyboard operation will wait befor continue


# Set of color for print color function
class ConsoleColors:
    RESET = "\033[0m"
    RED = "\033[91m"
    GREEN = "\033[92m"
    LIGHT_GRAY = "\033[90m"
    BLUE = "\033[94m"

# Print to console with colors
def print_colored(text, color):
    print(f"{color}{text}{ConsoleColors.RESET}")

# Read authentication from config file
def read_authentication_config(file_path, type):
    config = configparser.ConfigParser()
    config.read(file_path)
    account = config.get(type, 'account')
    password = config.get(type, 'password')
    return account, password

# Get the name of proccess from file path
def extract_filename_from_path(file_path):
    return os.path.basename(file_path)

# Check if proccess with given path is running or not
def is_process_running_by_path(executable_path):
    for process in psutil.process_iter(attrs=["name", "exe"]):
        if process.info["exe"] and process.info["exe"].lower() == executable_path.lower():
            return True
    return False

# Kill the proccess with given path
def kill_process_by_path(executable_path):
    for process in psutil.process_iter(attrs=["name", "exe"]):
        if process.info["exe"] and process.info["exe"].lower() == executable_path.lower():
            process.terminate()
            return True
    return False

# Detect proccess and run proccess
def active_proccess(executable_path):
    proccess_name = extract_filename_from_path(executable_path)
    if is_process_running_by_path(executable_path):
        print_colored(f"The process '{proccess_name}' is running, try to reopen it.", ConsoleColors.BLUE)
        kill_process_by_path(executable_path)
        subprocess.Popen(executable_path)
    else:
        print_colored(f"The process '{proccess_name}' is not running, try to open it.", ConsoleColors.BLUE)
        subprocess.Popen(executable_path)

# Detect element
def find_element_based_on_reference(reference_locations, 
    delay = wait_time_for_each_loop, max_loop= loop_time_for_response, confidence = basic_confidence):
    for reference_num, reference_location in enumerate(reference_locations, start=1):
        for i in range(max_loop):
            text_position = pyautogui.locateOnScreen(reference_location, confidence=confidence)
            if text_position is not None:
                print(f"--Target {reference_num} found at {text_position}")
                return text_position, reference_num
            else:
                print(f"--Not found, try again {i + 1}/{max_loop}")
                time.sleep(delay)
    return 0, 0

def find_reference_or_scroll(reference_locations, 
    delay = wait_time_for_each_loop, max_loop= loop_time_for_response, confidence = basic_confidence, scroll_amount = basic_scroll_amout):
    for reference_num, reference_location in enumerate(reference_locations, start=1):
        for i in range(max_loop):
            text_position = pyautogui.locateOnScreen(reference_location, confidence=confidence)
            if text_position is not None:
                print(f"--Target {reference_num} found at {text_position}")
                return text_position, reference_num
            else:
                print(f"--Not found, try again {i + 1}/{max_loop}")
                pyautogui.scroll(scroll_amount)
                time.sleep(delay)
    return 0, 0

# click at x, y position of a box object
def click_at_location_of_box(box_object):
    time.sleep(wait_time_for_mouse_click)
    pyautogui.click(box_object)

def is_reference_found_and_click(reference_location, is_click, 
    delay = wait_time_for_each_loop, max_loop = loop_time_for_response, confidence = basic_confidence):
    reference_name = format_file_name(reference_location)
    reference_position, reference_num = find_element_based_on_reference(reference_location, delay, max_loop, confidence)
    if not reference_position:
        print_colored(f"{reference_name} not found, end", ConsoleColors.RED)
        sys.exit(0)
    print_colored(f"{reference_name} found, continue", ConsoleColors.BLUE)
    if(is_click):
        click_at_location_of_box(reference_position)
        print_colored(f"Click at {reference_name} location", ConsoleColors.BLUE)

def is_reference_found_and_move_to(reference_location, is_move_to, 
    delay = wait_time_for_each_loop, max_loop = loop_time_for_response, confidence = basic_confidence):
    reference_name = format_file_name(reference_location)
    reference_position, reference_num = find_element_based_on_reference(reference_location, delay, max_loop, confidence)
    if not reference_position:
        print_colored(f"{reference_name} not found, end", ConsoleColors.RED)
        sys.exit(0)
    print_colored(f"{reference_name} found, continue", ConsoleColors.BLUE)
    if(is_move_to):
        pyautogui.moveTo(reference_position)
        print_colored(f"Moved to {reference_name} location", ConsoleColors.BLUE)

def press_key(key_name):
    pyautogui.press(key_name)
    print_colored(f"Pressed {key_name} key", ConsoleColors.BLUE)
    time.sleep(wait_time_for_keyboard_enter)

def press_key_with_delay(key_string, delay=0.2):
    for key in key_string:
        pyautogui.press(key)
        print(key, end='', flush=True)
        time.sleep(delay)
    print()
    print_colored(f"Pressed {key_string} key", ConsoleColors.BLUE)

def format_file_name(reference_location):
    # Extract the file name without the extension
    file_name = reference_location[0].split('/')[-1].split('.')[0]
    # Split the name into words based on underscores
    words = file_name.split('_')
    # Capitalize the first letter of each word and join them
    formatted_name = ' '.join(word.capitalize() for word in words)
    return formatted_name

def find_elements_at_the_same_time(reference_locations, 
    delay = wait_time_for_each_loop, max_loop= loop_time_for_response, confidence = basic_confidence):
    for i in range(max_loop):
        for index, reference_location in enumerate(reference_locations, start=1):
            text_position = pyautogui.locateOnScreen(reference_location, confidence=confidence)
            if text_position is not None:
                print(f"--Target found at {text_position}")
                print_colored(f"Found {(reference_location)} (Index: {index})", ConsoleColors.BLUE)
                return index

        print(f"--Not Found, try again {i + 1}/{max_loop}")
        time.sleep(delay)

    return None

def get_connected_wifi():
    system_platform = platform.system()

    if system_platform == 'Windows':
        try:
            result = os.popen('netsh wlan show interfaces').read()
            lines = result.split('\n')
            for line in lines:
                if "SSID" in line:
                    ssid = line.split(":")[1].strip()
                    return ssid
        except Exception as e:
            print(f"Error: {e}")
            return None

    elif system_platform == 'Linux':
        try:
            result = os.popen('iwgetid -r').read().strip()
            return result
        except Exception as e:
            print(f"Error: {e}")
            return None

    else:
        print(f"Unsupported platform: {system_platform}")
        return None

def get_connected_server_info():
    try:
        # Get the hostname and IP address of the connected server
        hostname = socket.gethostname()
        ip_address = socket.gethostbyname(hostname)
        return hostname, ip_address
    except Exception as e:
        print(f"Error: {e}")
        return None, None

def get_public_ip():
    try:
        # For IPv4
        response = requests.get('https://api64.ipify.org?format=json')
        # For IPv6
        # response = requests.get('https://api64.ipify.org?format=json')

        if response.status_code == 200:
            data = response.json()
            return data['ip']
        else:
            print(f"Failed to retrieve public IP. Status code: {response.status_code}")
            return None
    except Exception as e:
        print(f"Error: {e}")
        return None
    
def calculate_intersection_center(box1, box2):
    center_y = (box1.top + box1.height / 2) 
    center_x = (box2.left + box2.width / 2) 
    return center_x, center_y
