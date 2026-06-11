# -Build-an-alarm-clock-as-a-Python-CLI-application

Building a CLI alarm clock in Python is a great exercise in managing asynchronous tasks and terminal UI updates. Since we want a robust, user-friendly CLI without over-engineering, here is the refinement of the requirements, the system design, and the implementation plan.

1. Requirements & Scope Refinement:
To make a CLI alarm clock truly usable, it needs to handle background timing without freezing the terminal, and it should allow the user to interact with it while an alarm is waiting.

Core Features (The "Must-Haves")
Set Alarm: Allow users to set an alarm for a specific time (e.g., 14:30) or a countdown timer (e.g., in 10 minutes).

Active Tracking: A live background thread or loop that checks the time without blocking user input.

Acoustic/Visual Alert: Play a sound (using a lightweight cross-platform library) and flash the terminal when the alarm goes off.

Snooze/Dismiss: Ability to snooze for 5 minutes or dismiss the alarm entirely from the CLI.

Multiple Alarms: Support setting more than one alarm concurrently.

Out of Scope (For Now)
Persistent storage (alarms are lost when the app closes).

Cron-like recurring alarms (daily/weekly).

2. Architecture & Design:
A naive time.sleep() loop will lock the CLI, preventing the user from adding new alarms or canceling existing ones. We will use a Multi-threaded Architecture:

Main Thread (UI/Input Loop): Handles the interactive CLI menu, processes user commands, and prints the current status.

Background Thread (Alarm Manager): A single background daemon thread that continuously evaluates the list of active alarms against the current system time.

Component Diagram Concept:

+-------------------------------------------------------+
|                     Main Loop                         |
|  (User Input -> Add/List/Delete/Snooze Alarms)        |
+---------------------------+---------------------------+
                            | Updates Shared State
                            v
+-------------------------------------------------------+
|                 Shared Alarm List                     |
|  [ {id: 1, time: "14:30", label: "Wake up"}, ... ]   |
+---------------------------+---------------------------+
                            ^
                            | Monitors State
+---------------------------+---------------------------+
|               Background Monitor Thread                |
|  (Infinite loop checking time -> Triggers Alerts)     |
+-------------------------------------------------------+
Dependencies:

simpleaudio or playsound: For cross-platform audio playback.

threading: Built-in library to run the background monitor.

shutil / os: For clearing the screen and formatting the CLI beautifully.

3. Implementation Plan:

Step 1: Setup the data structures. An Alarm object tracking ID, target time, label, and status (active/snoozed/triggered).

Step 2: Build the background worker loop using threading.Thread(daemon=True).

Step 3: Implement the audio trigger interface.

Step 4: Build the interactive main CLI menu loop.

Step 5: Polish the terminal output (ANSI escape codes for color, clear screen mechanics).

4. The Code:

Here is the complete, self-contained Python CLI application.

(Note: To hear the audio, ensure you have a standard .wav file named alarm.wav in the same directory, or the code will fallback to a standard system beep).


# python code

import datetime
import os
import sys
import time
import threading
from dataclasses import dataclass

# Try importing a sound library, fallback to system beep if unavailable
try:
    import simpleaudio as sa
    AUDIO_AVAILABLE = True
except ImportError:
    AUDIO_AVAILABLE = False


@dataclass
class Alarm:
    id: int
    time: datetime.datetime
    label: str
    is_triggered: bool = False


