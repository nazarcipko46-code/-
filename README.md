import 'pygame'
import random
import math
import sys

'pygame'.init()
сохрани
# Налаштування
TILE = 40
COLS, ROWS = 21, 15
WIDTH, HEIGHT = COLS * TILE, ROWS * TILE
FPS = 60

screen = 'pygame'.display.set_mode((WIDTH, HEIGHT))
'pygame'.display.set_caption("DARK ESCAPE - Horror Game")
clock = 'pygame'.time.Clock()

font = 'pygame'.font.SysFont("consolas", 24, bold=True)
small_font = 'pygame'.font.SysFont("consolas", 17)
big_font = 'pygame'.font.SysFont("consolas", 48, bold=True)

BLACK = (5, 6, 10)
FLOOR = (40, 41, 47)
WALL = (17, 19, 25)
WHITE = (230, 230, 230)
RED = (220, 35, 40)
GREEN = (50, 220, 100)
GOLD = (255, 210, 60)

# # = стіна, . = підлога
MAP = [
    "#####################",
    "#.........#.........#",
    "#.###.###.#.###.###.#",
    "#...#.....#.....#...#",
    "###.#.###...###.#.###",
    "#.....#.......#.....#",
    "#.###.#.#####.#.###.#",
    "#.#...#...#...#...#.#",
    "#.#.#####.#.#####.#.#",
    "#...#.....#.....#...#",
    "###.#.###...###.#.###",
    "#.....#.......#.....#",
    "#.###.#.#####.#.###.#",
    "#.........#.........#",
    "#####################",
]

def reset_game():
    global player, monster, key_pos, exit_pos
    global has_key, game_state, message_timer

    player = [1.5 * TILE, 1.5 * TILE]
    monster = [19.5 * TILE, 13.5 * TILE]
    key_pos = [19.5 * TILE, 1.5 * TILE]
    exit_pos = [19.5 * TILE, 13.5 * TILE]

    # Монстр починає біля виходу, але рухається за гравцем
    monster = [17.5 * TILE, 13.5 * TILE]
    has_key = False
    game_state = "playing"
    message_timer = 0

