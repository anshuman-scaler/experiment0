The issue you are seeing ("bursty" logs, key presses bunching up) is a classic **Producer-Consumer bottleneck** caused by the Global Interpreter Lock (GIL) and Windows Message Pump saturation.

Here is what is happening:
1.  **Heavy Callbacks:** You are performing logic inside the callbacks (`on_move`, `on_press`). You are calling `self.metadata_manager.scale_coordinates(x, y)` and `_normalize_key_name`. Even if these take 0.1ms, `on_move` fires hundreds of times a second. This floods the Windows event hook, causing the OS to buffer "Release" events until your app catches up.
2.  **Mouse Flood:** Mouse movements are spamming your Event Queue. This keeps the Python GIL locked, preventing the `Recorder` thread from processing the queue and writing to disk efficiently.
3.  **Late Timestamping:** If the callback is delayed by the OS because the previous callback was slow, `time.perf_counter()` captures the time *after* the delay, not when the key was actually pressed.

### The Fix
1.  **Zero-Logic Callbacks:** The callback must do **nothing** except grab the timestamp and put raw data into the queue. Move scaling and normalization to the background thread.
2.  **Throttle Mouse Events:** You do not need 1000Hz mouse logging for video. Throttling to 60Hz (16ms) drastically reduces CPU load and unblocks keyboard events.
3.  **Optimized File I/O:** Write to the file in batches or let the OS buffer effectively, but ensuring the consumer loop is tight.

Here is the optimized `recorder.py`:

