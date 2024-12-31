# project
circular buffer on weather
import tkinter as tk
from tkinter import ttk
import random
import time
from matplotlib.figure import Figure
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

class CircularBuffer:
    def __init__(self, capacity):  
        self.capacity = capacity
        self.buffer = [0] * capacity
        self.head = 0
        self.tail = 0
        self.size = 0
        self.timestamps = [0] * capacity

    def push(self, item, timestamp):
        self.buffer[self.head] = item
        self.timestamps[self.head] = timestamp
        self.head = (self.head + 1) % self.capacity
        if self.size < self.capacity:
            self.size += 1
        else:
            self.tail = (self.tail + 1) % self.capacity

    def get_all(self):
        if self.size == 0:
            return [], []
        if self.tail < self.head:
            return (self.buffer[self.tail:self.head], 
                   self.timestamps[self.tail:self.head])
        return (self.buffer[self.tail:] + self.buffer[:self.head],
                self.timestamps[self.tail:] + self.timestamps[:self.head])

class SensorSimulator:
    @staticmethod
    def get_reading():
        return random.uniform(20.0, 30.0)

class DataProcessorGUI:
    def __init__(self, root):  
        self.root = root
        self.root.title("Advanced Temperature Monitor")
        self.root.geometry("1100x600")
        
        # Initialize components
        self.buffer = CircularBuffer(30)
        self.sensor = SensorSimulator()
        self.start_time = time.time()
        self.running = False
        
        # Apply theme
        self.setup_theme()
        self.setup_gui()

    def setup_theme(self):
        style = ttk.Style()
        style.theme_use('clam')
        
        # Define colors
        self.colors = {
            'primary': '#5442c9',
            'background': '#95bdb5',
            'text': '#0a0a0a',
            'text_secondary': '#ed1a1a'
        }
        
        # Configure styles
        style.configure('Main.TFrame', background=self.colors['background'])
        style.configure('Value.TLabel',
                       font=('Helvetica', 24),
                       foreground=self.colors['primary'])
        style.configure('Title.TLabel',
                       font=('Helvetica', 12, 'bold'),
                       foreground=self.colors['text'])
        style.configure('Status.TLabel',
                       font=('Helvetica', 10),
                       foreground=self.colors['text_secondary'])

    def setup_gui(self):
        # Main container
        main_frame = ttk.Frame(self.root, style='Main.TFrame', padding="10")
        main_frame.pack(fill=tk.BOTH, expand=True)

        # Stats Panel
        stats_frame = ttk.Frame(main_frame, style='Main.TFrame')
        stats_frame.pack(side=tk.LEFT, fill=tk.Y, padx=(0, 10))

        # Current Reading
        ttk.Label(stats_frame, text="Current Reading", 
                 style='Title.TLabel').pack(pady=(0, 5))
        self.current_reading = ttk.Label(stats_frame, text="-- °C", 
                                       style='Value.TLabel')
        self.current_reading.pack(pady=(0, 20))

        # Moving Average
        ttk.Label(stats_frame, text="Moving Average", 
                 style='Title.TLabel').pack(pady=(0, 5))
        self.moving_avg = ttk.Label(stats_frame, text="-- °C", 
                                  style='Value.TLabel')
        self.moving_avg.pack(pady=(0, 20))

        # Min/Max
        ttk.Label(stats_frame, text="Min/Max", 
                 style='Title.TLabel').pack(pady=(0, 5))
        self.minmax = ttk.Label(stats_frame, text="-- / --", 
                              style='Value.TLabel')
        self.minmax.pack(pady=(0, 20))

        # Graph Panel
        graph_frame = tk.Frame(main_frame, bg='white')
        graph_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        # Setup graph
        self.figure = Figure(figsize=(6, 4), dpi=100)
        self.plot = self.figure.add_subplot(111)
        self.plot.set_title('Temperature Readings')
        self.plot.set_xlabel('Time (s)')
        self.plot.set_ylabel('Temperature (°C)')
        
        self.canvas = FigureCanvasTkAgg(self.figure, master=graph_frame)
        self.canvas.draw()
        self.canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True)

        # Control Panel
        control_frame = ttk.Frame(main_frame, style='Main.TFrame')
        control_frame.pack(side=tk.BOTTOM, fill=tk.X, pady=(10, 0))

        # Buttons
        self.start_button = ttk.Button(control_frame, text="Start", 
                                     command=self.start_processing)
        self.start_button.pack(side=tk.LEFT, padx=5)

        self.stop_button = ttk.Button(control_frame, text="Stop", 
                                    command=self.stop_processing,
                                    state=tk.DISABLED)
        self.stop_button.pack(side=tk.LEFT, padx=5)

        # Status
        self.status = ttk.Label(control_frame, text="Status: Idle", 
                              style='Status.TLabel')
        self.status.pack(side=tk.RIGHT)

    def start_processing(self):
        self.running = True
        self.start_button.config(state=tk.DISABLED)
        self.stop_button.config(state=tk.NORMAL)
        self.status.config(text="Status: Running")
        self.update_reading()

    def stop_processing(self):
        self.running = False
        self.start_button.config(state=tk.NORMAL)
        self.stop_button.config(state=tk.DISABLED)
        self.status.config(text="Status: Stopped")

    def update_reading(self):
        if not self.running:  # Check if processing should stop
            return

        # Get new reading
        reading = self.sensor.get_reading()
        current_time = time.time() - self.start_time
        self.buffer.push(reading, current_time)

        # Get data for display
        temperatures, timestamps = self.buffer.get_all()
        avg = sum(temperatures) / len(temperatures) if temperatures else 0
        min_temp = min(temperatures) if temperatures else 0
        max_temp = max(temperatures) if temperatures else 0

        # Update stats
        self.current_reading.config(text=f"{reading:.1f} °C")
        self.moving_avg.config(text=f"{avg:.1f} °C")
        self.minmax.config(text=f"{min_temp:.1f} / {max_temp:.1f}")

        # Update graph
        self.plot.clear()
        self.plot.plot(timestamps, temperatures, 'b-')
        self.plot.set_title('Temperature Readings')
        self.plot.set_xlabel('Time (s)')
        self.plot.set_ylabel('Temperature (°C)')
        self.plot.grid(True)
        self.canvas.draw()

        # Schedule next update
        self.root.after(1000, self.update_reading)

def main():
    root = tk.Tk()
    app = DataProcessorGUI(root)
    root.mainloop()

if __name__ == "__main__":  
    main()