def wall_at_pixel(x, y):
    col = int(x // TILE)
    row = int(y // TILE)

    if col < 0 or row < 0 or col >= COLS or row >= ROWS:
        return True

    return MAP[row][col] == "#"

def can_move(x, y, radius=10):
    return not (
        wall_at_pixel(x - radius, y - radius)
        or wall_at_pixel(x + radius, y - radius)
        or wall_at_pixel(x - radius, y + radius)
        or wall_at_pixel(x + radius, y + radius)
    )

def move_entity(entity, dx, dy, speed):
    # Окрема перевірка осей дозволяє ковзати вздовж стін
    nx = entity[0] + dx * speed
    if can_move(nx, entity[1]):
        entity[0] = nx

    ny = entity[1] + dy * speed
    if can_move(entity[0], ny):
        entity[1] = ny

def draw_text(text, fnt, color, x, y, center=False):
    image = fnt.render(text, True, color)
    rect = image.get_rect()
    if center:
        rect.center = (x, y)
    else:
        rect.topleft = (x, y)
    screen.blit(image, rect)

def draw_world():
    for row in range(ROWS):
        for col in range(COLS):
            rect = 'pygame'.Rect(
                col * TILE, row * TILE, TILE, TILE
            )

            if MAP[row][col] == "#":
                'pygame'.draw.rect(screen, WALL, rect)
                'pygame'.draw.rect(
                    screen, (30, 32, 40), rect, 2
                )
            else:
                'pygame'.draw.rect(screen, FLOOR, rect)
                'pygame'.draw.line(
                    screen, (47, 48, 54),
                    (rect.x, rect.y),
                    (rect.right, rect.y)
                )

    # Вихід
    ex, ey = int(exit_pos[0]), int(exit_pos[1])
    exit_color = GREEN if has_key else RED
    'pygame'.draw.rect(
        screen, exit_color,
        (ex - 13, ey - 17, 26, 34), 3
    )
    'pygame'.draw.circle(
        screen, exit_color, (ex + 5, ey), 3
    )

    # Ключ
    if not has_key:
        kx, ky = int(key_pos[0]), int(key_pos[1])
        'pygame'.draw.circle(screen, GOLD, (kx, ky), 8, 3)
        'pygame'.draw.line(
            screen, GOLD, (kx + 7, ky), (kx + 17, ky), 3
        )
        'pygame'.draw.line(
            screen, GOLD, (kx + 13, ky), (kx + 13, ky + 5), 3
        )

    # Гравець
    px, py = int(player[0]), int(player[1])
    'pygame'.draw.circle(screen, (100, 190, 255), (px, py), 11)
    'pygame'.draw.circle(screen, WHITE, (px, py), 11, 2)

    # Монстр
    mx, my = int(monster[0]), int(monster[1])
    'pygame'.draw.circle(screen, (100, 12, 20), (mx, my), 15)
    'pygame'.draw.circle(screen, RED, (mx - 5, my - 3), 3)
    'pygame'.draw.circle(screen, RED, (mx + 5, my - 3), 3)

def draw_flashlight():
    # Затемнення з прозорим кругом навколо гравця
    darkness = 'pygame'.Surface(
        (WIDTH, HEIGHT), 'pygame'.SRCALPHA
    )
    darkness.fill((0, 0, 0, 242))

    px, py = int(player[0]), int(player[1])

    # Кілька шарів створюють м'яке світло
    for radius, alpha in [
        (125, 0), (115, 15), (100, 35),
        (80, 65), (55, 100), (30, 135)
    ]:
        'pygame'.draw.circle(
            darkness, (0, 0, 0, alpha),
            (px, py), radius
        )

    # Центральна видима область
    'pygame'.draw.circle(
        darkness, (0, 0, 0, 0), (px, py), 88
    )

    screen.blit(darkness, (0, 0))

    # Темний віньєтковий ефект навколо екрана
    # Сам ліхтарик завжди рухається за гравцем

def draw_hud():
    'pygame'.draw.rect(
        screen, (10, 10, 15), (0, 0, WIDTH, 36)
    )

    status = "KEY: FOUND" if has_key else "KEY: MISSING"
    color = GREEN if has_key else GOLD

    draw_text("DARK ESCAPE", small_font, WHITE, 10, 8)
    draw_text(status, small_font, color, 240, 8)
    draw_text("WASD / ARROWS", small_font, WHITE, 480, 8)

def draw_end_screen(title, subtitle, color):
    overlay = 'pygame'.Surface((WIDTH, HEIGHT), 'pygame'.SRCALPHA)
    overlay.fill((0, 0, 0, 205))
    screen.blit(overlay, (0, 0))

    draw_text(
        title, big_font, color,
        WIDTH // 2, HEIGHT // 2 - 45, True
    )
    draw_text(
        subtitle, font, WHITE,
        WIDTH // 2, HEIGHT // 2 + 15, True
    )
    draw_text(
        "Press R to restart | ESC to quit",
        small_font, GOLD,
        WIDTH // 2, HEIGHT // 2 + 60, True
    )

reset_game()

while True:
    dt = min(clock.tick(FPS) / 16.667, 2.0)

    for event in 'pygame'.event.get():
        if event.type == 'pygame'.QUIT:
            'pygame'.quit()
            sys.exit()

        if event.type == 'pygame'.KEYDOWN:
            if event.key == 'pygame'.K_ESCAPE:
                'pygame'.quit()
                sys.exit()

            if event.key == 'pygame'.K_r and game_state != "playing":
                reset_game()

    if game_state == "playing":
        keys = 'pygame'.key.get_pressed()

        dx = 0
        dy = 0

        if keys['pygame'.K_a] or keys['pygame'.K_LEFT]:
            dx -= 1
        if keys['pygame'.K_d] or keys['pygame'.K_RIGHT]:
            dx += 1
        if keys['pygame'.K_w] or keys['pygame'.K_UP]:
            dy -= 1
        if keys['pygame'.K_s] or keys['pygame'.K_DOWN]:
            dy += 1

        length = math.hypot(dx, dy)
        if length:
            dx /= length
            dy /= length

        move_entity(player, dx, dy, 3.0 * dt)

        # Монстр шукає шлях до гравця
        vx = player[0] - monster[0]
        vy = player[1] - monster[1]
        distance = math.hypot(vx, vy)

        if distance > 1:
            mx = vx / distance
            my = vy / distance

            # Монстр повільніший за гравця
            move_entity(monster, mx, my, 1.45 * dt,)

        # Підібрати ключ
        if not has_key:
            if math.hypot(
                player[0] - key_pos[0],
                player[1] - key_pos[1]
            ) < 22:
                has_key = True

        # Дістатися виходу
        if math.hypot(
            player[0] - exit_pos[0],
            player[1] - exit_pos[1]
        ) < 24:
            if has_key:
                game_state = "won"

        # Монстр спіймав гравця
        if math.hypot(
            player[0] - monster[0],
            player[1] - monster[1]
        ) < 23:
            game_state = "lost"

    draw_world()
    draw_flashlight()
    draw_hud()

    if game_state == "won":
        draw_end_screen(
            "YOU ESCAPED!",
            "You survived the darkness.",
            GREEN
        )
    elif game_state == "lost":
        draw_end_screen(
            "YOU DIED!",
            "The monster found you...",
            RED
        )

    'pygame'.display.flip()
