import pygame
import random
import math
import sys

W, H = 900, 560
FPS  = 60

BG        = (7,   7,  26)
GRID_COL  = (0,  40,  90)
CYAN      = (0, 229, 255)
GREEN     = (0, 255, 136)
BLUE      = (74, 158, 255)
YELLOW    = (255, 215,   0)
RED       = (255,  58, 110)
PURPLE    = (159,  43, 255)
ORANGE    = (255, 149,   0)
WHITE     = (255, 255, 255)
DARK      = (13,  26,  51)

STAR_T = [100, 280, 550, 900, 1350, 2000]

VIRUS_TYPES = [
    {"name": "WORM",    "body": (255,  58, 110), "eye": (255, 143, 171), "pts": 10},
    {"name": "TROJAN",  "body": (159,  43, 255), "eye": (200, 125, 255), "pts": 15},
    {"name": "MALWARE", "body": (255, 106,   0), "eye": (255, 170,  85), "pts": 20},
    {"name": "SPYWARE", "body": (0,  204,  68),  "eye": ( 85, 255, 136), "pts": 12},
    {"name": "RANSOM",  "body": (255,  34,  34), "eye": (255, 119, 119), "pts": 25},
]

def rand_binary(length=None):
    n = length or random.randint(4, 8)
    return "".join(random.choice("01") for _ in range(n))

def draw_rounded_rect(surf, color, rect, radius, width=0):
    pygame.draw.rect(surf, color, rect, width, border_radius=radius)

class Robot:
    X = 70

    def __init__(self):
        self.y      = H // 2
        self.target = H // 2

    def update(self, mouse_y):
        self.target = max(60, min(H - 90, mouse_y))
        self.y += (self.target - self.y) * 0.14

    def draw(self, surf, font_sm, t, tint=None, alpha=255, offset_y=0):
        x, y = int(self.X), int(self.y) + offset_y
        col = lambda c: tint if tint else c

        glow = pygame.Surface((90, 90), pygame.SRCALPHA)
        pygame.draw.ellipse(glow, (*col(CYAN), int(30 * alpha / 255)), (0, 0, 90, 90))
        surf.blit(glow, (x - 45, y - 10))

        for dx in (-14, 6):
            pygame.draw.rect(surf, col((26, 58, 92)),  (x + dx, y + 20, 10, 22))
            pygame.draw.rect(surf, col(CYAN),           (x + dx, y + 38, 10,  5))

        draw_rounded_rect(surf, col(DARK),        (x - 22, y - 8, 44, 30), 6)
        draw_rounded_rect(surf, col(CYAN),        (x - 22, y - 8, 44, 30), 6, 2)
        draw_rounded_rect(surf, col((13, 26, 51)),(x - 14, y - 2, 28, 16), 4)

        for i, lc in enumerate([GREEN, CYAN, RED]):
            pulse = 0.5 + 0.5 * math.sin(t * 3 + i * 1.2)
            c2 = tuple(int(v * pulse * alpha / 255) for v in col(lc))
            pygame.draw.circle(surf, c2, (x - 8 + i * 8, y + 6), 3)

        for dx in (-34, 22):
            pygame.draw.rect(surf, col((26, 58, 92)), (x + dx, y - 6, 12, 20))
        pygame.draw.circle(surf, col(CYAN), (x - 28, y - 6), 7)
        pygame.draw.circle(surf, col(CYAN), (x + 28, y - 6), 7)

        pygame.draw.rect(surf, col((17, 34, 68)), (x - 7, y - 16, 14, 10))
        draw_rounded_rect(surf, col((30, 53, 96)), (x - 20, y - 40, 40, 26), 8)
        draw_rounded_rect(surf, col(CYAN),         (x - 20, y - 40, 40, 26), 8, 2)

        for sx in (-1, 1):
            ex, ey = x + sx * 10, y - 28
            draw_rounded_rect(surf, col((0, 26, 51)), (ex - 7, ey - 6, 14, 11), 3)
            pulse = 0.6 + 0.4 * math.sin(t * 4 + sx)
            ec = tuple(int(v * pulse * alpha / 255) for v in col(CYAN))
            pygame.draw.circle(surf, ec, (ex, ey), 4)

        pygame.draw.line(surf, col(BLUE), (x, y - 40), (x, y - 56), 2)
        pulse = 0.6 + 0.4 * math.sin(t * 6)
        rc = tuple(int(v * pulse * alpha / 255) for v in col(RED))
        pygame.draw.circle(surf, rc, (x, y - 57), 5)

