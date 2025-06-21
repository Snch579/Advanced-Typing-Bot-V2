Advanced-Typing-Bot-V2
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

Code:
import tkinter as tk
from tkinter import ttk # For Notebook in dialog
from tkinter import scrolledtext, messagebox, filedialog
import pyautogui
import time
import threading
import random
import string
import json
import re
import os

try:
    from pynput import keyboard
    PYNPUT_AVAILABLE = True
except ImportError:
    PYNPUT_AVAILABLE = False
    print("Warning: pynput library not found. Emergency stop keybind (Esc) will not be available.")
    print("To enable it, install pynput: pip install pynput")

# --- Global Variables & Constants ---
stop_event = threading.Event()
keyboard_listener = None
utility_dialog_open = False

DEFAULT_DELAY_SEC = 3
DEFAULT_SPEED_S_CHAR = 0.02
DEFAULT_ERRATIC_MIN_S = 0.001
DEFAULT_ERRATIC_MAX_S = 0.1
DEFAULT_MISTAKE_PROBABILITY_PERCENT = 5.0
DEFAULT_LOOP_COUNT = 1

MISTAKE_VISIBILITY_PAUSE_MIN = 0.1
MISTAKE_VISIBILITY_PAUSE_MAX = 0.3
CORRECTION_ACTION_PAUSE_MIN = 0.05
CORRECTION_ACTION_PAUSE_MAX = 0.15
ERRATIC_WORD_GROUP_PAUSE_MIN = 0.8
ERRATIC_WORD_GROUP_PAUSE_MAX = 1.2
PUNCTUATION_CHARS = set(['.', ',', '!', '?', ';', ':'])
PUNCTUATION_EXTRA_PAUSE_MIN = 0.05
PUNCTUATION_EXTRA_PAUSE_MAX = 0.2

SETTINGS_FILENAME = "typer_settings_v5.json" # V5 for this layout

class TypingStoppedException(Exception):
    pass

# --- Tkinter Variables (Global for settings access) ---
delay_var, speed_var, erratic_var, min_erratic_speed_var, max_erratic_speed_var, \
mistake_prob_var, dark_mode_var, loop_typing_var, loop_count_var, \
trim_whitespace_var, normalize_line_endings_var, remove_extra_blank_lines_var, \
apply_cleaning_on_start_var, always_on_top_var = (None,) * 14 # Initialize all to None

# --- Theme Definitions ---
light_theme = {
    "root_bg": "#F0F0F0", "frame_bg": "#F0F0F0", "label_fg": "black",
    "button_bg": "#E1E1E1", "button_fg": "black", "button_active_bg": "#CFCFCF",
    "entry_bg": "white", "entry_fg": "black", "entry_insert_bg": "black",
    "entry_disabled_bg": "#F0F0F0", "entry_disabled_fg": "#A3A3A3",
    "text_area_bg": "white", "text_area_fg": "black", "text_area_insert_bg": "black",
    "scrolledtext_border": "#F0F0F0",
    "checkbutton_bg": "#F0F0F0", "checkbutton_fg": "black", "checkbutton_select_color": "#F0F0F0",
    "status_bar_bg": "#F0F0F0", "status_bar_fg": "black",
    "notebook_bg": "#F0F0F0", "tab_bg": "#E1E1E1", "tab_selected_bg": "#F0F0F0", "tab_fg": "black"
}
dark_theme = {
    "root_bg": "#2E2E2E", "frame_bg": "#2E2E2E", "label_fg": "#E0E0E0",
    "button_bg": "#4A4A4A", "button_fg": "#E0E0E0", "button_active_bg": "#5A5A5A",
    "entry_bg": "#3C3C3C", "entry_fg": "#E0E0E0", "entry_insert_bg": "#E0E0E0",
    "entry_disabled_bg": "#333333", "entry_disabled_fg": "#888888",
    "text_area_bg": "#1E1E1E", "text_area_fg": "#E0E0E0", "text_area_insert_bg": "#E0E0E0",
    "scrolledtext_border": "#2E2E2E",
    "checkbutton_bg": "#2E2E2E", "checkbutton_fg": "#E0E0E0", "checkbutton_select_color": "#4A4A4A",
    "status_bar_bg": "#1E1E1E", "status_bar_fg": "#E0E0E0",
    "notebook_bg": "#2E2E2E", "tab_bg": "#4A4A4A", "tab_selected_bg": "#2E2E2E", "tab_fg": "#E0E0E0"
}
current_theme = light_theme

