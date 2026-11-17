import pygame, random, sys

pygame.init()
screen = pygame.display.set_mode((0, 0))
W, H = screen.get_size()
clock = pygame.time.Clock()

BG = (18, 22, 38); CARD = (38, 44, 70); WHITE = (240, 240, 240)
GRAY = (100, 105, 125); GREEN = (60, 190, 90); RED = (210, 60, 60)
BLUE = (70, 120, 220); YEL = (255, 215, 60)
big = pygame.font.Font(None, int(W * 0.07))
mid = pygame.font.Font(None, int(W * 0.055))
small = pygame.font.Font(None, int(W * 0.045))

OUT = [4, 5, 13, 14, 16, 17, 18, 19, 21, 22, 23, 25, 26, 27, 32, 33]
INONLY = [34, 35, 36, 39]
FLASH = [6, 7, 8, 9, 10, 11]
ADC1 = [32, 33, 34, 35, 36, 39]
ADC2 = [4, 12, 13, 14, 15, 25, 26, 27]
TOUCH = [4, 13, 14, 27, 32, 33]

# (topic, sawal, sahi pins, galat pins, samjhao)
QS = [
 ("Output", "LED jalani hai. Kaunsa pin chalega?", OUT, INONLY + FLASH,
  "GPIO 34-39 sirf INPUT hote hain aur 6-11 flash chip ke hain. LED ko output pin chahiye."),
 ("Output", "Servo motor ko PWM signal dena hai. Kaunsa pin?", OUT, INONLY + FLASH,
  "PWM sirf output-capable pin se nikalta hai. 34-39 output nahi kar sakte."),
 ("Input", "Kaunsa pin SIRF input hai (output nahi kar sakta)?", INONLY, OUT,
  "GPIO 34, 35, 36, 39 sirf input hain. Inme pull-up resistor bhi nahi hota."),
 ("Analog", "Potentiometer padhna hai aur WiFi ON hai. Kaunsa ADC1 pin?", ADC1, ADC2,
  "WiFi ON ho to ADC2 kaam nahi karta. ADC1 pins: 32, 33, 34, 35, 36, 39."),
 ("I2C", "OLED screen ka default SDA pin kaunsa hai?", [21], [22, 23, 18, 19, 5, 17, 16, 4],
  "ESP32 ka default I2C: SDA = GPIO 21."),
 ("I2C", "OLED screen ka default SCL pin kaunsa hai?", [22], [21, 23, 18, 19, 5, 17, 16, 4],
  "ESP32 ka default I2C: SCL = GPIO 22."),
 ("Onboard", "Zyada tar DevKit mein built-in blue LED kis pin pe hoti hai?", [2],
  [4, 5, 12, 13, 15, 18, 19, 21],
  "Built-in LED aam taur par GPIO 2 pe hoti hai."),
 ("DAC", "Asli analog voltage (DAC) nikalna hai. Kaunsa pin?", [25, 26],
  [32, 33, 27, 14, 21, 22, 23, 18],
  "ESP32 ke 2 DAC pins hain: GPIO 25 aur 26."),
 ("Touch", "Touch sensor (capacitive) ke liye kaunsa pin?", TOUCH,
  [5, 16, 17, 18, 19, 21, 22, 23, 25, 26, 34, 35, 36, 39],
  "Touch pins: 4, 13, 14, 27, 32, 33 (aur kuch aur)."),
 ("UART", "UART2 ka TX pin kaunsa hai?", [17], [16, 21, 22, 23, 18, 19, 5],
  "UART2: TX = GPIO 17, RX = GPIO 16."),
 ("UART", "UART2 ka RX pin kaunsa hai?", [16], [17, 21, 22, 23, 18, 19, 5],
  "UART2: RX = GPIO 16, TX = GPIO 17."),
 ("SPI", "SPI (VSPI) ka SCK (clock) pin kaunsa hai?", [18], [19, 23, 5, 21, 22, 17],
  "VSPI: SCK = 18, MISO = 19, MOSI = 23, CS = 5."),
 ("SPI", "SPI (VSPI) ka MOSI pin kaunsa hai?", [23], [18, 19, 5, 21, 22, 17],
  "VSPI: MOSI = 23, MISO = 19, SCK = 18, CS = 5."),
 ("Danger", "Kaunse pins ko KABHI use nahi karna chahiye (flash chip)?", FLASH, OUT,
  "GPIO 6 se 11 flash memory se jude hain. Inhe use karne se board hang ho jata hai."),
]
TOPICS = sorted(set(x[0] for x in QS))

