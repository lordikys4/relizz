import tkinter as tk
import pyaudio
import numpy as np
import threading
import random
import colorsys

# --- Налаштування аудіо та гри ---
CHUNK = 1024
FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 44100
THRESHOLD = 3000 # Поріг для стрибка

class GeometryDashFixed:
    def __init__(self, root):
        self.root = root
        self.root.title("GD: Fixed Mechanics")
        self.canvas = tk.Canvas(root, width=800, height=450, bg="#000022", highlightthickness=0)
        self.canvas.pack()

        self.p = pyaudio.PyAudio()
        self.stream = None
        try:
            self.stream = self.p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK)
        except: 
            print("Мікрофон не знайдено")
        
        self.root.bind("<space>", lambda e: self.jump())
        
        self.hue = 0.6
        self.game_active = False
        self.score = 0
        self.high_score = 0
        self.particles = []
        self.player_trail = []
        self.current_vol = 0
        self.show_menu()

    def show_menu(self):
        self.canvas.delete("all")
        self.canvas.create_text(400, 150, text="MEGA DASH", font=("Courier", 50, "bold"), fill="#00ff88")
        self.canvas.create_text(400, 220, text=f"BEST: {self.high_score}", font=("Courier", 20), fill="white")
        btn = tk.Button(self.root, text="START", command=self.start_game, font=("Arial", 20), bg="#00ff88", fg="black")
        self.canvas.create_window(400, 320, window=btn)

    def start_game(self):
        self.canvas.delete("all")
        self.game_active = True
        self.speed = 13
        self.spawn_timer = 0
        self.score = 0
        
        # Гравець та інтерфейс
        self.player_id = self.canvas.create_rectangle(100, 304, 140, 344, fill="#00ff88", outline="white", width=2)
        self.vol_bar = self.canvas.create_rectangle(10, 440, 25, 440, fill="#ffff00", outline="white")
        self.score_text = self.canvas.create_text(700, 30, text="0", font=("Courier", 30), fill="white")
        
        self.game_objects = []
        self.particles = []
        self.player_trail = []
        self.y_velocity = 0
        self.gravity = 1.7
        self.is_jumping = False
        
        if self.stream: 
            threading.Thread(target=self.audio_listener, daemon=True).start()
        self.update()

    def audio_listener(self):
        while self.game_active and self.stream:
            try:
                data = self.stream.read(CHUNK, exception_on_overflow=False)
                val = np.max(np.abs(np.frombuffer(data, dtype=np.int16)))
                self.current_vol = val
                if val > THRESHOLD: 
                    self.root.after(0, self.jump)
            except: 
                break

    def jump(self):
        can_orb = False
        p = self.canvas.coords(self.player_id)
        if not p: return
        for obj in self.game_objects:
            if "orb" in self.canvas.gettags(obj):
                o = self.canvas.coords(obj)
                if abs(p[0]-o[0]) < 70 and abs(p[1]-o[1]) < 70:
                    can_orb = True
                    self.create_burst(o[0]+15, o[1]+15, "#00ffff", 10)
                    break
        
        if not self.is_jumping or can_orb:
            self.y_velocity = -23
            self.is_jumping = True
            self.create_burst(p[0]+20, p[3], "white", 8)

    def create_burst(self, x, y, color, count):
        for _ in range(count):
            p = self.canvas.create_oval(x, y, x+5, y+5, fill=color, outline="")
            vx, vy = random.uniform(-7, 7), random.uniform(-12, 2)
            self.particles.append([p, vx, vy, random.randint(10, 20)])

    def spawn_logic(self):
        self.spawn_timer += 1
        if self.spawn_timer > random.randint(25, 45):
            x = 850
            r = random.random()
            if r < 0.4: # Шип
                o = self.canvas.create_polygon(x, 344, x+25, 290, x+50, 344, fill="#ffaa00", outline="white", tags="spike")
            elif r < 0.7: # Блок
                o = self.canvas.create_rectangle(x, 230, x+120, 260, fill="#00ffff", outline="white", tags="block")
            elif r < 0.85: # Батут
                o = self.canvas.create_rectangle(x, 335, x+45, 344, fill="#ffff00", outline="orange", tags="trampoline")
            else: # Сфера
                o = self.canvas.create_oval(x, 180, x+35, 215, outline="#00ffff", width=4, tags="orb")
            
            self.game_objects.append(o)
            self.spawn_timer = 0

    def update(self):
        if not self.game_active: return
        
        # Оновлення шкали звуку
        vol_h = min(400, (self.current_vol / 15000) * 400)
        self.canvas.coords(self.vol_bar, 10, 440 - vol_h, 25, 440)
        
        # Ефекти фону
        self.hue += 0.002
        rgb = colorsys.hsv_to_rgb(self.hue % 1, 0.7, 0.2)
        self.canvas.config(bg='#%02x%02x%02x' % (int(rgb[0]*255), int(rgb[1]*255), int(rgb[2]*255)))
        
        # Генерація перешкод
        self.spawn_logic()
        
        # Фізика гравця
        self.y_velocity += self.gravity
        self.canvas.move(self.player_id, 0, self.y_velocity)
        
        p = self.canvas.coords(self.player_id)
        if p[3] >= 344:
            self.canvas.move(self.player_id, 0, 344 - p[3])
            self.y_velocity, self.is_jumping = 0, False
            p = self.canvas.coords(self.player_id)

        # Хітбокс гравця
        p_hb = [p[0]+7, p[1]+7, p[2]-7, p[3]-7]

        for obj in self.game_objects[:]:
            self.canvas.move(obj, -self.speed, 0)
            o = self.canvas.coords(obj)
            tags = self.canvas.gettags(obj)
            
            if o[0] < -150:
                self.score += 1
                self.canvas.itemconfig(self.score_text, text=str(self.score))
                self.canvas.delete(obj); self.game_objects.remove(obj)
                continue

            ox1, oy1, ox2, oy2 = (min(o[0::2]), min(o[1::2]), max(o[0::2]), max(o[1::2]))
            
            # Точні хітбокси перешкод
            if "spike" in tags: o_hit = [ox1 + 12, oy1 + 10, ox2 - 12, oy2]
            elif "block" in tags: o_hit = [ox1 + 5, oy1 + 5, ox2 - 5, oy2 - 5]
            else: o_hit = [ox1, oy1, ox2, oy2]

            # Перевірка зіткнення
            if not (p_hb[2] < o_hit[0] or p_hb[0] > o_hit[2] or p_hb[3] < o_hit[1] or p_hb[1] > o_hit[3]):
                if "spike" in tags: self.die(); return
                elif "trampoline" in tags:
                    self.y_velocity = -35; self.is_jumping = True
                elif "block" in tags:
                    if self.y_velocity > 0 and p[3] <= oy1 + 25:
                        self.canvas.move(self.player_id, 0, oy1 - p[3])
                        self.y_velocity, self.is_jumping = 0, False
                    else: self.die(); return

        # Частки
        for p_data in self.particles[:]:
            self.canvas.move(p_data[0], p_data[1], p_data[2])
            p_data[2] += 0.7; p_data[3] -= 1
            if p_data[3] <= 0:
                self.canvas.delete(p_data[0]); self.particles.remove(p_data)

        self.root.after(20, self.update)

    def die(self):
        self.game_active = False
        if self.score > self.high_score: self.high_score = self.score
        self.create_burst(120, 320, "red", 40)
        self.canvas.create_text(400, 200, text="BOOM!", font=("Courier", 60, "bold"), fill="red")
        self.root.after(1500, self.show_menu)

if __name__ == "__main__":
    root = tk.Tk()
    app = GeometryDashFixed(root)
    root.mainloop()