# --- Theming Function ---
def apply_theme_to_widget(widget, theme_colors):
    if not widget.winfo_exists(): return
    widget_class = widget.winfo_class()
    try: widget.config(fg=theme_colors["label_fg"])
    except tk.TclError: pass # Some widgets might not have 'fg' or it's complex
    try: widget.config(bg=theme_colors.get(f"{widget.winfo_name()}_bg", theme_colors["frame_bg"]))
    except tk.TclError: pass

    if widget_class == "Tk" or widget_class == "Toplevel": widget.config(bg=theme_colors["root_bg"])
    elif widget_class == "Frame" or widget_class == "Labelframe": widget.config(bg=theme_colors["frame_bg"])
    elif widget_class == "Label":
        widget.config(fg=theme_colors["label_fg"], bg=theme_colors.get(f"{widget.winfo_name()}_bg", theme_colors["frame_bg"]))
        if 'status_label' in globals() and widget == status_label: # Specific for status bar
             widget.config(bg=theme_colors["status_bar_bg"], fg=theme_colors["status_bar_fg"])
    elif widget_class == "Button":
        widget.config(fg=theme_colors["button_fg"], bg=theme_colors["button_bg"],
                      activebackground=theme_colors["button_active_bg"],
                      activeforeground=theme_colors["button_fg"])
        if widget['state'] == tk.DISABLED:
             widget.config(bg=theme_colors.get("button_disabled_bg", "#777777" if current_theme == dark_theme else "#D3D3D3"))
    elif widget_class == "Entry":
        widget.config(fg=theme_colors["entry_fg"], bg=theme_colors["entry_bg"],
                      insertbackground=theme_colors["entry_insert_bg"],
                      disabledbackground=theme_colors["entry_disabled_bg"],
                      disabledforeground=theme_colors["entry_disabled_fg"],
                      readonlybackground=theme_colors["entry_disabled_bg"])
    elif widget_class == "ScrolledText":
        widget.config(background=theme_colors["scrolledtext_border"]) # Border color
        try: # Access the actual Text widget inside ScrolledText
            text_widget = widget.children['!text']
            text_widget.configure(bg=theme_colors["text_area_bg"], fg=theme_colors["text_area_fg"],
                            insertbackground=theme_colors["text_area_insert_bg"])
        except (KeyError, tk.TclError): # Fallback if internal structure changes or error
            widget.configure(bg=theme_colors["text_area_bg"], fg=theme_colors["text_area_fg"],
                            insertbackground=theme_colors["text_area_insert_bg"])
    elif widget_class == "Text": # For Text widgets not part of ScrolledText (if any)
         widget.config(bg=theme_colors["text_area_bg"], fg=theme_colors["text_area_fg"],
                       insertbackground=theme_colors["text_area_insert_bg"])
    elif widget_class == "Checkbutton":
        widget.config(fg=theme_colors["checkbutton_fg"], bg=theme_colors["checkbutton_bg"],
                      activebackground=theme_colors["checkbutton_bg"],
                      activeforeground=theme_colors["checkbutton_fg"],
                      selectcolor=theme_colors["checkbutton_select_color"])
    elif widget_class == "TNotebook": # For ttk.Notebook
        style = ttk.Style(widget) # Use widget itself if available
        style.configure("TNotebook", background=theme_colors["notebook_bg"])
        style.map("TNotebook.Tab",
                  background=[("selected", theme_colors["tab_selected_bg"]), ("!selected", theme_colors["tab_bg"])],
                  foreground=[("selected", theme_colors["tab_fg"]), ("!selected", theme_colors["tab_fg"])])
        # Theme the actual tab frames (which are tk.Frame)
        for tab_id in widget.tabs():
            try:
                tab_frame = widget.nametowidget(tab_id)
                if tab_frame.winfo_exists(): # Make sure it's a widget we can configure
                     tab_frame.config(bg=theme_colors["frame_bg"]) # Theme the frame holding tab content
                     for child in tab_frame.winfo_children(): # Theme children within the tab frame
                         apply_theme_to_widget(child, theme_colors)
            except tk.TclError:
                pass # nametowidget can fail if tab_id isn't a direct child name
    
    # Recursively apply to children, except for ScrolledText and Notebook internal structure
    if widget_class not in ["ScrolledText", "TNotebook"]:
        for child in widget.winfo_children():
            apply_theme_to_widget(child, theme_colors)

def toggle_main_theme():
    global current_theme
    current_theme = dark_theme if dark_mode_var.get() else light_theme
    apply_theme_to_widget(root, current_theme)
    if utility_dialog_open and 'utility_dialog' in globals() and utility_dialog.winfo_exists():
        apply_theme_to_widget(utility_dialog, current_theme)
    toggle_erratic_dependent_options()
    toggle_loop_count_entry_state()

# --- Utility Functions ---
def check_and_raise_stop():
    if stop_event.is_set(): raise TypingStoppedException()

def update_counts_and_time(event=None):
    update_counts()
    update_estimated_time()

def update_counts():
    if 'text_area' in globals() and text_area.winfo_exists():
        content = text_area.get("1.0", tk.END).strip()
        char_count_var.set(f"Chars: {len(content)}")
        word_count_var.set(f"Words: {len(content.split()) if content else 0}")

def update_estimated_time():
    if 'text_area' not in globals() or not text_area.winfo_exists(): return
    num_chars = len(text_area.get("1.0", tk.END).strip())
    if num_chars == 0:
        estimated_time_var.set("Est. Time: 0s")
        return

    avg_interval = 0.0
    if erratic_var.get():
        min_s = min_erratic_speed_var.get()
        max_s = max_erratic_speed_var.get()
        avg_interval = (min_s + max_s) / 2.0
        # Add a bit for potential mistakes/pauses in erratic mode
        avg_interval += (mistake_prob_var.get() / 100.0) * \
                        ((MISTAKE_VISIBILITY_PAUSE_MIN + MISTAKE_VISIBILITY_PAUSE_MAX)/2 + \
                         (CORRECTION_ACTION_PAUSE_MIN + CORRECTION_ACTION_PAUSE_MAX)/2)
        # Add a bit for punctuation pauses
        avg_interval += 0.01 # Rough estimate for punctuation effect
    else:
        avg_interval = speed_var.get()

    if avg_interval <= 0: avg_interval = 0.01 # Avoid division by zero if speed is 0
    
    total_seconds = num_chars * avg_interval
    
    # Add delay time
    try: total_seconds += delay_var.get()
    except tk.TclError: pass # If delay_var is not yet valid

    if total_seconds < 60:
        estimated_time_var.set(f"Est. Time: {total_seconds:.1f}s")
    else:
        minutes = int(total_seconds / 60)
        seconds = total_seconds % 60
        estimated_time_var.set(f"Est. Time: {minutes}m {seconds:.0f}s")


