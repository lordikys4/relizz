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
THRESHOLD = 3000 # Поріг чутливості (чим вище число, тим голосніше треба кричати)

class GeometryDashComplete:
    def __init__(self, root):
        self.root = root
        self.root.title("GD: All Mechanics Edition")
        # Створення полотна для малювання гри
        self.canvas = tk.Canvas(root, width=800, height=450, bg="#000022", highlightthickness=0)
        self.canvas.pack()

        # Ініціалізація системи аудіо
        self.p = pyaudio.PyAudio()
        self.stream = None
        try:
            self.stream = self.p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK)
        except: 
            print("Мікрофон не знайдено")
        
        # Прив'язка клавіші пробіл для стрибка (резервний варіант)
        self.root.bind("<space>", lambda e: self.jump())
        
        # Початкові змінні стану гри
        self.hue = 0.6
        self.game_active = False
        self.gravity_dir = 1 # 1 - вниз, -1 - вгору (для порталів)
        self.score = 0
        self.high_score = 0
        self.particles = []
        self.player_trail = []
        self.current_vol = 0
        self.show_menu()

    def show_menu(self):
        # Очищення екрана та показ головного меню
        self.canvas.delete("all")
        self.canvas.create_text(400, 150, text="MEGA DASH", font=("Courier", 50, "bold"), fill="#ffff00")
        self.canvas.create_text(400, 220, text=f"BEST: {self.high_score}", font=("Courier", 20), fill="white")
        btn = tk.Button(self.root, text="START", command=self.start_game, font=("Arial", 20), bg="#ffff00", fg="black")
        self.canvas.create_window(400, 320, window=btn)

    def start_game(self):
        # Скидання параметрів перед початком нового раунду
        self.canvas.delete("all")
        self.game_active = True
        self.speed = 13
        self.spawn_timer = 0
        self.score = 0
        self.gravity_dir = 1
        # Створення об'єкта гравця (куб)
        self.player_id = self.canvas.create_rectangle(100, 304, 140, 344, fill="#00ff88", outline="white", width=2)
        # Створення візуального індикатора гучності зліва
        self.vol_bar = self.canvas.create_rectangle(10, 440, 25, 440, fill="#ffff00", outline="white")
        # Текст поточного рахунку
        self.score_text = self.canvas.create_text(700, 30, text="0", font=("Courier", 30), fill="white")
        
        self.game_objects = []
        self.particles = []
        self.player_trail = []
        self.y_velocity = 0
        self.gravity = 1.7
        self.is_jumping = False
        
        # Запуск фонового потоку для прослуховування мікрофона
        if self.stream: 
            threading.Thread(target=self.audio_listener, daemon=True).start()
        self.update()

    def audio_listener(self):
        # Функція, що постійно перевіряє рівень шуму в мікрофоні
        while self.game_active and self.stream:
            try:
                data = self.stream.read(CHUNK, exception_on_overflow=False)
                val = np.max(np.abs(np.frombuffer(data, dtype=np.int16)))
                self.current_vol = val
                # Якщо звук гучніший за THRESHOLD — викликаємо стрибок
                if val > THRESHOLD: 
                    self.root.after(0, self.jump)
            except: 
                break

    def jump(self):
        # Логіка стрибка та взаємодії з блакитними сферами (orb)
        can_orb = False
        p = self.canvas.coords(self.player_id)
        for obj in self.game_objects:
            if "orb" in self.canvas.gettags(obj):
                o = self.canvas.coords(obj)
                # Перевірка, чи гравець достатньо близько до сфери для стрибка в повітрі
                if abs(p[0]-o[0]) < 65 and abs(p[1]-o[1]) < 65:
                    can_orb = True
                    self.create_burst(o[0]+15, o[1]+15, "#00ffff", 10)
                    break
        
        # Стрибок дозволений, якщо гравець на підлозі або торкається сфери
        if not self.is_jumping or can_orb:
            self.y_velocity = -23 * self.gravity_dir
            self.is_jumping = True
            self.create_burst(p[0]+20, p[3] if self.gravity_dir==1 else p[1], "white", 8)

    def create_burst(self, x, y, color, count):
        # Створення ефекту часток (іскор)
        for _ in range(count):
            p = self.canvas.create_oval(x, y, x+5, y+5, fill=color, outline="")
            vx, vy = random.uniform(-7, 7), random.uniform(-12, 2)
            self.particles.append([p, vx, vy, random.randint(10, 20)])

    def spawn_logic(self):
        # Випадкова генерація перешкод на шляху
        self.spawn_timer += 1
        if self.spawn_timer > random.randint(22, 45):
            x = 850
            r = random.random()
            if r < 0.35: # Генерація Шипа
                o = self.canvas.create_polygon(x, 344, x+25, 290, x+50, 344, fill="#ffaa00", outline="white", tags="spike")
            elif r < 0.55: # Генерація Блоку
                o = self.canvas.create_rectangle(x, 230, x+120, 260, fill="#00ffff", outline="white", tags="block")
            elif r < 0.7: # Генерація Порталу зміни гравітації
                o = self.canvas.create_rectangle(x, 100, x+30, 300, fill="white", stipple="gray50", outline="#ff00ff", tags="portal")
            elif r < 0.85: # Генерація Батута
                o = self.canvas.create_rectangle(x, 335, x+45, 344, fill="#ffff00", outline="orange", tags="trampoline")
            else: # Генерація Сфери для подвійного стрибка
                o = self.canvas.create_oval(x, 180, x+35, 215, outline="#00ffff", width=4, tags="orb")
            
            self.game_objects.append(o)
            self.spawn_timer = 0

    def update_effects(self):
        # Оновлення кольору фону (градієнт)
        self.hue += 0.002
        rgb = colorsys.hsv_to_rgb(self.hue % 1, 0.7, 0.2)
        self.canvas.config(bg='#%02x%02x%02x' % (int(rgb[0]*255), int(rgb[1]*255), int(rgb[2]*255)))
        
        # Оновлення індикатора мікрофона
        vol_h = min(400, (self.current_vol / 15000) * 400)
        self.canvas.coords(self.vol_bar, 10, 440 - vol_h, 25, 440)

        # Створення шлейфу (Trail) за гравцем
        p_pos = self.canvas.coords(self.player_id)
        t = self.canvas.create_rectangle(p_pos[0], p_pos[1], p_pos[2], p_pos[3], fill="#00ff88", outline="")
        self.player_trail.append([t, 1.0])
        for trail in self.player_trail[:]:
            trail[1] -= 0.15 # Зменшення прозорості шлейфу
            if trail[1] <= 0:
                self.canvas.delete(trail[0]); self.player_trail.remove(trail)
            else: 
                self.canvas.itemconfig(trail[0], stipple="gray50")

        # Рух і видалення часток
        for p_data in self.particles[:]:
            self.canvas.move(p_data[0], p_data[1], p_data[2])
            p_data[2] += 0.7; p_data[3] -= 1
            if p_data[3] <= 0:
                self.canvas.delete(p_data[0]); self.particles.remove(p_data)

    def update(self):
        if not self.game_active: return
        self.update_effects()
        self.spawn_logic()
        
        # Фізика падіння та стрибка
        self.y_velocity += self.gravity * self.gravity_dir
        self.canvas.move(self.player_id, 0, self.y_velocity)
        
        p = self.canvas.coords(self.player_id)
        # Обробка приземлення на підлогу або стелю
        if self.gravity_dir == 1 and p[3] >= 344:
            self.canvas.move(self.player_id, 0, 344 - p[3])
            self.y_velocity, self.is_jumping = 0, False
            p = self.canvas.coords(self.player_id)
        elif self.gravity_dir == -1 and p[1] <= 50:
            self.canvas.move(self.player_id, 0, 50 - p[1])
            self.y_velocity, self.is_jumping = 0, False
            p = self.canvas.coords(self.player_id)

        # --- Робота з ХІТБОКСАМИ ---
        # Створення невидимого хитбокса гравця (менший за візуальний куб на 8 пікселів)
        p_hitbox = [p[0]+8, p[1]+8, p[2]-8, p[3]-8]

        for obj in self.game_objects[:]:
            self.canvas.move(obj, -self.speed, 0) # Рух перешкод вліво
            o = self.canvas.coords(obj)
            tags = self.canvas.gettags(obj)
            
            # Нарахування очок, коли об'єкт виходить за екран
            if o[0] < -150:
                self.score += 1
                self.canvas.itemconfig(self.score_text, text=str(self.score))
                self.canvas.delete(obj); self.game_objects.remove(obj)
                continue

            # Отримання точного (зменшеного) хитбокса об'єкта
            o_hitbox = self.get_accurate_hitbox(o, tags[0])

            # Перевірка на зіткнення хитбоксів
            if self.check_collision(p_hitbox, o_hitbox):
                if "spike" in tags: 
                    self.die(); return
                elif "portal" in tags:
                    self.gravity_dir *= -1 # Зміна гравітації
                    self.create_burst(o[0], 200, "white", 15)
                    self.canvas.delete(obj); self.game_objects.remove(obj)
                elif "trampoline" in tags:
                    self.y_velocity = -40 * self.gravity_dir # Потужний стрибок
                    self.is_jumping = True
                elif "block" in tags:
                    # Перевірка приземлення на верхню частину блоку
                    if (self.gravity_dir == 1 and self.y_velocity > 0 and p[3] <= o[1]+20) or \
                       (self.gravity_dir == -1 and self.y_velocity < 0 and p[1] >= o[3]-20):
                        self.y_velocity, self.is_jumping = 0, False
                    else: 
                        self.die(); return
        self.root.after(20, self.update) # Рекурсивний виклик оновлення кожні 20мс

    def get_accurate_hitbox(self, o, tag):
        # Функція для розрахунку "чесного" хитбокса
        ox1, oy1, ox2, oy2 = (min(o[0::2]), min(o[1::2]), max(o[0::2]), max(o[1::2]))
        
        if tag == "spike":
            # Шип має дуже малий хитбокс по центру, щоб не вбивати за кінчик
            return [ox1 + 15, oy1 + 15, ox2 - 15, oy2]
        elif tag == "block":
            # Блок зменшено на 10 пікселів для комфортної гри
            return [ox1 + 10, oy1 + 10, ox2 - 10, oy2 - 10]
        elif tag == "trampoline":
            # Батут реагує майже по всій ширині
            return [ox1 + 5, oy1, ox2 - 5, oy2]
        else:
            return [ox1, oy1, ox2, oy2]

    def check_collision(self, p, o):
        # Стандартна перевірка перетину двох прямокутників (AABB)
        return not (p[2] < o[0] or p[0] > o[2] or p[3] < o[1] or p[1] > o[3])

    def die(self):
        # Зупинка гри та анімація вибуху
        self.game_active = False
        if self.score > self.high_score: 
            self.high_score = self.score
        self.create_burst(120, 320, "red", 40)
        self.canvas.create_text(400, 200, text="BOOM!", font=("Courier", 60, "bold"), fill="red")
        self.root.after(1500, self.show_menu)

if __name__ == "__main__":
    root = tk.Tk()
    app = GeometryDashComplete(root)
    root.mainloop()