```python
import json
import os
import re
import time
from datetime import datetime
from platform import system
from queue import Queue, Empty

from pynput import keyboard, mouse
from pynput.keyboard import KeyCode
from PyQt6.QtCore import QThread, pyqtSignal

from .logger import get_logger
from .metadata import MetadataManager
from .screen_recorder import ScreenRecorder
from .util import fix_windows_dpi_scaling, get_recordings_dir

# Import Windows scroll fix for trackpad support
if system() == "Windows":
    from .windows_scroll_fix import start_windows_scroll_hook, stop_windows_scroll_hook


class Recorder(QThread):
    """
    Makes recordings. Optimized for low-latency event capture.
    """
    
    recording_stopped = pyqtSignal()
    initialization_failed = pyqtSignal(str)

    def __init__(self, natural_scrolling: bool, resolution: str = "1920x1080", fps: int = 30, task_uuid: str = None, task_slug: str = None):
        super().__init__()
        self.logger = get_logger('taggr.recorder')
        
        self.natural_scrolling = natural_scrolling
        self.resolution = resolution
        self.fps = fps
        self.task_uuid = task_uuid
        self.task_slug = task_slug
        
        # Throttling state for mouse moves
        self._last_move_ts = 0
        self._move_throttle_interval = 0.016  # ~60Hz (16ms)
        
        if system() == "Windows":
            fix_windows_dpi_scaling()
            
        self.recording_path = self._get_recording_path()
        self._is_recording = False
        self._stop_timestamp_ms = None
        self.windows_scroll_hook_active = False

        self.event_queue = None
        self.events_file = None
        self.metadata_manager = None
        self.screen_recorder = None
        self.mouse_listener = None
        self.keyboard_listener = None
        
    # =========================================================================
    # OPTIMIZED CALLBACKS (Zero Logic)
    # =========================================================================
    
    def on_move(self, x, y):
        # 1. Timestamp immediately
        now = time.perf_counter()
        
        # 2. Throttle: If less than 16ms passed since last move, ignore.
        # This prevents mouse events from starving keyboard events in the queue.
        if now - self._last_move_ts < self._move_throttle_interval:
            return
        self._last_move_ts = now
        
        # 3. Put RAW data only. No scaling, no math.
        # We use a tuple for speed: (type, timestamp, data_args...)
        self.event_queue.put(("move", now * 1000, x, y), block=False)
        
    def on_click(self, x, y, button, pressed):
        now = time.perf_counter()
        # No throttling for clicks, we want them all
        self.event_queue.put(("click", now * 1000, x, y, button, pressed), block=False)
    
    def on_scroll(self, x, y, dx, dy):
        now = time.perf_counter()
        self.event_queue.put(("scroll", now * 1000, x, y, dx, dy), block=False)
    
    def _windows_scroll_callback(self, x, y, dx, dy):
        now = time.perf_counter()
        self.event_queue.put(("scroll", now * 1000, x, y, dx, dy), block=False)

    def on_press(self, key):
        now = time.perf_counter()
        # Pass the raw key object. Normalize later.
        self.event_queue.put(("press", now * 1000, key), block=False)

    def on_release(self, key):
        now = time.perf_counter()
        self.event_queue.put(("release", now * 1000, key), block=False)

    # =========================================================================
    # PROCESSING LOGIC (Moved from callbacks to Consumer Thread)
    # =========================================================================

    def _process_event(self, raw_event):
        """Do the heavy lifting here, not in the input thread."""
        evt_type = raw_event[0]
        ts_ms = raw_event[1]
        
        processed_event = None

        if evt_type == "move":
            _, _, x, y = raw_event
            scaled_x, scaled_y = self.metadata_manager.scale_coordinates(x, y)
            processed_event = {
                "time_stamp_ms": ts_ms,
                "action": "move",
                "x": scaled_x,
                "y": scaled_y
            }

        elif evt_type == "click":
            _, _, x, y, button, pressed = raw_event
            scaled_x, scaled_y = self.metadata_manager.scale_coordinates(x, y)
            processed_event = {
                "time_stamp_ms": ts_ms,
                "action": "click",
                "x": scaled_x,
                "y": scaled_y,
                "button": button.name,
                "pressed": pressed
            }

        elif evt_type == "scroll":
            _, _, x, y, dx, dy = raw_event
            scaled_x, scaled_y = self.metadata_manager.scale_coordinates(x, y)
            processed_event = {
                "time_stamp_ms": ts_ms,
                "action": "scroll",
                "x": scaled_x,
                "y": scaled_y,
                "dx": dx,
                "dy": dy
            }

        elif evt_type == "press":
            _, _, key = raw_event
            processed_event = {
                "time_stamp_ms": ts_ms,
                "action": "press",
                "name": self._normalize_key_name(key)
            }

        elif evt_type == "release":
            _, _, key = raw_event
            processed_event = {
                "time_stamp_ms": ts_ms,
                "action": "release",
                "name": self._normalize_key_name(key)
            }

        return processed_event

    def run(self):
        self.logger.info("Starting recording session")
        self._is_recording = True
        
        try:
            # Initialization (Same as before)
            self.event_queue = Queue()
            events_file_path = os.path.join(self.recording_path, "events.jsonl")
            # Use a buffer size (e.g., 8KB) to reduce disk I/O syscalls
            self.events_file = open(events_file_path, "a", buffering=8192)
            
            self.metadata_manager = MetadataManager(
                recording_path=self.recording_path, 
                natural_scrolling=self.natural_scrolling,
                video_resolution=self.resolution,
                fps=self.fps,
                task_uuid=self.task_uuid,
                task_slug=self.task_slug
            )
            self.metadata_manager.collect()
            
            # Initialize Video
            try:
                self.screen_recorder = ScreenRecorder(
                    output_path=self.recording_path,
                    resolution=self.resolution,
                    fps=self.fps
                )
            except RuntimeError as e:
                 # ... Error handling ...
                 self.initialization_failed.emit(str(e))
                 return

            if not self.screen_recorder.start_recording():
                self.initialization_failed.emit("Failed to start recording")
                return
            
            recording_start_timestamp_ms = time.perf_counter() * 1000
            self.metadata_manager.set_recording_start_time(recording_start_timestamp_ms)

            # Wait for First Frame (Critical for sync)
            wait_start = time.time()
            max_wait = 3.0
            first_frame_detected = False
            
            while time.time() - wait_start < max_wait:
                if self.screen_recorder.get_first_frame_timestamp_ms() is not None:
                    first_frame_detected = True
                    break
                time.sleep(0.05)
            
            if not first_frame_detected:
                self.logger.warning("Starting capture without first frame confirmation")

            # Start Input Listeners
            self.mouse_listener = mouse.Listener(
                on_move=self.on_move, on_click=self.on_click, on_scroll=self.on_scroll)
            self.keyboard_listener = keyboard.Listener(
                on_press=self.on_press, on_release=self.on_release)
            
            self.mouse_listener.start()
            self.keyboard_listener.start()
            
            input_capture_start_timestamp_ms = time.perf_counter() * 1000
            self.metadata_manager.set_input_capture_start_time(input_capture_start_timestamp_ms)
            
            # Video start reference
            video_start_ms = self.screen_recorder.get_first_frame_timestamp_ms() or input_capture_start_timestamp_ms

            # -----------------------------------------------------------------
            # MAIN CONSUMER LOOP
            # -----------------------------------------------------------------
            while self._is_recording or not self.event_queue.empty():
                try:
                    # Timeout allows checking self._is_recording periodically
                    raw_event = self.event_queue.get(timeout=0.1)
                except Empty:
                    continue

                # Filter late events
                if self._stop_timestamp_ms is not None:
                    # Check raw timestamp (index 1 in tuple)
                    if raw_event[1] > self._stop_timestamp_ms:
                        continue
                
                # Convert raw data to JSON-ready dict
                final_event = self._process_event(raw_event)
                
                if final_event:
                    final_event["second_in_video"] = (final_event["time_stamp_ms"] - video_start_ms) / 1000.0
                    self.events_file.write(json.dumps(final_event) + "\n")

            self.logger.info("Event loop finished")
            
        except Exception as e:
            self.logger.error(f"Error: {e}", exc_info=True)
            self.initialization_failed.emit(str(e))
        finally:
            # Cleanup ensures file is flushed/closed
            if self.events_file:
                self.events_file.flush()
                self.events_file.close()

    # ... (Keep stop_recording, _calculate_frame_sync_drift, _normalize_key_name, _get_recording_path as they were) ...
```

### Why this fixes the lag:
1.  **Callback Latency:** The `pynput` thread now executes roughly 5 lines of trivial code (get time, check throttle, put in queue). It returns instantly to the OS, so Windows doesn't delay the next event message.
2.  **Mouse Throttle:** `on_move` now only fires ~60 times a second instead of ~500-1000. This is the biggest factor. High-DPI mice can generate thousands of interrupts. If you queue all of them, the consumer thread falls behind, and keyboard events stuck in the same queue get delayed.
3.  **Correct Timestamp:** We call `time.perf_counter()` as the *first instruction* in the callback. Even if the consumer thread is 5 seconds behind (processing a backlog), the timestamp recorded is the *creation* time, not the *processing* time.

This will ensure your key presses and releases appear in the log with the correct millisecond delta, matching your physical typing speed.