# --- Text Operations (called from Utility Dialog) ---
def load_text_from_file_action():
    filepath = filedialog.askopenfilename(defaultextension=".txt", filetypes=[("Text Files", "*.txt"), ("All Files", "*.*")])
    if not filepath: return
    try:
        with open(filepath, "r", encoding="utf-8") as f:
            text_area.delete("1.0", tk.END); text_area.insert("1.0", f.read())
        status_var.set(f"Loaded: {os.path.basename(filepath)}"); update_counts_and_time()
    except Exception as e: messagebox.showerror("Error Loading", f"{e}"); status_var.set(f"Error loading: {e}")

def save_text_to_file_action():
    content = text_area.get("1.0", tk.END).strip()
    if not content: messagebox.showwarning("Empty Text", "Nothing to save."); return
    filepath = filedialog.asksaveasfilename(defaultextension=".txt", filetypes=[("Text Files", "*.txt"), ("All Files", "*.*")])
    if not filepath: return
    try:
        with open(filepath, "w", encoding="utf-8") as f: f.write(content)
        status_var.set(f"Saved: {os.path.basename(filepath)}")
    except Exception as e: messagebox.showerror("Error Saving", f"{e}"); status_var.set(f"Error saving: {e}")

def paste_from_clipboard_action():
    try:
        text_area.insert(tk.INSERT, root.clipboard_get()); status_var.set("Pasted from clipboard."); update_counts_and_time()
    except tk.TclError: messagebox.showwarning("Clipboard Error", "Clipboard empty/not text."); status_var.set("Clipboard error.")
    except Exception as e: messagebox.showerror("Paste Error", f"{e}"); status_var.set(f"Error pasting: {e}")

def clear_text_area_action(): text_area.delete("1.0", tk.END); status_var.set("Text area cleared."); update_counts_and_time()

def copy_to_clipboard_action():
    content = text_area.get("1.0", tk.END).strip()
    if not content: messagebox.showwarning("Empty Text", "Nothing to copy."); return
    try:
        root.clipboard_clear()
        root.clipboard_append(content)
        status_var.set("Text copied to clipboard.")
    except Exception as e: messagebox.showerror("Copy Error", f"{e}"); status_var.set(f"Error copying: {e}")


def apply_text_cleaning_action():
    current_text = text_area.get("1.0", tk.END); original_text_for_comparison = current_text
    modified_text = current_text
    if trim_whitespace_var.get(): modified_text = modified_text.strip() + ("\n" if modified_text.strip() and original_text_for_comparison.endswith('\n') else "")
    if normalize_line_endings_var.get(): modified_text = modified_text.replace("\r\n", "\n").replace("\r", "\n")
    if remove_extra_blank_lines_var.get(): modified_text = re.sub(r'\n\s*\n', '\n\n', modified_text); modified_text = re.sub(r'\n{3,}', '\n\n', modified_text)
    if modified_text != original_text_for_comparison:
        text_area.delete("1.0", tk.END); text_area.insert("1.0", modified_text)
        status_var.set("Text cleaning applied."); update_counts_and_time()
    else: status_var.set("Text cleaning (no changes).")

# --- Settings Management ---
def get_all_settings():
    return {
        "delay_sec": delay_var.get(), "speed_s_char": speed_var.get(),
        "erratic_mode": erratic_var.get(), "erratic_min_s": min_erratic_speed_var.get(),
        "erratic_max_s": max_erratic_speed_var.get(), "mistake_prob_percent": mistake_prob_var.get(),
        "dark_mode": dark_mode_var.get(), "loop_typing": loop_typing_var.get(),
        "loop_count": loop_count_var.get(),
        "trim_whitespace": trim_whitespace_var.get(),
        "normalize_line_endings": normalize_line_endings_var.get(),
        "remove_extra_blank_lines": remove_extra_blank_lines_var.get(),
        "apply_cleaning_on_start": apply_cleaning_on_start_var.get(),
        "always_on_top": always_on_top_var.get()
    }

def apply_all_settings(settings_dict):
    delay_var.set(settings_dict.get("delay_sec", DEFAULT_DELAY_SEC))
    speed_var.set(settings_dict.get("speed_s_char", DEFAULT_SPEED_S_CHAR))
    erratic_var.set(settings_dict.get("erratic_mode", False))
    min_erratic_speed_var.set(settings_dict.get("erratic_min_s", DEFAULT_ERRATIC_MIN_S))
    max_erratic_speed_var.set(settings_dict.get("erratic_max_s", DEFAULT_ERRATIC_MAX_S))
    mistake_prob_var.set(settings_dict.get("mistake_prob_percent", DEFAULT_MISTAKE_PROBABILITY_PERCENT))
    dark_mode_var.set(settings_dict.get("dark_mode", False))
    loop_typing_var.set(settings_dict.get("loop_typing", False))
    loop_count_var.set(settings_dict.get("loop_count", DEFAULT_LOOP_COUNT))
    trim_whitespace_var.set(settings_dict.get("trim_whitespace", False))
    normalize_line_endings_var.set(settings_dict.get("normalize_line_endings", False))
    remove_extra_blank_lines_var.set(settings_dict.get("remove_extra_blank_lines", False))
    apply_cleaning_on_start_var.set(settings_dict.get("apply_cleaning_on_start", False))
    always_on_top_var.set(settings_dict.get("always_on_top", False))

    toggle_main_theme() # Updates main theme and dependent options
    toggle_always_on_top() # Apply always on top state
    update_estimated_time() # Update time estimate based on new settings