class Bullet:
    def __init__(self, x, y, ty):
        dx, dy = W - x, ty - y
        d = math.hypot(dx, dy) or 1
        sp = 13
        self.x, self.y = float(x), float(y)
        self.vx, self.vy = dx / d * sp, dy / d * sp
        self.label = rand_binary(random.randint(4, 7))
        self.life = 90
        self.color = random.choice([CYAN, GREEN])

    def update(self):
        self.x += self.vx; self.y += self.vy; self.life -= 1

    def alive(self):
        return self.life > 0 and self.x < W + 20 and -20 < self.y < H + 20

    def draw(self, surf, font_sm):
        angle = math.degrees(math.atan2(self.vy, self.vx))
        lw = len(self.label) * 5 + 4
        s = pygame.Surface((lw, 14), pygame.SRCALPHA)
        pygame.draw.ellipse(s, (*self.color, 210), (0, 0, lw, 14))
        txt = font_sm.render(self.label, True, BG)
        s.blit(txt, (lw//2 - txt.get_width()//2, 7 - txt.get_height()//2))
        rot = pygame.transform.rotate(s, -angle)
        surf.blit(rot, (int(self.x) - rot.get_width()//2, int(self.y) - rot.get_height()//2))

class Virus:
    def __init__(self, level):
        vt = random.choice(VIRUS_TYPES)
        self.r = random.randint(20, 32)
        self.x = float(W + self.r)
        self.y = float(random.randint(50, H - 70))
        self.vx = -(0.7 + level * 0.2 + random.random() * 0.3)
        self.vy = (random.random() - 0.5) * 0.5
        self.hp = 1 + level // 3
        self.max_hp = self.hp
        self.body = vt["body"]; self.eye = vt["eye"]
        self.name = vt["name"]; self.pts = vt["pts"]
        self.wobble = random.uniform(0, math.pi * 2)
        self.tentacles = random.randint(4, 6)

    def update(self, t):
        self.x += self.vx; self.y += self.vy; self.wobble += 0.025
        if self.y < self.r or self.y > H - self.r - 18:
            self.vy *= -1

    def alive(self):
        return self.x > -self.r * 2

    def draw(self, surf, font_sm, t):
        cx, cy = int(self.x), int(self.y)

        for i in range(self.tentacles):
            a = (i / self.tentacles) * math.pi * 2 + t * 0.5
            l = self.r * 0.9 + math.sin(t * 2 + i) * 6
            sx = cx + math.cos(a) * self.r * 0.6
            sy = cy + math.sin(a) * self.r * 0.6
            ex = cx + math.cos(a) * l + math.sin(a + t) * 4
            ey = cy + math.sin(a) * l + math.cos(a + t) * 4
            pygame.draw.line(surf, self.body, (int(sx), int(sy)), (int(ex), int(ey)), 3)

        glow_col = tuple(min(255, v + 60) for v in self.body)
        pygame.draw.circle(surf, glow_col, (cx, cy), self.r + 3)
        pygame.draw.circle(surf, self.body, (cx, cy), self.r)
        pygame.draw.circle(surf, self.eye,  (cx, cy), self.r, 2)

        for sx in (-1, 1):
            ex = cx + sx * int(self.r * 0.32)
            ey = cy - int(self.r * 0.15)
            pygame.draw.circle(surf, (0, 0, 0),   (ex, ey), int(self.r * 0.22))
            pygame.draw.circle(surf, WHITE,        (ex, ey), int(self.r * 0.12))
            pygame.draw.circle(surf, (220, 0, 0), (ex + 2, ey + 2), int(max(1, self.r * 0.07)))

        pygame.draw.arc(surf, (0, 0, 0),
                        (cx - int(self.r*0.3), cy + int(self.r*0.1), int(self.r*0.6), int(self.r*0.2)),
                        math.pi, 0, 2)

        lbl = font_sm.render(self.name, True, self.eye)
        surf.blit(lbl, (cx - lbl.get_width()//2, cy + self.r + 4))

        if self.max_hp > 1:
            bw = self.r * 2
            pygame.draw.rect(surf, (51, 0, 0),  (cx - self.r, cy + self.r + 18, bw, 5))
            pygame.draw.rect(surf, self.body,   (cx - self.r, cy + self.r + 18,
                                                  int(bw * self.hp / self.max_hp), 5))

class Powerup:
    COLORS = {"double": GREEN,  "nova": ORANGE, "clone": PURPLE}
    LABELS = {"double": "x2",   "nova": "PWR",  "clone": "2x"}
    NAMES  = {"double": "DOUBLE SHOT", "nova": "NOVA BLAST", "clone": "CLONE"}

    def __init__(self):
        self.kind  = random.choice(["double", "nova", "clone"])
        self.x     = float(W - 20)
        self.y     = float(random.randint(70, H - 100))
        self.vx    = -2.5
        self.r     = 16
        self.pulse = random.uniform(0, math.pi * 2)
        self.life  = 320
        self.color = self.COLORS[self.kind]

    def update(self):
        self.x += self.vx; self.life -= 1; self.pulse += 0.08

    def alive(self):
        return self.x > -self.r and self.life > 0

    def draw(self, surf, font_sm):
        sc = 1 + math.sin(self.pulse) * 0.15
        r  = int(self.r * sc)
        cx, cy = int(self.x), int(self.y)
        glow = pygame.Surface((r*4, r*4), pygame.SRCALPHA)
        pygame.draw.circle(glow, (*self.color, 50), (r*2, r*2), r*2)
        surf.blit(glow, (cx - r*2, cy - r*2))
        pygame.draw.circle(surf, self.color, (cx, cy), r)
        lbl = font_sm.render(self.LABELS[self.kind], True, BG)
        surf.blit(lbl, (cx - lbl.get_width()//2, cy - lbl.get_height()//2))
        nm = font_sm.render(self.NAMES[self.kind], True, self.color)
        surf.blit(nm, (cx - nm.get_width()//2, cy + r + 5))

class Particle:
    def __init__(self, x, y, color):
        a = random.uniform(0, math.pi * 2)
        sp = random.uniform(1.5, 5.5)
        self.x, self.y = float(x), float(y)
        self.vx, self.vy = math.cos(a)*sp, math.sin(a)*sp
        self.life = random.randint(25, 50); self.max = self.life
        self.color = color; self.size = random.uniform(2, 5)
        self.lbl = rand_binary(4) if random.random() < 0.3 else None

    def update(self):
        self.x += self.vx; self.y += self.vy
        self.vy += 0.09; self.vx *= 0.96; self.life -= 1

    def alive(self): return self.life > 0

    def draw(self, surf, font_sm):
        al = self.life / self.max
        col = tuple(int(v * al) for v in self.color)
        if self.lbl:
            txt = font_sm.render(self.lbl, True, col)
            surf.blit(txt, (int(self.x), int(self.y)))
        else:
            r = max(1, int(self.size * al))
            pygame.draw.circle(surf, col, (int(self.x), int(self.y)), r)

class FloatingText:
    def __init__(self, x, y, text, color):
        self.x, self.y = float(x), float(y)
        self.text = text; self.color = color
        self.life = 55; self.vy = -1.6

    def update(self): self.y += self.vy; self.life -= 1

    def alive(self): return self.life > 0

    def draw(self, surf, font_md):
        al = min(255, int(self.life / 18 * 255))
        txt = font_md.render(self.text, True, self.color)
        txt.set_alpha(al)
        surf.blit(txt, (int(self.x) - txt.get_width()//2, int(self.y)))

class Shockwave:
    def __init__(self, x, y):
        self.x, self.y = x, y
        self.r = 10; self.max_r = 280
        self.life = 32; self.max_life = 32

    def update(self):
        prog = 1 - (self.life / self.max_life)
        self.r = int(self.max_r * prog); self.life -= 1

    def alive(self): return self.life > 0

    def draw(self, surf):
        al = int(self.life / self.max_life * 200)
        if self.r > 0:
            s = pygame.Surface((W, H), pygame.SRCALPHA)
            pygame.draw.circle(s, (*ORANGE, al),      (self.x, self.y), self.r, 4)
            pygame.draw.circle(s, (*YELLOW, al//2),   (self.x, self.y), max(1, self.r-14), 8)
            surf.blit(s, (0, 0))

class Game:
    def __init__(self):
        pygame.init()
        self.screen = pygame.display.set_mode((W, H))
        pygame.display.set_caption("CYBER DEFENDER")
        self.clock = pygame.time.Clock()

        self.font_sm  = pygame.font.SysFont("Courier New", 10, bold=True)
        self.font_md  = pygame.font.SysFont("Courier New", 13, bold=True)
        self.font_lg  = pygame.font.SysFont("Courier New", 20, bold=True)
        self.font_xl  = pygame.font.SysFont("Courier New", 26, bold=True)
        self.font_xxl = pygame.font.SysFont("Courier New", 34, bold=True)

        self.state = "menu"
        self.reset()

    def reset(self):
        self.robot         = Robot()
        self.score         = 0
        self.lives         = 3
        self.g_level       = 1
        self.killed        = 0
        self.stars         = 0
        self.bullets       = []
        self.viruses       = []
        self.particles     = []
        self.float_texts   = []
        self.powerups      = []
        self.shockwaves    = []
        self.spawn_timer   = 0
        self.spawn_int     = 90
        self.pup_timer     = 0
        self.double_active = False
        self.double_timer  = 0
        self.clone_active  = False
        self.clone_timer   = 0
        self.t             = 0.0

    def add_ft(self, x, y, text, color):
        self.float_texts.append(FloatingText(x, y, text, color))

    def fire(self, from_y, mouse_y):
        b = Bullet(Robot.X + 18, from_y, mouse_y)
        self.bullets.append(b)
        if self.double_active:
            self.bullets.append(Bullet(Robot.X + 18, from_y - 10, mouse_y - 10))
            self.bullets.append(Bullet(Robot.X + 18, from_y + 10, mouse_y + 10))

    def trigger_nova(self):
        cx, cy = Robot.X, int(self.robot.y)
        self.shockwaves.append(Shockwave(cx, cy))
        self.add_ft(W//2, H//2 - 40, "NOVA BLAST!", ORANGE)
        to_del = []
        for v in self.viruses:
            if math.hypot(v.x - cx, v.y - cy) < 280:
                pts = v.pts * (1 + self.g_level // 2)
                self.score += pts; self.killed += 1
                self.add_ft(v.x, v.y - v.r, f"+{pts}", ORANGE)
                for _ in range(8): self.particles.append(Particle(v.x, v.y, v.body))
                to_del.append(v)
        self.viruses = [v for v in self.viruses if v not in to_del]
        self.update_stars()

    def update_stars(self):
        s = sum(1 for th in STAR_T if self.score >= th)
        if s > self.stars:
            self.stars = s
            self.add_ft(W//2, 90, f"★  STAR {self.stars} UNLOCKED!  ★", YELLOW)
            if self.stars >= 6:
                self.state = "win"

    def update(self, mouse_y):
        self.t += 1 / FPS
        self.robot.update(mouse_y)

        self.spawn_timer += 1
        if self.spawn_timer >= self.spawn_int:
            self.viruses.append(Virus(self.g_level))
            self.spawn_timer = 0
            self.spawn_int = max(22, 90 - self.g_level * 8)

        self.pup_timer += 1
        if self.pup_timer >= max(140, 220 - self.g_level * 6):
            self.powerups.append(Powerup()); self.pup_timer = 0

        if self.double_active:
            self.double_timer -= 1
            if self.double_timer <= 0: self.double_active = False
        if self.clone_active:
            self.clone_timer -= 1
            if self.clone_timer <= 0: self.clone_active = False

        for b in self.bullets: b.update()
        self.bullets = [b for b in self.bullets if b.alive()]

        for v in self.viruses: v.update(self.t)

        for p in self.powerups: p.update()
        collected = []
        for p in self.powerups:
            if math.hypot(p.x - Robot.X, p.y - self.robot.y) < p.r + 24:
                if p.kind == "double":
                    self.double_active = True; self.double_timer = 380
                    self.add_ft(Robot.X + 60, self.robot.y - 50, "DOUBLE SHOT!", GREEN)
                elif p.kind == "nova":
                    self.trigger_nova()
                elif p.kind == "clone":
                    self.clone_active = True; self.clone_timer = 420
                    self.add_ft(Robot.X + 60, self.robot.y - 50, "CLONE ACTIVE!", PURPLE)
                collected.append(p)
        self.powerups = [p for p in self.powerups if p not in collected and p.alive()]

        for s in self.shockwaves: s.update()
        self.shockwaves = [s for s in self.shockwaves if s.alive()]

        dead_b, dead_v = set(), set()
        for bi, b in enumerate(self.bullets):
            for vi, v in enumerate(self.viruses):
                if math.hypot(b.x - v.x, b.y - v.y) < v.r + 7:
                    dead_b.add(bi); v.hp -= 1
                    for _ in range(6): self.particles.append(Particle(v.x, v.y, v.body))
                    if v.hp <= 0:
                        pts = v.pts * (1 + self.g_level // 2)
                        self.score += pts; self.killed += 1
                        self.add_ft(v.x, v.y - v.r, f"+{pts}", GREEN)
                        dead_v.add(vi)
                        self.g_level = min(12, 1 + self.killed // 8)
                    else:
                        self.add_ft(v.x, v.y - v.r, "HIT!", ORANGE)
                    self.update_stars()
        self.bullets = [b for i, b in enumerate(self.bullets) if i not in dead_b]
        self.viruses = [v for i, v in enumerate(self.viruses) if i not in dead_v]

        surviving = []
        for v in self.viruses:
            if not v.alive():
                self.lives -= 1
                for _ in range(10): self.particles.append(Particle(Robot.X + 10, int(v.y), v.body))
                self.add_ft(W//2, H//2, "BREACHED!  -1 LIFE", RED)
                if self.lives <= 0: self.state = "gameover"
            else:
                surviving.append(v)
        self.viruses = surviving

        for p in self.particles: p.update()
        self.particles = [p for p in self.particles if p.alive()]
        for f in self.float_texts: f.update()
        self.float_texts = [f for f in self.float_texts if f.alive()]

    def draw_grid(self):
        for x in range(0, W, 44):
            pygame.draw.line(self.screen, GRID_COL, (x, 0), (x, H))
        for y in range(0, H, 44):
            pygame.draw.line(self.screen, GRID_COL, (0, y), (W, y))

    def draw_hud(self):
        pygame.draw.rect(self.screen, (10, 10, 34), (0, 0, W, 38))
        pygame.draw.line(self.screen, (26, 42, 90), (0, 38), (W, 38))

        def item(label, val, cx):
            l = self.font_sm.render(label, True, BLUE)
            v = self.font_md.render(str(val), True, CYAN)
            self.screen.blit(l, (cx - l.get_width()//2, 4))
            self.screen.blit(v, (cx - v.get_width()//2, 16))

        item("SCORE",  self.score,   90)
        item("LEVEL",  self.g_level, 210)
        item("LIVES",  "♥ " * self.lives, 320)
        item("KILLS",  self.killed,  440)

        sl = self.font_sm.render("STARS", True, BLUE)
        self.screen.blit(sl, (560 - sl.get_width()//2, 4))
        for i in range(6):
            s = self.font_md.render("★" if i < self.stars else "☆", True, YELLOW if i < self.stars else (45,45,45))
            self.screen.blit(s, (546 + i * 22, 15))

        for i, (kind, col, abbr) in enumerate([("double", GREEN, "x2"), ("nova", ORANGE, "NV"), ("clone", PURPLE, "CL")]):
            active = (kind == "double" and self.double_active) or (kind == "clone" and self.clone_active)
            al = 255 if active else 70
            lc = tuple(int(v * al / 255) for v in col)
            draw_rounded_rect(self.screen, lc, (W - 108 + i * 34, 8, 28, 22), 4, 1)
            lb = self.font_sm.render(abbr, True, lc)
            self.screen.blit(lb, (W - 108 + i * 34 + 14 - lb.get_width()//2, 16))

    def draw_progress_bar(self):
        prev = STAR_T[self.stars - 1] if self.stars > 0 else 0
        curr = STAR_T[self.stars]     if self.stars < 6  else STAR_T[5]
        pct  = 1.0 if self.stars >= 6 else min(1.0, (self.score - prev) / max(1, curr - prev))
        bx, by, bw, bh = 10, H - 13, W - 20, 8
        pygame.draw.rect(self.screen, (26, 48, 96), (bx, by, bw, bh))
        pygame.draw.rect(self.screen, YELLOW if self.stars >= 6 else CYAN, (bx, by, int(bw * pct), bh))
        pygame.draw.rect(self.screen, (42, 74, 128), (bx, by, bw, bh), 1)
        if self.stars < 6:
            lbl = self.font_sm.render(f"Next star: {curr - self.score} pts", True, BLUE)
            self.screen.blit(lbl, (W - lbl.get_width() - 14, H - 25))

    def draw_playing(self):
        self.screen.fill(BG)
        self.draw_grid()

        scan = pygame.Surface((W, H), pygame.SRCALPHA)
        for y in range(0, H, 4):
            pygame.draw.line(scan, (0, 0, 0, 16), (0, y), (W, y))
        self.screen.blit(scan, (0, 0))

        dl = pygame.Surface((2, H), pygame.SRCALPHA)
        dl.fill((*CYAN, 40))
        self.screen.blit(dl, (Robot.X + 34, 0))

        for s in self.shockwaves: s.draw(self.screen)
        for p in self.powerups:   p.draw(self.screen, self.font_sm)
        for v in self.viruses:    v.draw(self.screen, self.font_sm, self.t)
        for b in self.bullets:    b.draw(self.screen, self.font_sm)
        for p in self.particles:  p.draw(self.screen, self.font_sm)
        for f in self.float_texts: f.draw(self.screen, self.font_md)

        if self.double_active:
            gs = pygame.Surface((W, H), pygame.SRCALPHA)
            ry = int(self.robot.y)
            pygame.draw.line(gs, (*GREEN, 45), (Robot.X, ry - 10), (W, ry - 10))
            pygame.draw.line(gs, (*GREEN, 45), (Robot.X, ry + 10), (W, ry + 10))
            self.screen.blit(gs, (0, 0))

        if self.clone_active:
            self.robot.draw(self.screen, self.font_sm, self.t, tint=PURPLE, alpha=140, offset_y=-90)

        self.robot.draw(self.screen, self.font_sm, self.t)
        self.draw_hud()
        self.draw_progress_bar()

    def draw_overlay_panel(self, border_col, w=540, h=360):
        px, py = W//2 - w//2, H//2 - h//2
        panel = pygame.Surface((w, h), pygame.SRCALPHA)
        panel.fill((8, 8, 30, 218))
        self.screen.blit(panel, (px, py))
        pygame.draw.rect(self.screen, border_col, (px, py, w, h), 2, border_radius=8)
        return px, py

    def draw_menu(self):
        self.screen.fill(BG)
        self.draw_grid()
        self.robot.draw(self.screen, self.font_sm, self.t)

        px, py = self.draw_overlay_panel(CYAN, 560, 390)

        title = self.font_xxl.render("CYBER DEFENDER", True, CYAN)
        self.screen.blit(title, (W//2 - title.get_width()//2, py + 14))
        sub = self.font_sm.render("Binary-powered defense against cyber viruses", True, BLUE)
        self.screen.blit(sub, (W//2 - sub.get_width()//2, py + 54))

        for i, (txt, col) in enumerate([
            ("Robot on the LEFT — move mouse UP/DOWN to aim",  WHITE),
            ("Click anywhere to fire binary projectiles",       WHITE),
            ("Collect glowing orbs for power-up bonuses!",     YELLOW),
        ]):
            s = self.font_sm.render(txt, True, col)
            self.screen.blit(s, (W//2 - s.get_width()//2, py + 74 + i * 17))

        pups = [("DOUBLE SHOT", GREEN, "Triple bullets"),
                ("NOVA BLAST",  ORANGE,"Wipes all viruses"),
                ("CLONE",       PURPLE,"Duplicate robot")]
        for i, (nm, col, desc) in enumerate(pups):
            bx = px + 22 + i * 172
            by = py + 130
            draw_rounded_rect(self.screen, col, (bx, by, 158, 50), 6, 1)
            n = self.font_sm.render(f"◆ {nm}", True, col)
            d = self.font_sm.render(desc, True, (180,180,180))
            self.screen.blit(n, (bx + 6, by + 8))
            self.screen.blit(d, (bx + 6, by + 28))

        th = self.font_sm.render("STAR THRESHOLDS:", True, BLUE)
        self.screen.blit(th, (W//2 - th.get_width()//2, py + 198))
        for i, v in enumerate(STAR_T):
            s = self.font_sm.render(f"{'★'*(i+1)} {v}pts", True, YELLOW)
            self.screen.blit(s, (px + 22 + (i % 3) * 182, py + 215 + (i // 3) * 17))

        ctrl = self.font_sm.render("MOUSE = aim  |  CLICK = fire  |  ESC = quit", True, (90,130,190))
        self.screen.blit(ctrl, (W//2 - ctrl.get_width()//2, py + 264))

        pulse = 0.6 + 0.4 * math.sin(self.t * 3)
        bc = tuple(int(v * pulse) for v in CYAN)
        draw_rounded_rect(self.screen, DARK, (W//2-95, py+292, 190, 42), 6)
        draw_rounded_rect(self.screen, bc,   (W//2-95, py+292, 190, 42), 6, 2)
        btn = self.font_lg.render("▶  BOOT PROTOCOL", True, bc)
        self.screen.blit(btn, (W//2 - btn.get_width()//2, py + 303))

    def draw_win(self):
        self.screen.fill(BG)
        self.draw_grid()
        for p in self.particles: p.draw(self.screen, self.font_sm)

        px, py = self.draw_overlay_panel(GREEN, 560, 350)

        title = self.font_xl.render("DEFENDER UNIT SAYS:", True, GREEN)
        self.screen.blit(title, (W//2 - title.get_width()//2, py + 18))

        robot_txt = self.font_lg.render("[ ROBOT ]", True, CYAN)
        self.screen.blit(robot_txt, (W//2 - robot_txt.get_width()//2, py + 52))

        for i, (msg, col) in enumerate([
            ('"Defense protocol: COMPLETE."',                   GREEN),
            ('"All cyber threats successfully neutralized."',   CYAN),
            ('"System integrity: 100%. No intrusions."',        CYAN),
            ('"The network is SECURE. Mission accomplished."',  GREEN),
        ]):
            s = self.font_sm.render(msg, True, col)
            self.screen.blit(s, (W//2 - s.get_width()//2, py + 86 + i * 19))

        for i in range(6):
            s = self.font_xl.render("★", True, YELLOW)
            self.screen.blit(s, (W//2 - 3*32 + i*32, py + 168))

        stats = self.font_sm.render(f"Final Score: {self.score}   |   Viruses Destroyed: {self.killed}", True, (90,160,90))
        self.screen.blit(stats, (W//2 - stats.get_width()//2, py + 220))

        pulse = 0.6 + 0.4 * math.sin(self.t * 3)
        gc = tuple(int(v * pulse) for v in GREEN)
        draw_rounded_rect(self.screen, DARK, (W//2-95, py+254, 190, 42), 6)
        draw_rounded_rect(self.screen, gc,   (W//2-95, py+254, 190, 42), 6, 2)
        btn = self.font_lg.render("↺  NEW GAME", True, gc)
        self.screen.blit(btn, (W//2 - btn.get_width()//2, py + 265))

    def draw_gameover(self):
        self.screen.fill(BG)
        self.draw_grid()
        for p in self.particles: p.draw(self.screen, self.font_sm)

        px, py = self.draw_overlay_panel(RED, 560, 370)

        title = self.font_xl.render("CRITICAL INTRUSION DETECTED", True, RED)
        self.screen.blit(title, (W//2 - title.get_width()//2, py + 18))

        for i, (msg, col) in enumerate([
            ('"WARNING: Viral entities breached the firewall."', (255,106,106)),
            ('"System infection in progress..."',                 (255,106,106)),
            ('"Encrypting files: ████████░░ 82%"',               (255,154,106)),
            ('"Corrupted processes: ALL"',                        (255,154,106)),
            ('"SYSTEM COMPROMISED. Rebooting..."',                (255,106,106)),
        ]):
            s = self.font_sm.render(msg, True, col)
            self.screen.blit(s, (W//2 - s.get_width()//2, py + 62 + i * 19))

        for i in range(6):
            ch  = "★" if i < self.stars else "☆"
            col = YELLOW if i < self.stars else (40, 40, 40)
            s   = self.font_xl.render(ch, True, col)
            self.screen.blit(s, (W//2 - 3*32 + i*32, py + 172))

        stats = self.font_sm.render(f"Score: {self.score}   |   Stars: {self.stars}/6", True, (120,80,80))
        self.screen.blit(stats, (W//2 - stats.get_width()//2, py + 226))

        pulse = 0.6 + 0.4 * math.sin(self.t * 3)
        rc = tuple(int(v * pulse) for v in RED)
        draw_rounded_rect(self.screen, DARK, (W//2-95, py+260, 190, 42), 6)
        draw_rounded_rect(self.screen, rc,   (W//2-95, py+260, 190, 42), 6, 2)
        btn = self.font_lg.render("↺  REBOOT", True, rc)
        self.screen.blit(btn, (W//2 - btn.get_width()//2, py + 271))

    def run(self):
        while True:
            mx, my = pygame.mouse.get_pos()

            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    pygame.quit(); sys.exit()
                if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                    pygame.quit(); sys.exit()
                if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                    if self.state == "playing":
                        self.fire(self.robot.y, my)
                        if self.clone_active:
                            self.fire(self.robot.y - 90, my)
                    elif self.state in ("menu", "win", "gameover"):
                        self.reset()
                        self.state = "playing"

            self.t += 1 / FPS

            if self.state == "playing":
                self.update(my)
                self.draw_playing()
            elif self.state == "menu":
                self.robot.update(my)
                self.draw_menu()
            elif self.state == "win":
                for p in self.particles: p.update()
                self.particles = [p for p in self.particles if p.alive()]
                if random.random() < 0.12:
                    self.particles.append(Particle(random.randint(0,W), random.randint(0,H), YELLOW))
                self.draw_win()
            elif self.state == "gameover":
                for p in self.particles: p.update()
                self.particles = [p for p in self.particles if p.alive()]
                self.draw_gameover()

            pygame.display.flip()
            self.clock.tick(FPS)


if __name__ == "__main__":
    Game().run()
