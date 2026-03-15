import tkinter as tk
import pyaudio
import numpy as np
import threading

# --- НАЛАШТУВАННЯ ЗВУКУ ---
CHUNK = 1024
FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 44100
THRESHOLD = 2000 

class GeometryDashLevels:
    def __init__(self, root):
        self.root = root
        self.root.title("Voice Dash: Level Selector")
        self.root.geometry("800x550")
        self.root.resizable(False, False)

        self.canvas = tk.Canvas(root, width=800, height=550, bg="#002266", highlightthickness=0)
        self.canvas.pack()

        # Звук
        self.p = pyaudio.PyAudio()
        self.stream = self.p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK)

        self.game_active = False
        self.running = True
        self.obstacle_speed = 12
        self.current_level_name = ""
        
        # Список кнопок для видалення
        self.menu_buttons = []

        threading.Thread(target=self.audio_listener, daemon=True).start()
        self.show_main_menu()

    def clear_menu_buttons(self):
        """Видаляє всі віджети кнопок з екрана"""
        for btn in self.menu_buttons:
            btn.destroy()
        self.menu_buttons = []

    def show_main_menu(self):
        """Головне меню з вибором рівнів"""
        self.canvas.delete("all")
        self.game_active = False
        self.clear_menu_buttons()
        
        # Фон
        self.canvas.create_rectangle(0, 0, 800, 550, fill="#001a44")
        self.canvas.create_text(400, 80, text="ОБЕРИ РІВЕНЬ", font=("Courier", 50, "bold"), fill="#00ff88")

        # --- КНОПКИ РІВНІВ ---
        levels = [
            ("ЛЕГКО (Easy)", 10, "#00ff88", 180),
            ("НОРМАЛЬНО (Normal)", 15, "#ffff00", 260),
            ("СКЛАДНО (Hard)", 22, "#ffaa00", 340),
            ("ДЕМОН (Extreme)", 30, "#ff3333", 420)
        ]

        for text, speed, color, y in levels:
            btn = tk.Button(
                self.root, text=text, 
                command=lambda s=speed, n=text: self.start_level(s, n),
                font=("Courier", 18, "bold"), bg=color, width=25, cursor="hand2"
            )
            self.menu_buttons.append(btn)
            self.canvas.create_window(400, y, window=btn)

        self.canvas.create_text(400, 500, text="Використовуй голос для стрибка!", font=("Arial", 12), fill="white")

    def start_level(self, speed, name):
        """Запуск гри з обраною швидкістю"""
        self.obstacle_speed = speed
        self.current_level_name = name
        self.clear_menu_buttons()
        self.setup_game()

    def setup_game(self):
        self.canvas.delete("all")
        self.game_active = True
        self.score = 0
        
        # Ігровий світ
        self.canvas.create_rectangle(0, 0, 800, 550, fill="#0055ff") # Небо
        self.canvas.create_rectangle(0, 340, 800, 550, fill="#002266", outline="") # Земля
        
        # Тексти
        self.score_label = self.canvas.create_text(400, 50, text="SCORE: 0", font=("Courier", 35, "bold"), fill="white")
        self.canvas.create_text(400, 90, text=f"MODE: {self.current_level_name}", font=("Arial", 12, "bold"), fill="#00ff88")

        # Об'єкти
        self.player_id = self.canvas.create_rectangle(100, 300, 140, 340, fill="#00ff88", outline="white", width=2)
        self.spike_id = self.canvas.create_polygon(800, 340, 820, 300, 840, 340, fill="black", outline="white", width=2)

        self.y_velocity = 0
        self.gravity = 1.4
        self.is_jumping = False
        self.update()

    def audio_listener(self):
        while self.running:
            try:
                data = self.stream.read(CHUNK, exception_on_overflow=False)
                peak = np.max(np.abs(np.frombuffer(data, dtype=np.int16)))
                if self.game_active and peak > THRESHOLD and not self.is_jumping:
                    self.y_velocity = -18
                    self.is_jumping = True
            except: pass

    def update(self):
        if not self.game_active: return

        # Фізика гравця
        self.canvas.move(self.player_id, 0, self.y_velocity)
        p_pos = self.canvas.coords(self.player_id)

        if p_pos[3] < 340:
            self.y_velocity += self.gravity
        else:
            self.y_velocity = 0
            self.is_jumping = False
            self.canvas.coords(self.player_id, 100, 300, 140, 340)

        # Рух шипа
        self.canvas.move(self.spike_id, -self.obstacle_speed, 0)
        s_pos = self.canvas.coords(self.spike_id)

        if s_pos[0] < -50:
            self.canvas.coords(self.spike_id, 800, 340, 820, 300, 840, 340)
            self.score += 1
            self.canvas.itemconfig(self.score_label, text=f"SCORE: {self.score}")

        # Зіткнення
        if self.check_collision(p_pos, s_pos):
            self.game_over()
            return

        self.root.after(20, self.update)

    def check_collision(self, p, s):
        pad = 8
        return p[2]-pad > s[0] and p[0]+pad < s[4] and p[3]-pad > s[3]

    def game_over(self):
        self.game_active = False
        self.canvas.create_rectangle(0, 0, 800, 550, fill="black", stipple="gray50")
        self.canvas.create_text(400, 180, text="КРАШ!", font=("Courier", 60, "bold"), fill="#ff4444")
        
        # Кнопки після програшу
        btn_restart = tk.Button(self.root, text="СПРОБУВАТИ ЗНОВУ", command=self.setup_game, font=("Arial", 14, "bold"), bg="white", width=20)
        btn_menu = tk.Button(self.root, text="В МЕНЮ", command=self.show_main_menu, font=("Arial", 14), bg="#cccccc", width=20)
        
        self.menu_buttons.extend([btn_restart, btn_menu])
        self.canvas.create_window(400, 280, window=btn_restart)
        self.canvas.create_window(400, 340, window=btn_menu)

if __name__ == "__main__":
    root = tk.Tk()
    game = GeometryDashLevels(root)
    root.mainloop()