def save_settings_to_file_action():
    settings_to_save = get_all_settings()
    filepath = filedialog.asksaveasfilename(initialfile=SETTINGS_FILENAME, defaultextension=".json", filetypes=[("JSON Files", "*.json")])
    if not filepath: return False
    try:
        with open(filepath, "w") as f: json.dump(settings_to_save, f, indent=4)
        status_var.set(f"Settings saved: {os.path.basename(filepath)}"); return True
    except Exception as e: messagebox.showerror("Error Saving", f"{e}"); status_var.set(f"Error saving settings: {e}"); return False

def load_settings_from_file_action():
    filepath = filedialog.askopenfilename(initialfile=SETTINGS_FILENAME, defaultextension=".json", filetypes=[("JSON Files", "*.json")])
    if not filepath: return
    try:
        with open(filepath, "r") as f: loaded_settings = json.load(f)
        apply_all_settings(loaded_settings)
        status_var.set(f"Settings loaded: {os.path.basename(filepath)}")
    except FileNotFoundError: messagebox.showwarning("Load Error", "File not found."); reset_all_settings_action(silent=True); status_var.set("Settings file not found.")
    except Exception as e: messagebox.showerror("Load Error", f"{e}"); reset_all_settings_action(silent=True); status_var.set(f"Error loading settings: {e}")

def reset_all_settings_action(silent=False):
    default_settings = {
        "delay_sec": DEFAULT_DELAY_SEC, "speed_s_char": DEFAULT_SPEED_S_CHAR, "erratic_mode": False,
        "erratic_min_s": DEFAULT_ERRATIC_MIN_S, "erratic_max_s": DEFAULT_ERRATIC_MAX_S,
        "mistake_prob_percent": DEFAULT_MISTAKE_PROBABILITY_PERCENT, "dark_mode": False,
        "loop_typing": False, "loop_count": DEFAULT_LOOP_COUNT, "trim_whitespace": False,
        "normalize_line_endings": False, "remove_extra_blank_lines": False, "apply_cleaning_on_start": False,
        "always_on_top": False
    }
    apply_all_settings(default_settings)
    if not silent: status_var.set("All settings reset to defaults.")

# --- UI State Toggles (Main Window) ---
def toggle_input_states(typing_active):
    global current_theme
    button_disabled_bg_color = current_theme.get("button_disabled_bg", "#777777" if current_theme == dark_theme else "#D3D3D3")
    entry_state = tk.DISABLED if typing_active else tk.NORMAL
    button_state = tk.DISABLED if typing_active else tk.NORMAL

    if typing_active: start_button.config(state=tk.DISABLED, bg=button_disabled_bg_color); stop_button.config(state=tk.NORMAL, bg=current_theme["button_bg"])
    else: start_button.config(state=tk.NORMAL, bg=current_theme["button_bg"]); stop_button.config(state=tk.DISABLED, bg=button_disabled_bg_color)

    delay_entry.config(state=entry_state); speed_entry.config(state=entry_state)
    min_erratic_speed_entry.config(state=entry_state); max_erratic_speed_entry.config(state=entry_state)
    mistake_prob_entry.config(state=entry_state); loop_count_entry.config(state=entry_state)
    erratic_checkbutton.config(state=entry_state); dark_mode_checkbox.config(state=entry_state)
    loop_typing_checkbox.config(state=entry_state)
    always_on_top_checkbox.config(state=entry_state) # New checkbox
    open_utility_dialog_button.config(state=button_state)

    if not typing_active: toggle_erratic_dependent_options(); toggle_loop_count_entry_state()

def toggle_erratic_dependent_options(event=None): # Add event for binding
    if 'start_button' not in globals() or not start_button.winfo_exists(): return
    is_erratic = erratic_var.get()
    main_typing_active = start_button['state'] == tk.DISABLED
    state_normal = tk.NORMAL if not main_typing_active else tk.DISABLED
    state_disabled = tk.DISABLED
    
    speed_entry.config(state=state_disabled if is_erratic else state_normal)
    min_erratic_speed_entry.config(state=state_normal if is_erratic else state_disabled)
    max_erratic_speed_entry.config(state=state_normal if is_erratic else state_disabled)
    mistake_prob_entry.config(state=state_normal if is_erratic else state_disabled)

    apply_theme_to_widget(speed_entry, current_theme)
    apply_theme_to_widget(min_erratic_speed_entry, current_theme)
    apply_theme_to_widget(max_erratic_speed_entry, current_theme)
    apply_theme_to_widget(mistake_prob_entry, current_theme)
    update_estimated_time()

