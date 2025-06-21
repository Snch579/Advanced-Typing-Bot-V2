# Advanced-Typing-Bot-V2
Python GUI tool using Tkinter for automated typing with configurable speed, delay, randomness, and simulated mistakes. Supports text file operations, text cleaning, themes, settings persistence, looping, and time estimates. Uses pyautogui for typing and pynput for optional emergency stop.

Features:
  Tkinter-based graphical user interface
  Adjustable typing speed, delays, and randomization
  Simulated typing mistakes with automatic correction
  Text file import/export, clipboard operations, and text cleaning
  Light and dark mode themes
  Persistent user settings (save/load)
  Looping support for repeated typing sessions
  Real-time estimated typing duration display
  Emergency stop functionality via Escape key (requires pynput)
  Direct typing into any focused application window using pyautogui

Use Cases:
  Automated form filling (non-critical systems)
  UI automation demonstrations and testing
  Simulated typing for presentations or recordings
  Stress testing text input interfaces
  UX studies involving typing behavior

Installation:
  Requirements:
    Python 3.7 or higher
    pyautogui
    pynput (optional, for emergency stop feature)
    tkinter (typically included with standard Python)

  Setup:
    Clone the repository:
    git clone https://github.com/yourusername/advanced-typing-bot.git
    cd advanced-typing-bot
    
  Install dependencies:
    pip install pyautogui pynput
    The pynput package is optional but recommended for full functionality.

Usage:
  Run the application:
    python main.py
    
  Load or paste the text to be typed.
  Configure typing parameters including speed, delay, randomness, and error simulation.
  Start the typing process and switch to the target window.
  Press the Escape key to interrupt typing if necessary (requires pynput).
  Note: This tool simulates real keyboard input. Ensure the correct window is focused before starting.

License:
This project is licensed under the MIT License. See the LICENSE file for details.

Disclaimer:
This tool is intended for personal, educational, and non-critical use cases. The author assumes no responsibility for unintended consequences, misuse, or data loss resulting from its operation.

