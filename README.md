import math
import os
import random
import sys
import pygame

pygame.init()

WIDTH, HEIGHT = 800, 600
FPS = 60
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Honeybee - Olivia Rodrigo")
clock = pygame.time.Clock()


class Particle:

  def __init__(self, x, y, color=None, speed_range=(2, 7)):
    self.x = x
    self.y = y
    angle = random.uniform(0, 2 * math.pi)
    speed = random.uniform(*speed_range)
    self.vx = math.cos(angle) * speed
    self.vy = math.sin(angle) * speed
    self.life = random.randint(30, 60)
    self.max_life = self.life
    self.color = color or random.choice([
        (255, 105, 180),
        (255, 182, 193),
        (255, 20, 147),
        (255, 255, 255),
    ])
    self.size = random.uniform(2, 5)

  def update(self):
    self.x += self.vx
    self.y += self.vy
    self.vy += 0.1
    self.life -= 1

  def draw(self, surface):
    if self.life > 0:
      alpha = int((self.life / self.max_life) * 255)
      surf = pygame.Surface((int(self.size * 2), int(self.size * 2)), pygame.SRCALPHA)
      pygame.draw.circle(surf, (*self.color, alpha),
                         (int(self.size), int(self.size)), self.size)
      surface.blit(surf, (self.x - self.size, self.y - self.size))


class InteractiveHeart:

  def __init__(self, x, y):
    self.x = x
    self.y = y
    self.scale = 0.05
    self.max_scale = random.uniform(1.2, 1.8)
    self.growing = True
    self.particles = []
    self.is_exploded = False
    self.explosion_started = False

  def update(self):
    if self.growing:
      self.scale += 0.022
      if self.scale >= self.max_scale:
        self.growing = False
    elif not self.is_exploded:
      # Trigger explosion
      self.is_exploded = True
      self.explosion_started = True
      for _ in range(120):
        self.particles.append(Particle(self.x, self.y))

    # Update particles if exploded
    for p in self.particles[:]:
      p.update()
      if p.life <= 0:
        self.particles.remove(p)

  def draw(self, surface):
    if not self.is_exploded:
      points = []
      t = 0
      while t < 2 * math.pi:
        hx = 16 * (math.sin(t) ** 3)
        hy = -(
            13 * math.cos(t)
            - 5 * math.cos(2 * t)
            - 2 * math.cos(3 * t)
            - math.cos(4 * t)
        )

        # Scale and translate
        px = self.x + hx * self.scale * 3
        py = self.y + hy * self.scale * 3
        points.append((px, py))
        t += 0.05

      if len(points) > 2:
        glow = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        glow_alpha = int(26 + 22 * self.scale / self.max_scale)
        pygame.draw.circle(
          glow, (255, 63, 158, glow_alpha),
          (int(self.x), int(self.y)), int(78 * self.scale)
        )
        surface.blit(glow, (0, 0))
        pygame.draw.polygon(surface, (183, 34, 104), points)
        inner_points = [
          (self.x + (px - self.x) * 0.92, self.y + (py - self.y) * 0.92)
          for px, py in points
        ]
        pygame.draw.polygon(surface, (247, 74, 142), inner_points)
        pygame.draw.aalines(surface, (255, 214, 238), True, points)
        pygame.draw.ellipse(
          surface, (255, 190, 219),
          (self.x - 25 * self.scale, self.y - 34 * self.scale,
           12 * self.scale, 24 * self.scale)
        )
    else:
      # Draw explosion particles
      for p in self.particles:
        p.draw(surface)


class Firework:

  def __init__(self, x, target_x, target_y, color):
    self.x = x
    self.y = HEIGHT // 2 - 12
    self.target_x = target_x
    self.target_y = target_y
    self.color = color
    self.vx = (target_x - x) / 42
    self.vy = (target_y - self.y) / 42
    self.trail = []
    self.particles = []
    self.exploded = False

  def update(self):
    if not self.exploded:
      self.trail.append((self.x, self.y))
      if len(self.trail) > 9:
        self.trail.pop(0)
      self.x += self.vx
      self.y += self.vy
      if math.hypot(self.x - self.target_x, self.y - self.target_y) < 8:
        self.exploded = True
        for _ in range(48):
          self.particles.append(Particle(self.x, self.y, self.color, (1.5, 5)))
    else:
      for particle in self.particles[:]:
        particle.update()
        if particle.life <= 0:
          self.particles.remove(particle)

  def draw(self, surface):
    if not self.exploded:
      for index, (trail_x, trail_y) in enumerate(self.trail):
        alpha = int(35 + index * 18)
        trail_surface = pygame.Surface((8, 8), pygame.SRCALPHA)
        pygame.draw.circle(trail_surface, (*self.color, alpha), (4, 4), 2)
        surface.blit(trail_surface, (trail_x - 4, trail_y - 4))
      pygame.draw.circle(surface, (255, 255, 255), (int(self.x), int(self.y)), 3)
    else:
      for particle in self.particles:
        particle.draw(surface)