def toggle_loop_count_entry_state(event=None): # Add event for binding
    if 'start_button' not in globals() or not start_button.winfo_exists(): return
    main_typing_active = start_button['state'] == tk.DISABLED
    state = tk.NORMAL if loop_typing_var.get() and not main_typing_active else tk.DISABLED
    loop_count_entry.config(state=state)
    apply_theme_to_widget(loop_count_entry, current_theme)

def toggle_always_on_top(event=None): # Add event for binding
    if 'root' in globals() and root.winfo_exists():
        root.attributes("-topmost", always_on_top_var.get())
        status_var.set(f"Always on Top: {'Enabled' if always_on_top_var.get() else 'Disabled'}")

# --- Utility Dialog ---
def open_utility_dialog():
    global utility_dialog_open, utility_dialog
    if utility_dialog_open: utility_dialog.lift(); utility_dialog.focus_set(); return
    utility_dialog_open = True
    utility_dialog = tk.Toplevel(root)
    utility_dialog.title("Utilities & App Settings")
    utility_dialog.geometry("550x400") # Slightly taller for new button
    utility_dialog.resizable(True, True)
    utility_dialog.transient(root)
    utility_dialog.protocol("WM_DELETE_WINDOW", lambda: close_utility_dialog(utility_dialog))
    notebook = ttk.Notebook(utility_dialog)
    text_ops_tab = tk.Frame(notebook); notebook.add(text_ops_tab, text=' Text Operations ')
    tk.Button(text_ops_tab, text="Load from File", command=load_text_from_file_action).pack(pady=5, padx=10, fill=tk.X)
    tk.Button(text_ops_tab, text="Save to File", command=save_text_to_file_action).pack(pady=5, padx=10, fill=tk.X)
    tk.Button(text_ops_tab, text="Paste from Clipboard", command=paste_from_clipboard_action).pack(pady=5, padx=10, fill=tk.X)
    tk.Button(text_ops_tab, text="Copy to Clipboard", command=copy_to_clipboard_action).pack(pady=5, padx=10, fill=tk.X) # New button
    tk.Button(text_ops_tab, text="Clear Main Text Area", command=clear_text_area_action).pack(pady=5, padx=10, fill=tk.X)
    cleaning_tab = tk.Frame(notebook); notebook.add(cleaning_tab, text=' Text Cleaning ')
    tk.Checkbutton(cleaning_tab, text="Trim Leading/Trailing Whitespace", variable=trim_whitespace_var).pack(anchor='w', padx=10, pady=2)
    tk.Checkbutton(cleaning_tab, text="Normalize Line Endings (to \\n)", variable=normalize_line_endings_var).pack(anchor='w', padx=10, pady=2)
    tk.Checkbutton(cleaning_tab, text="Remove Extra Blank Lines", variable=remove_extra_blank_lines_var).pack(anchor='w', padx=10, pady=2)
    tk.Button(cleaning_tab, text="Apply Cleaning to Main Text", command=apply_text_cleaning_action).pack(pady=10, padx=10, fill=tk.X)
    tk.Checkbutton(cleaning_tab, text="Apply Cleaning on Start Typing", variable=apply_cleaning_on_start_var).pack(anchor='w', padx=10, pady=5)
    app_settings_tab = tk.Frame(notebook); notebook.add(app_settings_tab, text=' App Settings ')
    tk.Button(app_settings_tab, text="Save Current Settings to File", command=save_settings_to_file_action).pack(pady=5, padx=10, fill=tk.X)
    tk.Button(app_settings_tab, text="Load Settings from File", command=load_settings_from_file_action).pack(pady=5, padx=10, fill=tk.X)
    tk.Button(app_settings_tab, text="Reset All Settings to Defaults", command=lambda: reset_all_settings_action(silent=False)).pack(pady=10, padx=10, fill=tk.X)
    notebook.pack(expand=True, fill='both', padx=10, pady=10)
    close_button_dialog = tk.Button(utility_dialog, text="Close Utilities", command=lambda: close_utility_dialog(utility_dialog))
    close_button_dialog.pack(pady=(0,10))
    apply_theme_to_widget(utility_dialog, current_theme)
    utility_dialog.grab_set(); utility_dialog.wait_window()

def close_utility_dialog(dialog):
    global utility_dialog_open
    utility_dialog_open = False; dialog.destroy()