class CLIAlarmClock:
    def __init__(self):
        self.alarms = []
        self.alarm_id_counter = 1
        self.lock = threading.Lock()
        self.running = True
        self.active_alert = False

    def clear_screen(self):
        os.system('cls' if os.name == 'nt' else 'clear')

    def play_alarm_sound(self):
        """Plays a sound loop until the alarm is dismissed or snoozed."""
        print("\n🔔 ALARM GOING OFF! 🔔")
        while self.active_alert:
            if AUDIO_AVAILABLE:
                try:
                    wave_obj = sa.WaveObject.from_wave_file("alarm.wav")
                    play_obj = wave_obj.play()
                    # Wait for audio to finish or alarm to be dismissed
                    while play_obj.is_playing() and self.active_alert:
                        time.sleep(0.1)
                    if not self.active_alert:
                        play_obj.stop()
                except Exception:
                    # Fallback if the wav file is missing
                    sys.stdout.write('\a')
                    sys.stdout.flush()
                    time.sleep(1)
            else:
                # System bell fallback
                sys.stdout.write('\a')
                sys.stdout.flush()
                time.sleep(1)

    def alarm_monitor(self):
        """Background thread loop checking if any alarms have hit their target time."""
        while self.running:
            now = datetime.datetime.now()
            with self.lock:
                for alarm in self.alarms:
                    if not alarm.is_triggered and now >= alarm.time:
                        alarm.is_triggered = True
                        self.active_alert = True
                        # Trigger alert handler in a separate thread so it doesn't block monitoring
                        threading.Thread(target=self.play_alarm_sound, daemon=True).start()
                        
            time.sleep(1)

    def add_alarm(self):
        self.clear_screen()
        print("=== Add New Alarm ===")
        print("1. Set absolute time (HH:MM)")
        print("2. Set countdown timer (Minutes)")
        choice = input("Select option: ").strip()

        now = datetime.datetime.now()
        label = input("Enter a label for this alarm: ").strip() or "Alarm"

        if choice == '1':
            time_str = input("Enter time (24h format, HH:MM): ").strip()
            try:
                target_time = datetime.datetime.strptime(time_str, "%H:%M")
                # Adjust to today or tomorrow depending on if the time has passed
                alarm_time = now.replace(hour=target_time.hour, minute=target_time.minute, second=0, microsecond=0)
                if alarm_time < now:
                    alarm_time += datetime.timedelta(days=1)
            except ValueError:
                print("❌ Invalid time format!")
                time.sleep(1.5)
                return
        elif choice == '2':
            minutes_str = input("Enter minutes from now: ").strip()
            try:
                minutes = int(minutes_str)
                alarm_time = now + datetime.timedelta(minutes=minutes)
            except ValueError:
                print("❌ Invalid number of minutes!")
                time.sleep(1.5)
                return
        else:
            print("❌ Invalid option!")
            time.sleep(1.5)
            return

        with self.lock:
            new_alarm = Alarm(id=self.alarm_id_counter, time=alarm_time, label=label)
            self.alarms.append(new_alarm)
            self.alarm_id_counter += 1
        
        print(f"✅ Alarm successfully set for {alarm_time.strftime('%Y-%m-%d %H:%M:%S')}")
        time.sleep(2)

    def list_alarms(self):
        self.clear_screen()
        print("=== Active Alarms ===")
        with self.lock:
            # Filter out triggered/handled alarms
            self.alarms = [a for a in self.alarms if not a.is_triggered]
            
            if not self.alarms:
                print("No active alarms.")
            else:
                for alarm in self.alarms:
                    time_str = alarm.time.strftime("%H:%M:%S")
                    print(f"[{alarm.id}] {time_str} - {alarm.label}")
        print("\nPress Enter to return to menu...")
        input()

    def handle_active_alarm(self):
        """Intervention overlay menu when an alarm goes off."""
        while self.active_alert:
            self.clear_screen()
            print("🚨 🚨 🚨 ALARM IS RINGING 🚨 🚨 🚨")
            print("1. Dismiss")
            print("2. Snooze (5 Minutes)")
            choice = input("Action: ").strip()

            if choice == '1':
                self.active_alert = False
                with self.lock:
                    self.alarms = [a for a in self.alarms if not a.is_triggered]
                print("Alarm dismissed.")
                time.sleep(1.5)
            elif choice == '2':
                self.active_alert = False
                with self.lock:
                    # Find the triggered alarm and reset its time
                    triggered_alarms = [a for a in self.alarms if a.is_triggered]
                    if triggered_alarms:
                        # Reschedule the first triggered one found for simplicity
                        main_alarm = triggered_alarms[0]
                        main_alarm.time = datetime.datetime.now() + datetime.timedelta(minutes=5)
                        main_alarm.is_triggered = False
                        main_alarm.label = f"{main_alarm.label} (Snoozed)"
                    # Clean out any other dangling triggered statuses
                    self.alarms = [a for a in self.alarms if not a.is_triggered or a == main_alarm]
                print("Snoozed for 5 minutes...")
                time.sleep(1.5)

    def main_menu(self):
        # Start background monitor thread
        monitor_thread = threading.Thread(target=self.alarm_monitor, daemon=True)
        monitor_thread.start()

        while self.running:
            # Check if an alarm hijacked the system state
            if self.active_alert:
                self.handle_active_alarm()
                continue

            self.clear_screen()
            now_str = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            print(f"====================================")
            print(f"      CLI ALARM CLOCK               ")
            print(f"      Current Time: {now_str}      ")
            print(f"====================================")
            print("1. Add Alarm / Timer")
            print("2. View Active Alarms")
            print("3. Exit")
            print("------------------------------------")
            
            # Use a short timeout concept for input or simple non-blocking structure 
            # so the time display updates gracefully if they stay idle.
            print("Select an option: ", end="", flush=True)
            
            # Simple choice evaluation
            choice = input().strip()

            if choice == '1':
                self.add_alarm()
            elif choice == '2':
                self.list_alarms()
            elif choice == '3':
                self.running = False
                print("Goodbye!")
            else:
                # If they typed nothing or garbage, just refresh loop
                continue


if __name__ == "__main__":
    clock = CLIAlarmClock()
    try:
        clock.main_menu()
    except KeyboardInterrupt:
        print("\nApplication closed via keyboard interrupt.")
        sys.exit(0)
