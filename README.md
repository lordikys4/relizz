import tkinter as tk
import pyaudio
import numpy as np
import threading
import random
import colorsys

# --- НАЛАШТУВАННЯ ---
CHUNK = 1024
FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 44100
THRESHOLD = 2000

class GeometryDashEffects:
    def __init__(self, root):
        self.root = root
        self.root.title("GD: Neon Effects Edition")
        self.canvas = tk.Canvas(root, width=800, height=450, bg="#000022", highlightthickness=0)
        self.canvas.pack()

        self.p = pyaudio.PyAudio()
        self.stream = self.p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK)
        
        self.root.bind("<space>", lambda e: self.jump())
        
        self.hue = 0.6  # Початковий колір (синій)
        self.game_active = False
        self.show_menu()

    def show_menu(self):
        self.canvas.delete("all")
        self.canvas.create_text(400, 200, text="NEON DASH", font=("Courier", 60, "bold"), fill="#00ff88")
        btn = tk.Button(self.root, text="START INFINITE", command=self.start_game, font=("Arial", 20), bg="#00ff88")
        self.canvas.create_window(400, 300, window=btn)

    def start_game(self):
        self.canvas.delete("all")
        self.game_active = True
        self.score = 0
        self.speed = 10
        self.spawn_timer = 0
        
        # Гравець
        self.player_id = self.canvas.create_rectangle(100, 304, 136, 340, fill="#00ff88", outline="white", width=2)
        
        # Об'єкти та ефекти
        self.game_objects = []
        self.particles = []
        self.y_velocity = 0
        self.gravity = 1.5
        self.is_jumping = False

        threading.Thread(target=self.audio_listener, daemon=True).start()
        self.update()

    def audio_listener(self):
        while self.game_active:
            try:
                data = self.stream.read(CHUNK, exception_on_overflow=False)
                if np.max(np.abs(np.frombuffer(data, dtype=np.int16))) > THRESHOLD:
                    self.jump()
            except: pass

    def jump(self):
        if not self.is_jumping:
            self.y_velocity = -19
            self.is_jumping = True
            self.create_particles(118, 340, "#00ff88") # Іскри при відштовхуванні

    def create_particles(self, x, y, color):
        """Ефект часток (іскри)"""
        for _ in range(5):
            p = self.canvas.create_oval(x, y, x+4, y+4, fill=color, outline="")
            vx = random.uniform(-5, 2)
            vy = random.uniform(-10, -2)
            self.particles.append([p, vx, vy, 10]) # [id, speed_x, speed_y, life]

    def update_effects(self):
        """Плавна зміна фону та часток"""
        # 1. Зміна кольору неба
        self.hue += 0.001
        if self.hue > 1: self.hue = 0
        rgb = colorsys.hsv_to_rgb(self.hue, 0.7, 0.3)
        color = '#%02x%02x%02x' % (int(rgb[0]*255), int(rgb[1]*255), int(rgb[2]*255))
        self.canvas.config(bg=color)

        # 2. Оновлення часток
        for p_data in self.particles[:]:
            p_id, vx, vy, life = p_data
            self.canvas.move(p_id, vx, vy)
            p_data[2] += 0.5 # гравітація для часток
            p_data[3] -= 1   # життя часток
            if p_data[3] <= 0:
                self.canvas.delete(p_id)
                self.particles.remove(p_data)

    def spawn_logic(self):
        """Динамічна генерація перешкод"""
        self.spawn_timer += 1
        if self.spawn_timer > random.randint(30, 70):
            x = 850
            choice = random.random()
            if choice < 0.4: # Шип
                o = self.canvas.create_polygon(x, 340, x+20, 300, x+40, 340, fill="black", outline="#ff0044", width=2, tags="spike")
            elif choice < 0.7: # Подвійний шип
                o = self.canvas.create_polygon(x, 340, x+20, 300, x+40, 340, x+40, 340, x+60, 300, x+80, 340, fill="black", outline="#ff0044", tags="spike")
            else: # Платформа
                o = self.canvas.create_rectangle(x, 240, x+80, 270, fill="#333", outline="white", width=2, tags="block")
            
            self.game_objects.append(o)
            self.spawn_timer = 0

    def update(self):
        if not self.game_active: return

        self.update_effects()
        self.spawn_logic()

        # Фізика гравця
        self.y_velocity += self.gravity
        self.canvas.move(self.player_id, 0, self.y_velocity)
        p = self.canvas.coords(self.player_id)

        if p[3] >= 340:
            if self.is_jumping: # Тільки-но приземлився
                self.create_particles(p[0], 340, "white")
            self.canvas.move(self.player_id, 0, 340 - p[3])
            self.y_velocity = 0
            self.is_jumping = False

        # Рух та колізії об'єктів
        for obj in self.game_objects[:]:
            self.canvas.move(obj, -self.speed, 0)
            o_pos = self.canvas.coords(obj)
            tag = self.canvas.gettags(obj)[0]

            if o_pos[0] < -100:
                self.canvas.delete(obj)
                self.game_objects.remove(obj)
                self.score += 1
                continue

            if self.check_collision(p, o_pos):
                if tag == "spike": self.game_over()
                elif tag == "block":
                    if self.y_velocity > 0 and p[3] <= o_pos[1] + 15:
                        self.canvas.move(self.player_id, 0, o_pos[1] - p[3])
                        self.y_velocity = 0
                        self.is_jumping = False
                    else: self.game_over()

        self.root.after(20, self.update)

    def check_collision(self, p, o):
        return p[2]-5 > o[0] and p[0]+5 < (o[2] if len(o)==4 else o[4]) and \
               p[3]-5 > o[1] and p[1]+5 < (o[3] if len(o)==4 else o[5])

    def game_over(self):
        self.game_active = False
        self.canvas.create_text(400, 200, text="GAME OVER", font=("Courier", 50, "bold"), fill="#ff0044")
        self.root.after(2000, self.show_menu)

if __name__ == "__main__":
    root = tk.Tk()
    app = GeometryDashEffects(root)
    root.mainloop()
    game = GeometryDashLevels(root)
    root.mainloop()