# --- Core Typing Logic --- (Identical to previous, just ensure vars are correct)
def start_typing_action():
    stop_event.clear()
    if apply_cleaning_on_start_var.get(): apply_text_cleaning_action()
    text_to_type_full = text_area.get("1.0", tk.END).strip()
    try: selected_text = text_area.get(tk.SEL_FIRST, tk.SEL_LAST).strip()
    except tk.TclError: selected_text = ""
    text_to_type = selected_text if selected_text else text_to_type_full
    if not text_to_type: status_var.set("Error: No text to type!"); return
    is_erratic_mode = erratic_var.get()
    try:
        delay_seconds = delay_var.get()
        if not (0 <= delay_seconds <= 600): status_var.set("Error: Delay 0-600s."); return
        type_interval_val, min_erratic_speed_val, max_erratic_speed_val, mistake_probability_decimal = 0.0, 0.0, 0.0, 0.0
        if is_erratic_mode:
            min_erratic_speed_val = min_erratic_speed_var.get(); max_erratic_speed_val = max_erratic_speed_var.get()
            mistake_prob_percent_val = mistake_prob_var.get()
            if not (0.0 <= min_erratic_speed_val <= 5.0 and 0.0 <= max_erratic_speed_val <= 5.0): status_var.set("Error: Erratic speeds 0.0-5.0s."); return
            if min_erratic_speed_val > max_erratic_speed_val: status_var.set("Error: Min erratic > Max."); return
            if not (0.0 <= mistake_prob_percent_val <= 100.0): status_var.set("Error: Mistake Prob 0-100%."); return
            mistake_probability_decimal = mistake_prob_percent_val / 100.0
        else:
            type_interval_val = speed_var.get()
            if not (0.0 <= type_interval_val <= 5.0): status_var.set("Error: Interval 0.0-5.0s."); return
        num_loops = loop_count_var.get() if loop_typing_var.get() else 1
        if not (isinstance(num_loops, int) and num_loops >= 1): status_var.set("Error: Loop positive int."); return
    except (tk.TclError, ValueError): status_var.set("Error: Invalid numeric settings."); return
    toggle_input_states(typing_active=True)
    threading.Thread(target=_typing_thread_task, daemon=True, kwargs={
        "text_to_type": text_to_type, "delay": delay_seconds, "is_erratic": is_erratic_mode,
        "regular_interval": type_interval_val, "min_erratic": min_erratic_speed_val,
        "max_erratic": max_erratic_speed_val, "mistake_prob": mistake_probability_decimal,
        "num_loops": num_loops}).start()

def stop_typing_action():
    if not stop_event.is_set(): root.after(0, status_var.set, "Stopping..."); stop_event.set()

def _typing_thread_task(text_to_type, delay, is_erratic, regular_interval, min_erratic, max_erratic, mistake_prob, num_loops):
    pyautogui.PAUSE = 0.0 
    try:
        for i in range(delay, 0, -1):
            check_and_raise_stop()
            status_message = f"Typing in {i}s..." if delay > 0 else "Typing NOW..."
            if i == 1 and delay > 0 : status_message = "Typing NOW..."
            root.after(0, lambda sm=status_message: status_var.set(sm + " Switch window!"))
            time.sleep(1)
        if (delay > 0 or text_to_type) and num_loops > 0 : root.after(0, status_var.set, "Typing... Switch window!"); time.sleep(0.1)
        for loop_iter in range(num_loops):
            check_and_raise_stop()
            root.after(0, lambda li=loop_iter+1, nl=num_loops: status_var.set(f"Loop {li}/{nl}..." if nl > 1 else "Typing..."))
            words = text_to_type.split(' '); num_total_words = len(words); word_streak_for_pause = 0
            words_until_long_pause = random.randint(3, 5) if is_erratic else float('inf')
            for word_idx, word in enumerate(words):
                for char_idx, char_in_word in enumerate(word):
                    check_and_raise_stop()
                    if is_erratic and char_in_word.isalpha() and random.random() < mistake_prob:
                        wrong_char = random.choice(string.ascii_lowercase); wrong_char = wrong_char.upper() if char_in_word.isupper() else wrong_char
                        if wrong_char == char_in_word: wrong_char = random.choice([c for c in string.ascii_lowercase if c != char_in_word.lower()]); wrong_char = wrong_char.upper() if char_in_word.isupper() else wrong_char
                        pyautogui.write(wrong_char)
                        check_and_raise_stop(); time.sleep(random.uniform(MISTAKE_VISIBILITY_PAUSE_MIN, MISTAKE_VISIBILITY_PAUSE_MAX))
                        check_and_raise_stop(); pyautogui.press('backspace')
                        check_and_raise_stop(); time.sleep(random.uniform(CORRECTION_ACTION_PAUSE_MIN, CORRECTION_ACTION_PAUSE_MAX))
                    check_and_raise_stop(); pyautogui.write(char_in_word)
                    char_sleep_interval = random.uniform(min(min_erratic, max_erratic), max(min_erratic, max_erratic)) if is_erratic else regular_interval
                    time.sleep(char_sleep_interval)
                    if is_erratic and char_in_word in PUNCTUATION_CHARS: time.sleep(random.uniform(PUNCTUATION_EXTRA_PAUSE_MIN, PUNCTUATION_EXTRA_PAUSE_MAX))
                if is_erratic: word_streak_for_pause += 1
                if word_idx < num_total_words - 1:
                    check_and_raise_stop(); pyautogui.write(' ')
                    time.sleep(random.uniform(min(min_erratic, max_erratic), max(min_erratic, max_erratic)) if is_erratic else regular_interval)
                if is_erratic and word_streak_for_pause >= words_until_long_pause and word_idx < num_total_words - 1:
                    check_and_raise_stop(); time.sleep(random.uniform(ERRATIC_WORD_GROUP_PAUSE_MIN, ERRATIC_WORD_GROUP_PAUSE_MAX))
                    word_streak_for_pause = 0; words_until_long_pause = random.randint(3, 5)
            if loop_iter < num_loops - 1: check_and_raise_stop(); pyautogui.write(' '); time.sleep(0.5)
        root.after(0, status_var.set, "Typing complete! Ready.")
    except TypingStoppedException: root.after(0, status_var.set, "Typing stopped by user. Ready.")
    except Exception as e: root.after(0, lambda em=str(e) if e else "Unknown error": status_var.set(f"Error: {em}"))
    finally: stop_event.clear(); root.after(0, lambda: toggle_input_states(typing_active=False))