class LyricDisplay:

  def __init__(self, index):
    self.index = index
    self.life = FPS * 8

  def update(self):
    self.life -= 1

  def draw(self, surface, font, small_font):
    if self.life <= 0:
      return
    alpha = max(0, min(255, self.life * 255 // (FPS * 2)))
    line = LYRICS[self.index % len(LYRICS)]
    text = font.render(line, True, (255, 220, 242))
    shadow = small_font.render("after the spark", True, (255, 160, 211))
    text.set_alpha(alpha)
    shadow.set_alpha(alpha // 2)
    surface.blit(shadow, (WIDTH // 2 - shadow.get_width() // 2, HEIGHT - 76))
    surface.blit(text, (WIDTH // 2 - text.get_width() // 2, HEIGHT - 52))


LYRICS = [
    "So I guess that it's true",
    "Time can heal even the worst of wounds",
  "And the clichés I knew",
    "Seemed so commonplace when I saw you",
    "Let's just walk in the dark",
    "Hop the fence in the park",
    "Baby boy, honeybee",
    "God, I love the way you look at me",
    "And it's too hard to describe this",
    "In a way that feels honest",
    "But even when I'm quiet",
    "I love you, baby, I promise",
    "And I hope I never see what your face looks like going",
    "A face I swear that I could spend my whole life knowing",
    "Here's to hoping",
    "Pick me up, walk me home",
    "And it feels like God threw me a bone",
    "Sticky sweet, tangerine",
    "Would you sit and keep me company?",
    "In the dark, I'm not scared",
    "I just reach and you're right there",
    "Shooting stars, racing cars",
    "Everything I own just feels like ours",
    "It's too hard to describe this",
    "in a way that feels honest",
    "But even when I'm quiet",
    "I love you, baby, I promise",
    "and I hope I never see what your face looks like going",
    "A face I swear that I could spend my whole life knowing",
    "Here's to hoping",
]


def draw_sky(surface, stars):
  for y in range(HEIGHT):
    blend = y / HEIGHT
    color = (int(17 + 32 * blend), int(19 + 10 * blend), int(58 + 42 * blend))
    pygame.draw.line(surface, color, (0, y), (WIDTH, y))
  for x, y, radius, alpha in stars:
    star = pygame.Surface((radius * 4, radius * 4), pygame.SRCALPHA)
    pygame.draw.circle(star, (255, 231, 177, alpha), (radius * 2, radius * 2), radius)
    surface.blit(star, (x - radius * 2, y - radius * 2))
  pygame.draw.ellipse(surface, (48, 43, 91), (-70, 80, 260, 70))
  pygame.draw.ellipse(surface, (53, 47, 99), (590, 120, 300, 80))


def draw_honey_jar(surface, x, y):
  jar_glow = pygame.Surface((120, 140), pygame.SRCALPHA)
  pygame.draw.ellipse(jar_glow, (255, 217, 126, 80), (10, 24, 90, 94))
  surface.blit(jar_glow, (x - 52, y - 48))

  pygame.draw.ellipse(surface, (255, 218, 123), (x - 28, y - 8, 56, 90))
  pygame.draw.rect(surface, (228, 169, 71), (x - 22, y + 10, 44, 50), border_radius=10)
  pygame.draw.rect(surface, (255, 211, 110), (x - 18, y + 14, 36, 42), border_radius=8)
  pygame.draw.rect(surface, (139, 84, 40), (x - 18, y - 18, 36, 14), border_radius=5)
  pygame.draw.ellipse(surface, (255, 219, 134), (x - 10, y + 34, 20, 12))
  pygame.draw.ellipse(surface, (255, 228, 160), (x - 12, y + 8, 24, 12))
  pygame.draw.circle(surface, (255, 202, 89), (x - 8, y + 22), 4)
  pygame.draw.circle(surface, (255, 202, 89), (x + 8, y + 22), 4)
  pygame.draw.line(surface, (245, 191, 80), (x - 8, y + 22), (x + 8, y + 22), 2)

  for yi in range(12, 52, 8):
    pygame.draw.line(surface, (165, 112, 41), (x - 8, y + yi), (x + 8, y + yi), 2)

  pygame.draw.arc(surface, (255, 230, 167), (x - 26, y - 18, 52, 28), 0.9, 2.2, 3)


def draw_butterflies(surface, tick):
  for butterfly in butterflies:
    x = butterfly[0] + math.sin(tick * butterfly[2] + butterfly[3]) * butterfly[4]
    y = butterfly[1] + math.cos(tick * butterfly[2] * 0.9 + butterfly[3]) * butterfly[5]

    wing = 9
    left_x = x - 5
    right_x = x + 5
    left_y = y - 3
    right_y = y - 3

    pygame.draw.ellipse(surface, (175, 118, 255), (left_x - wing, left_y, wing, wing * 0.9))
    pygame.draw.ellipse(surface, (175, 118, 255), (right_x, right_y, wing, wing * 0.9))
    pygame.draw.ellipse(surface, (217, 176, 255), (left_x - wing + 2, left_y + 2, wing - 3, wing * 0.8))
    pygame.draw.ellipse(surface, (217, 176, 255), (right_x + 2, right_y + 2, wing - 3, wing * 0.8))
    pygame.draw.line(surface, (105, 72, 148), (x, y - 4), (x, y + 6), 2)
    pygame.draw.circle(surface, (200, 160, 255), (x, y), 2)


# Main Loop
hearts = []
fireworks = []
lyric = None
lyric_index = 0
lyric_font = pygame.font.SysFont("segoe script", 22, italic=True)
small_font = pygame.font.SysFont("segoe script", 16, italic=True)
credit_font = pygame.font.SysFont("segoe script", 18, italic=True)
music_file = os.path.join(os.path.dirname(__file__), "honeybee.mp3")
if os.path.exists(music_file):
  pygame.mixer.music.load(music_file)
  pygame.mixer.music.play(-1)
random.seed(12)
stars = [
    (random.randint(20, WIDTH - 20), random.randint(24, HEIGHT - 130),
     random.choice((1, 1, 2)), random.randint(90, 210))
    for _ in range(70)
]
butterflies = [
    (random.randint(80, 200), random.randint(300, 440), random.uniform(0.8, 1.4), random.uniform(0, 6.28), random.uniform(18, 30), random.uniform(10, 20))
    for _ in range(8)
]
random.seed()
running = True

while running:
  tick = pygame.time.get_ticks() / 1000.0
  draw_sky(screen, stars)
  draw_butterflies(screen, tick)
  draw_honey_jar(screen, 73, 424)
  credit = credit_font.render("honeybee", True, (255, 221, 153))
  artist = small_font.render("Olivia Rodrigo", True, (255, 194, 205))
  screen.blit(credit, (WIDTH // 2 - credit.get_width() // 2, 25))
  screen.blit(artist, (WIDTH // 2 - artist.get_width() // 2, 49))

  for event in pygame.event.get():
    if event.type == pygame.QUIT:
      running = False
    elif event.type == pygame.MOUSEBUTTONDOWN:
      hearts.append(InteractiveHeart(*event.pos))

  # Update and draw hearts
  for h in hearts[:]:
    h.update()
    h.draw(screen)
    if h.is_exploded and h.explosion_started:
      h.explosion_started = False
      lyric_index += 1
      lyric = LyricDisplay(lyric_index - 1)
      colors = [(255, 94, 176), (255, 206, 90), (121, 220, 255)]
      for side in (-1, 1):
        fireworks.append(Firework(
            h.x + side * 28,
            max(50, min(WIDTH - 50, h.x + side * random.randint(145, 220))),
            max(90, min(HEIGHT - 180, h.y + random.randint(-120, 20))),
            random.choice(colors),
        ))
    if h.is_exploded and not h.particles:
      hearts.remove(h)

  for firework in fireworks[:]:
    firework.update()
    firework.draw(screen)
    if firework.exploded and not firework.particles:
      fireworks.remove(firework)

  if lyric is not None:
    lyric.update()
    lyric.draw(screen, lyric_font, small_font)
    if lyric.life <= 0:
      lyric = None

  pygame.display.flip()
  clock.tick(FPS)

pygame.quit()
sys.exit() 
                
                