mist = {t: 0 for t in TOPICS}
total_wrong = {t: 0 for t in TOPICS}
TOTAL_ROUNDS = 10
state = {}

def reset_game():
    state.update(round=0, score=0, bot=0, hints=3, over=False, last=None)
    for t in TOPICS:
        mist[t] = 0
        total_wrong[t] = 0
    new_q()

def new_q():
    st = state
    st["round"] += 1
    weights = [1 + 3 * mist[x[0]] for x in QS]
    while True:
        q = random.choices(QS, weights)[0]
        if q is not st["last"]:
            break
    st["last"] = q
    st["q"] = q
    st["correct"] = random.choice(q[2])
    pool = [p for p in set(q[3]) if p not in q[2]]
    opts = random.sample(pool, 3) + [st["correct"]]
    random.shuffle(opts)
    st["opts"] = opts
    st["answered"] = False
    st["chosen"] = None
    st["elim"] = set()
    st["bot_pick"] = None

def bot_answer():
    done = max(1, state["round"])
    acc = state["score"] / done
    p = min(0.9, 0.4 + 0.5 * acc)   # AI bot player ki skill ke hisab se badalta hai
    if random.random() < p:
        state["bot_pick"] = state["correct"]
    else:
        wrong = [o for o in state["opts"] if o != state["correct"]]
        state["bot_pick"] = random.choice(wrong)
    if state["bot_pick"] == state["correct"]:
        state["bot"] += 1

def ai_hint():
    if state["hints"] <= 0 or state["answered"]:
        return
    wrong = [o for o in state["opts"] if o != state["correct"] and o not in state["elim"]]
    random.shuffle(wrong)
    for o in wrong[:2]:
        state["elim"].add(o)
    state["hints"] -= 1

def wrap(text, font, maxw):
    words = text.split(); lines = []; cur = ""
    for w in words:
        t = (cur + " " + w).strip()
        if font.size(t)[0] <= maxw:
            cur = t
        else:
            lines.append(cur); cur = w
    if cur:
        lines.append(cur)
    return lines