# --- Hotkey Listener ---
def on_emergency_stop_press(key):
    try:
        if key == keyboard.Key.esc and 'start_button' in globals() and start_button.winfo_exists() and start_button['state'] == tk.DISABLED:
            if root and root.winfo_exists(): root.after(0, stop_typing_action)
            else: stop_event.set()
    except Exception as e: print(f"Error in on_emergency_stop_press: {e}")
def start_keyboard_listener():
    global keyboard_listener
    if PYNPUT_AVAILABLE:
        try: keyboard_listener = keyboard.Listener(on_press=on_emergency_stop_press); threading.Thread(target=keyboard_listener.start, daemon=True).start(); print("Esc listener started.")
        except Exception as e: print(f"Failed to start keyboard listener: {e}")
def stop_keyboard_listener():
    if PYNPUT_AVAILABLE and keyboard_listener and keyboard_listener.is_alive():
        try: keyboard_listener.stop(); print("Esc listener stopped.")
        except Exception as e: print(f"Error stopping keyboard listener: {e}")

# --- GUI Setup ---
root = tk.Tk()
root.title("Practical Quick Typer")
root.geometry("600x720") # Adjusted size
root.resizable(True, True)

# Initialize Tkinter Variables
delay_var = tk.IntVar(value=DEFAULT_DELAY_SEC)
speed_var = tk.DoubleVar(value=DEFAULT_SPEED_S_CHAR)
erratic_var = tk.BooleanVar(value=False)
min_erratic_speed_var = tk.DoubleVar(value=DEFAULT_ERRATIC_MIN_S)
max_erratic_speed_var = tk.DoubleVar(value=DEFAULT_ERRATIC_MAX_S)
mistake_prob_var = tk.DoubleVar(value=DEFAULT_MISTAKE_PROBABILITY_PERCENT)
dark_mode_var = tk.BooleanVar(value=False)
loop_typing_var = tk.BooleanVar(value=False)
loop_count_var = tk.IntVar(value=DEFAULT_LOOP_COUNT)
trim_whitespace_var = tk.BooleanVar(value=False)
normalize_line_endings_var = tk.BooleanVar(value=False)
remove_extra_blank_lines_var = tk.BooleanVar(value=False)
apply_cleaning_on_start_var = tk.BooleanVar(value=False)
always_on_top_var = tk.BooleanVar(value=False)
char_count_var = tk.StringVar(value="Chars: 0")
word_count_var = tk.StringVar(value="Words: 0")
estimated_time_var = tk.StringVar(value="Est. Time: 0s")

# --- Main Layout Frames ---
text_area_frame = tk.Frame(root)
text_area_frame.pack(pady=5, padx=10, fill=tk.BOTH, expand=True)
main_controls_frame = tk.Frame(root)
main_controls_frame.pack(pady=5, padx=10, fill=tk.X)
status_bar_frame = tk.Frame(root)
status_bar_frame.pack(side=tk.BOTTOM, fill=tk.X)

# --- Text Area Frame Content ---
instruction_label = tk.Label(text_area_frame, text="1. Paste text. 2. Adjust settings. 3. Click 'Start Typing', then switch window." + ("\n(Esc to stop)" if PYNPUT_AVAILABLE else ""), justify=tk.LEFT)
instruction_label.pack(pady=(0,5), anchor='w')
text_area = scrolledtext.ScrolledText(text_area_frame, wrap=tk.WORD, height=15)
text_area.pack(pady=5, fill=tk.BOTH, expand=True)
text_area.focus()
text_area.bind("<KeyRelease>", update_counts_and_time) # Update time too

counts_frame = tk.Frame(text_area_frame)
counts_frame.pack(fill=tk.X, pady=(2,0))
tk.Label(counts_frame, textvariable=char_count_var, anchor='w').pack(side=tk.LEFT, padx=5)
tk.Label(counts_frame, textvariable=word_count_var, anchor='w').pack(side=tk.LEFT, padx=5)
tk.Label(counts_frame, textvariable=estimated_time_var, anchor='w').pack(side=tk.LEFT, padx=10) # New Estimated Time

# --- Main Controls Frame Content ---
typing_params_labelframe = tk.LabelFrame(main_controls_frame, text="Typing Parameters")
typing_params_labelframe.pack(fill=tk.X, pady=5)
params_grid_frame = tk.Frame(typing_params_labelframe)
params_grid_frame.pack(padx=5, pady=5, fill=tk.X)

# Row 0: Delay, Dark Mode, Always on Top
tk.Label(params_grid_frame, text="Delay (sec):").grid(row=0, column=0, sticky="w", pady=2, padx=5)
delay_entry = tk.Entry(params_grid_frame, textvariable=delay_var, width=7)
delay_entry.grid(row=0, column=1, sticky="w", pady=2, padx=5)
delay_entry.bind("<KeyRelease>", update_estimated_time) # Update time on change

# Sub-frame for right-aligned checkboxes
right_checkbox_frame = tk.Frame(params_grid_frame)
right_checkbox_frame.grid(row=0, column=2, columnspan=2, sticky="e", pady=0, padx=0)
dark_mode_checkbox = tk.Checkbutton(right_checkbox_frame, text="Dark Mode", variable=dark_mode_var, command=toggle_main_theme)
dark_mode_checkbox.pack(side=tk.RIGHT, padx=(5,0))
always_on_top_checkbox = tk.Checkbutton(right_checkbox_frame, text="Always on Top", variable=always_on_top_var, command=toggle_always_on_top)
always_on_top_checkbox.pack(side=tk.RIGHT, padx=(10,5))


# Row 1: Interval, Erratic Speed CB
tk.Label(params_grid_frame, text="Interval (s/char):").grid(row=1, column=0, sticky="w", pady=2, padx=5)
speed_entry = tk.Entry(params_grid_frame, textvariable=speed_var, width=7)
speed_entry.grid(row=1, column=1, sticky="w", pady=2, padx=5)
speed_entry.bind("<KeyRelease>", update_estimated_time)

erratic_checkbutton = tk.Checkbutton(params_grid_frame, text="Erratic Speed", variable=erratic_var, command=toggle_erratic_dependent_options)
erratic_checkbutton.grid(row=1, column=2, columnspan=2, sticky="e", pady=2, padx=10)

# Row 2, 3, 4: Erratic Speed Options (aligned under Interval/Delay)
tk.Label(params_grid_frame, text="Min Erratic (s):").grid(row=2, column=0, sticky="w", pady=2, padx=5)
min_erratic_speed_entry = tk.Entry(params_grid_frame, textvariable=min_erratic_speed_var, width=7)
min_erratic_speed_entry.grid(row=2, column=1, sticky="w", pady=2, padx=5)
min_erratic_speed_entry.bind("<KeyRelease>", update_estimated_time)

tk.Label(params_grid_frame, text="Max Erratic (s):").grid(row=3, column=0, sticky="w", pady=2, padx=5)
max_erratic_speed_entry = tk.Entry(params_grid_frame, textvariable=max_erratic_speed_var, width=7)
max_erratic_speed_entry.grid(row=3, column=1, sticky="w", pady=2, padx=5)
max_erratic_speed_entry.bind("<KeyRelease>", update_estimated_time)

tk.Label(params_grid_frame, text="Mistake Prob (%):").grid(row=4, column=0, sticky="w", pady=2, padx=5)
mistake_prob_entry = tk.Entry(params_grid_frame, textvariable=mistake_prob_var, width=7)
mistake_prob_entry.grid(row=4, column=1, sticky="w", pady=2, padx=5)
mistake_prob_entry.bind("<KeyRelease>", update_estimated_time)

# Row 5: Loop Typing
loop_typing_checkbox = tk.Checkbutton(params_grid_frame, text="Loop Typing:", variable=loop_typing_var, command=toggle_loop_count_entry_state)
loop_typing_checkbox.grid(row=5, column=0, sticky="w", pady=2, padx=5)
loop_count_entry = tk.Entry(params_grid_frame, textvariable=loop_count_var, width=5)
loop_count_entry.grid(row=5, column=1, sticky="w", pady=2, padx=5)

params_grid_frame.columnconfigure(3, weight=1) # Allows right_checkbox_frame to align right

action_utility_frame = tk.Frame(main_controls_frame)
action_utility_frame.pack(fill=tk.X, pady=10)
start_button = tk.Button(action_utility_frame, text="Start Typing", command=start_typing_action, width=15)
start_button.pack(side=tk.LEFT, padx=5, expand=True)
stop_button = tk.Button(action_utility_frame, text="Stop Typing", command=stop_typing_action, width=15, state=tk.DISABLED)
stop_button.pack(side=tk.LEFT, padx=5, expand=True)
open_utility_dialog_button = tk.Button(action_utility_frame, text="Utilities & App Settings...", command=open_utility_dialog, width=20)
open_utility_dialog_button.pack(side=tk.RIGHT, padx=5, expand=True) # Changed to RIGHT

# --- Status Bar ---
status_var = tk.StringVar()
status_label = tk.Label(status_bar_frame, textvariable=status_var, bd=1, relief=tk.SUNKEN, anchor=tk.W)
status_label.pack(fill=tk.X)

# --- App Closing Protocol ---
def on_app_closing():
    global utility_dialog_open
    stop_keyboard_listener()
    if stop_event and not stop_event.is_set(): stop_event.set()
    if utility_dialog_open and 'utility_dialog' in globals() and utility_dialog.winfo_exists():
        utility_dialog.destroy(); utility_dialog_open = False
    try:
        if messagebox.askyesno("Save Settings?", "Save current settings before closing?"):
            save_settings_to_file_action()
    except Exception as e: print(f"Error during pre-close save: {e}")
    root.destroy()
root.protocol("WM_DELETE_WINDOW", on_app_closing)

# --- Initial UI State Setup ---
reset_all_settings_action(silent=True)
try:
    if os.path.exists(SETTINGS_FILENAME):
        with open(SETTINGS_FILENAME, "r") as f: loaded_s = json.load(f)
        apply_all_settings(loaded_s)
        status_var.set(f"Auto-loaded settings from {SETTINGS_FILENAME}")
    else: status_var.set("No settings file found. Using defaults.")
except Exception as e: status_var.set(f"Error auto-loading: {e}. Using defaults."); reset_all_settings_action(silent=True)

update_counts_and_time() # Initial update
start_keyboard_listener()
toggle_input_states(typing_active=False)

root.mainloop()



License:
This project is licensed under the MIT License. See the LICENSE file for details.

Disclaimer:
This tool is intended for personal, educational, and non-critical use cases. The author assumes no responsibility for unintended consequences, misuse, or data loss resulting from its operation.