def center(txt, font, col, y):
    s = font.render(txt, True, col)
    screen.blit(s, (W // 2 - s.get_width() // 2, y))

def opt_rects():
    bw, bh = int(W * 0.42), int(H * 0.10)
    xs = [int(W * 0.06), int(W * 0.52)]
    ys = [int(H * 0.35), int(H * 0.47)]
    return [pygame.Rect(xs[i % 2], ys[i // 2], bw, bh) for i in range(4)]

def btn_rects():
    bw, bh = int(W * 0.40), int(H * 0.08)
    return (pygame.Rect(int(W * 0.06), int(H * 0.88), bw, bh),
            pygame.Rect(int(W * 0.54), int(H * 0.88), bw, bh))

def draw():
    screen.fill(BG)
    if state["over"]:
        center("Game Over!", big, YEL, int(H * 0.08))
        center("Tum: " + str(state["score"]) + "/" + str(TOTAL_ROUNDS), mid, WHITE, int(H * 0.2))
        center("ESP-Bot: " + str(state["bot"]) + "/" + str(TOTAL_ROUNDS), mid, WHITE, int(H * 0.27))
        if state["score"] > state["bot"]:
            res = "Tum jeete! Shabash!"
        elif state["score"] < state["bot"]:
            res = "Bot jeeta. Dobara try karo!"
        else:
            res = "Barabari!"
        center(res, mid, GREEN, int(H * 0.36))
        worst = max(TOPICS, key=lambda t: total_wrong[t])
        y = int(H * 0.48)
        center("AI Coach ki salah:", mid, YEL, y)
        if total_wrong[worst] == 0:
            msg = "Koi galti nahi! Tum ESP32 expert ho!"
        else:
            msg = "Topic '" + worst + "' ki practice karo (" + str(total_wrong[worst]) + " galti)"
        for i, ln in enumerate(wrap(msg, small, int(W * 0.85))):
            center(ln, small, WHITE, y + int(H * 0.07) + i * int(H * 0.045))
        center("Naya game: screen par tap karo", small, GRAY, int(H * 0.88))
        pygame.display.flip()
        return
    st = state
    center("ESP32 Pin Hunter", big, YEL, int(H * 0.02))
    center("Round " + str(st["round"]) + "/" + str(TOTAL_ROUNDS) + "   Tum: " + str(st["score"]) +
           "   Bot: " + str(st["bot"]), small, GRAY, int(H * 0.09))
    card = pygame.Rect(int(W * 0.05), int(H * 0.14), int(W * 0.9), int(H * 0.19))
    pygame.draw.rect(screen, CARD, card, border_radius=14)
    center("Topic: " + st["q"][0], small, BLUE, card.y + 10)
    for i, ln in enumerate(wrap(st["q"][1], mid, int(card.w * 0.92))):
        center(ln, mid, WHITE, card.y + int(H * 0.05) + i * int(H * 0.04))
    for o, r in zip(st["opts"], opt_rects()):
        col = BLUE
        if o in st["elim"]:
            col = (50, 52, 65)
        if st["answered"]:
            if o == st["correct"]:
                col = GREEN
            elif o == st["chosen"]:
                col = RED
        pygame.draw.rect(screen, col, r, border_radius=14)
        t = mid.render("GPIO " + str(o), True, WHITE)
        screen.blit(t, (r.centerx - t.get_width() // 2, r.centery - t.get_height() // 2))
    y = int(H * 0.60)
    if st["answered"]:
        right = st["chosen"] == st["correct"]
        center("Sahi jawab!" if right else "Galat! Sahi: GPIO " + str(st["correct"]),
               mid, GREEN if right else RED, y)
        for i, ln in enumerate(wrap(st["q"][4], small, int(W * 0.9))):
            center(ln, small, WHITE, y + int(H * 0.055) + i * int(H * 0.04))
        bp = st["bot_pick"]
        center("ESP-Bot ne GPIO " + str(bp) + " chuna: " + ("sahi" if bp == st["correct"] else "galat"),
               small, YEL, int(H * 0.82))
    else:
        center("Sahi pin par tap karo", small, GRAY, y)
    hb, nb = btn_rects()
    pygame.draw.rect(screen, BLUE if st["hints"] > 0 and not st["answered"] else GRAY, hb, border_radius=12)
    t = small.render("AI Hint (" + str(st["hints"]) + ")", True, WHITE)
    screen.blit(t, (hb.centerx - t.get_width() // 2, hb.centery - t.get_height() // 2))
    pygame.draw.rect(screen, GREEN if st["answered"] else GRAY, nb, border_radius=12)
    t = small.render("Aage", True, WHITE)
    screen.blit(t, (nb.centerx - t.get_width() // 2, nb.centery - t.get_height() // 2))
    pygame.display.flip()

def handle_tap(pos):
    if state["over"]:
        reset_game()
        return
    hb, nb = btn_rects()
    if hb.collidepoint(pos):
        ai_hint()
        return
    if nb.collidepoint(pos) and state["answered"]:
        if state["round"] >= TOTAL_ROUNDS:
            state["over"] = True
        else:
            new_q()
        return
    if not state["answered"]:
        for o, r in zip(state["opts"], opt_rects()):
            if r.collidepoint(pos) and o not in state["elim"]:
                state["chosen"] = o
                state["answered"] = True
                topic = state["q"][0]
                if o == state["correct"]:
                    state["score"] += 1
                    mist[topic] = max(0, mist[topic] - 1)
                else:
                    mist[topic] += 1          # AI Coach yaad rakhta hai
                    total_wrong[topic] += 1
                bot_answer()
                return

reset_game()
running = True
while running:
    draw()
    for e in pygame.event.get():
        if e.type == pygame.QUIT:
            running = False
        elif e.type == pygame.MOUSEBUTTONDOWN:
            handle_tap(e.pos)
    clock.tick(30)
pygame.quit()
sys.exit()
