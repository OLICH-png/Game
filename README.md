Есть два файла. server.py это код серверной части, client.py, код клиентской части проекта
client.py

import socket
import pygame

server_ip = '92.63.193.145'
# server_ip = '123.123.123.123'

WIDTH_WINDOW, HEIGHT_WINDOW = 1500, 900
colours = {'0': (255, 255, 0), '1': (255, 0, 0), '2': (0, 255, 0), '3': (0, 255, 255), '4': (128, 0, 128)}
my_name = 'Игрок'
GRID_COLOUR = (150, 150, 150)


def find(s):
    otkr = None
    for i in range(len(s)):
        if s[i] == '<':
            otkr = i
        if s[i] == '>' and otkr != None:
            zakr = i
            res = s[otkr + 1:zakr]
            return res
    return ''


def write_name(x, y, r, name):
    font = pygame.font.Font(None, r)
    text = font.render(name, True, (0, 0, 0))
    rect = text.get_rect(center=(x, y))
    screen.blit(text, rect)


def draw_opponents(data):
    for i in range(len(data)):
        j = data[i].split(' ')

        x = WIDTH_WINDOW // 2 + int(j[0])
        y = HEIGHT_WINDOW // 2 + int(j[1])
        r = int(j[2])
        c = colours[j[3]]
        pygame.draw.circle(screen, c, (x, y), r)

        if len(j) == 5: write_name(x, y, r, j[4])


class Me():
    def __init__(self, data):
        data = data.split()
        self.r = int(data[0])
        self.colour = data[1]

    def update(self, new_r):
        self.r = new_r

    def draw(self):
        if self.r != 0:
            pygame.draw.circle(screen, colours[self.colour],
                               (WIDTH_WINDOW // 2, HEIGHT_WINDOW // 2), self.r)

            write_name(WIDTH_WINDOW // 2, HEIGHT_WINDOW // 2, self.r, my_name)


class Grid():
    def __init__(self, screen):
        self.screen = screen
        self.x = 500
        self.y = 500
        self.start_size = 200
        self.size = self.start_size

    def update(self, r_x, r_y, L):
        self.size = self.start_size // L
        self.x = -self.size + (-r_x) % (self.size)
        self.y = -self.size + (-r_y) % (self.size)

    def draw(self):
        for i in range(WIDTH_WINDOW // self.size + 2):
            pygame.draw.line(self.screen, GRID_COLOUR,
                             [self.x + i * self.size, 0],  # координаты верхнего конца отрезка
                             [self.x + i * self.size, HEIGHT_WINDOW],  # координаты нижнего конца отрезка
                             1)

        for i in range(HEIGHT_WINDOW // self.size + 2):
            pygame.draw.line(self.screen, GRID_COLOUR,
                             [0, self.y + i * self.size],
                             [WIDTH_WINDOW, self.y + i * self.size],
                             1)


# создание окна игры
pygame.init()
screen = pygame.display.set_mode((WIDTH_WINDOW, HEIGHT_WINDOW))
pygame.display.set_caption(' Круги.ИО')

# подключение к серверу
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
sock.connect((server_ip, 12345))

# отправляем серверу свой ник и размеры окна
sock.send(('.' + my_name + ' ' + str(WIDTH_WINDOW) + ' ' + str(HEIGHT_WINDOW) + '.').encode())

# получаем свой размер и цвет
data = sock.recv(64).decode()

# подтверждаем получение
sock.send('!'.encode())
me = Me(data)
grid = Grid(screen)

v = (0, 0)
old_v = (0, 0)
running = True
while running:
    # обработка событий
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
    # считаем положение мыши игрока
    if pygame.mouse.get_focused():
        pos = pygame.mouse.get_pos()
        v = (pos[0] - WIDTH_WINDOW // 2, pos[1] - HEIGHT_WINDOW // 2)

        if (v[0]) ** 2 + (v[1]) ** 2 <= me.r ** 2:
            v = (0, 0)

    # отправляем вектор желаемого направления движения,
    # если он поменялся
    if v != old_v:
        old_v = v
        message = '<' + str(v[0]) + ',' + str(v[1]) + '>'
        sock.send(message.encode())

    # получение нового состояния игрового поля
    try:
        data = sock.recv(2 ** 20)
    except:
        running = False
        continue
    data = data.decode()
    data = find(data)
    data = data.split(',')

    # обработка сообщения с сервера
    if data != ['']:
        parametrs = list(map(int, data[0].split(' ')))
        me.update(parametrs[0])
        grid.update(parametrs[1], parametrs[2], parametrs[3])

        # Рисуем новое состояние игрового поля
        screen.fill('gray25')
        grid.draw()
        draw_opponents(data[1:])
        me.draw()

    pygame.display.update()
pygame.quit()

server.py

import socket
import time
import pygame
import random

work_on_server = False
server_ip = 'localhost'
#server_ip = 'localhost'

FPS = 100
WIDTH_ROOM, HEIGHT_ROOM = 4000, 4000
WIDTH_SERVER_WINDOW, HEIGHT_SERVER_WINDOW = 300, 300

START_PLAYER_SIZE = 50
MICROBE_SIZE = 15

MOBS_QUANTITY = 25
MICROBES_QUANTITY = (WIDTH_ROOM * HEIGHT_ROOM) // 80000

colours = {'0': (255, 255, 0), '1': (255, 0, 0), '2': (0, 255, 0), '3': (0, 255, 255), '4': (128, 0, 128)}


def new_r(R, r):
    return (R ** 2 + r ** 2) ** 0.5


def find(s):
    otkr = None
    for i in range(len(s)):
        if s[i] == '<':
            otkr = i
        if s[i] == '>' and otkr != None:
            zakr = i
            res = s[otkr + 1:zakr]
            res = list(map(int, res.split(',')))
            return res
    return ''


class Microbe():
    def __init__(self, x, y, r, colour):
        self.x = x
        self.y = y
        self.r = r
        self.colour = colour


class Player():
    def __init__(self, conn, addr, x, y, r, colour):
        self.conn = conn
        self.addr = addr
        self.x = x
        self.y = y
        self.r = r
        self.colour = colour
        self.L = 1

        self.name = 'Bot'
        self.width_window = 1000
        self.height_window = 800
        self.w_vision = 1000
        self.h_vision = 800

        self.errors = 0
        self.dead = 0
        self.ready = False
        self.abs_speed = 30 / (self.r ** 0.5)
        self.speed_x = 0
        self.speed_y = 0

    def set_options(self, data):
        data = data[1:-1].split(' ')
        self.name = data[0]
        self.width_window = int(data[1])
        self.height_window = int(data[2])
        self.w_vision = int(data[1])
        self.h_vision = int(data[2])

    def change_speed(self, v):
        if (v[0] == 0) and (v[1] == 0):
            self.speed_x = 0
            self.speed_y = 0
        else:
            lenv = (v[0] ** 2 + v[1] ** 2) ** 0.5
            v = (v[0] / lenv, v[1] / lenv)
            v = (v[0] * self.abs_speed, v[1] * self.abs_speed)
            self.speed_x, self.speed_y = v[0], v[1]

    def update(self):
        # x coordinate
        if self.x - self.r <= 0:
            if self.speed_x >= 0:
                self.x += self.speed_x
        else:
            if self.x + self.r >= WIDTH_ROOM:
                if self.speed_x <= 0:
                    self.x += self.speed_x
            else:
                self.x += self.speed_x

        # y coordinate
        if self.y - self.r <= 0:
            if self.speed_y >= 0:
                self.y += self.speed_y
        else:
            if self.y + self.r >= HEIGHT_ROOM:
                if self.speed_y <= 0:
                    self.y += self.speed_y
            else:
                self.y += self.speed_y

        # abs_speed
        if self.r != 0:
            self.abs_speed = 30 / (self.r ** 0.5)
        else:
            self.abs_speed = 0

        # r
        if self.r >= 100:
            self.r -= self.r / 18000

        # L
        if (self.r >= self.w_vision / 4) or (self.r >= self.h_vision / 4):
            if (self.w_vision <= WIDTH_ROOM) or (self.h_vision <= HEIGHT_ROOM):
                self.L *= 2
                self.w_vision = self.width_window * self.L
                self.h_vision = self.height_window * self.L
        if (self.r < self.w_vision / 8) and (self.r < self.h_vision / 8):
            if self.L > 1:
                self.L = self.L // 2
                self.w_vision = self.width_window * self.Llen
                self.h_vision = self.height_window * self.L


# создание сокета
main_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
main_socket.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
main_socket.bind((server_ip, 12345))
main_socket.setblocking(0)
main_socket.listen(5)

# создание графического окна сервера
pygame.init()
if not work_on_server:
    screen = pygame.display.set_mode((WIDTH_SERVER_WINDOW, HEIGHT_SERVER_WINDOW))
clock = pygame.time.Clock()

# создание стартового набора ботов
players = [Player(None, None,
                  random.randint(0, WIDTH_ROOM),
                  random.randint(0, HEIGHT_ROOM),
                  random.randint(10, 100),
                  str(random.randint(0, 4))
                  )
           for i in range(MOBS_QUANTITY)]

# создание стартового набора корма
microbes = [Microbe(random.randint(0, WIDTH_ROOM),
                    random.randint(0, HEIGHT_ROOM),
                    MICROBE_SIZE,
                    str(random.randint(0, 4)))
            for i in range(MICROBES_QUANTITY)]
tick = -1
server_works = True
while server_works:
    tick += 1
    clock.tick(FPS)
    if tick == 200:
        tick = 0
        # проверим, есть ли желающие войти в игру
        try:
            new_socket, addr = main_socket.accept()
            print('Connected')
            new_socket.setblocking(0)

            spawn = random.choice(microbes)
            new_player = Player(new_socket, addr,
                                spawn.x,
                                spawn.y,
                                START_PLAYER_SIZE,
                                str(random.randint(0, 4)))

            microbes.remove(spawn)
            players.append(new_player)

        except:
            pass
        # дополняем список ботов
        for i in range(MOBS_QUANTITY - len(players)):
            if len(microbes) != 0:
                spawn = random.choice(microbes)
                players.append(Player(None, None,
                                      spawn.x,
                                      spawn.y,
                                      random.randint(10, 100),
                                      str(random.randint(0, 4))
                                      )
                               )
                microbes.remove(spawn)

        # дополняем список корма
        new_microbes = [Microbe(random.randint(0, WIDTH_ROOM),
                                random.randint(0, HEIGHT_ROOM),
                                MICROBE_SIZE,
                                str(random.randint(0, 4)))
                        for i in range(MICROBES_QUANTITY - len(microbes))
                        ]

        microbes = microbes + new_microbes

    # считываем команды игроков
    for player in players:
        if player.conn != None:
            try:
                data = player.conn.recv(1024)
                data = data.decode()

                if data[0] == '!':  # пришло сообщение о готовности к диалогу
                    player.ready = True
                else:
                    if data[0] == '.' and data[-1] == '.':  # пришло имя и размер окна игрока
                        player.set_options(data)
                        player.conn.send((str(START_PLAYER_SIZE) + ' ' + player.colour).encode())
                    else:  # пришел курсор
                        data = find(data)
                        player.change_speed(data)
            except:
                pass
        else:
            if tick == 100:
                data = [random.randint(-100, 100), random.randint(-100, 100)]
                player.change_speed(data)

        player.update()

    # определим, что видит каждый игрок
    visible_balls = [[] for i in range(len(players))]

    for i in range(len(players)):
        # каких микробов видит i игрок
        for k in range(len(microbes)):
            dist_x = microbes[k].x - players[i].x
            dist_y = microbes[k].y - players[i].y

            # i видит k
            if (
                    (abs(dist_x) <= (players[i].w_vision) // 2 + microbes[k].r)
                    and
                    (abs(dist_y) <= (players[i].h_vision) // 2 + microbes[k].r)
            ):
                # i может съесть k
                if (dist_x ** 2 + dist_y ** 2) ** 0.5 <= players[i].r:
                    players[i].r = new_r(players[i].r, microbes[k].r)
                    microbes[k].r = 0

                if (players[i].conn != None) and (microbes[k].r != 0):
                    # подготовим данные к добавлению в список видимых шаров
                    x_ = str(round(dist_x / players[i].L))
                    y_ = str(round(dist_y / players[i].L))
                    r_ = str(round(microbes[k].r / players[i].L))
                    c_ = microbes[k].colour

                    visible_balls[i].append(x_ + ' ' + y_ + ' ' + r_ + ' ' + c_)

        for j in range(i + 1, len(players)):
            # рассматриваем пару i и j игрока
            dist_x = players[j].x - players[i].x
            dist_y = players[j].y - players[i].y

            # i видит j
            if (
                    (abs(dist_x) <= (players[i].w_vision) // 2 + players[j].r)
                    and
                    (abs(dist_y) <= (players[i].h_vision) // 2 + players[j].r)
            ):
                # i может съесть j
                if ((dist_x ** 2 + dist_y ** 2) ** 0.5 <= players[i].r and
                        players[i].r > 1.1 * players[j].r):
                    players[i].r = new_r(players[i].r, players[j].r)
                    players[j].r, players[j].speed_x, players[j].speed_y = 0, 0, 0

                if players[i].conn != None:
                    # подготовим данные к добавлению в список видимых шаров
                    x_ = str(round(dist_x / players[i].L))
                    y_ = str(round(dist_y / players[i].L))
                    r_ = str(round(players[j].r / players[i].L))
                    c_ = players[j].colour
                    n_ = players[j].name

                    if players[j].r >= 30 * players[i].L:
                        visible_balls[i].append(x_ + ' ' + y_ + ' ' + r_ + ' ' + c_ + ' ' + n_)
                    else:
                        visible_balls[i].append(x_ + ' ' + y_ + ' ' + r_ + ' ' + c_)

            # j видит i
            if (
                    (abs(dist_x) <= (players[j].w_vision) // 2 + players[i].r)
                    and
                    (abs(dist_y) <= (players[j].h_vision) // 2 + players[i].r)
            ):
                # j может съесть i
                if ((dist_x ** 2 + dist_y ** 2) ** 0.5 <= players[j].r and
                        players[j].r > 1.1 * players[i].r):
                    players[j].r = new_r(players[j].r, players[i].r)
                    players[i].r, players[i].speed_x, players[i].speed_y = 0, 0, 0

                if players[j].conn != None:
                    # подготовим данные к добавлению в список видимых шаров
                    x_ = str(round(-dist_x / players[j].L))
                    y_ = str(round(-dist_y / players[j].L))
                    r_ = str(round(players[i].r / players[j].L))
                    c_ = players[i].colour
                    n_ = players[i].name

                    if players[i].r >= 30 * players[j].L:
                        visible_balls[j].append(x_ + ' ' + y_ + ' ' + r_ + ' ' + c_ + ' ' + n_)
                    else:
                        visible_balls[j].append(x_ + ' ' + y_ + ' ' + r_ + ' ' + c_)

    # формируем ответ каждому игроку
    otvets = ['' for i in range(len(players))]
    for i in range(len(players)):
        r_ = str(round(players[i].r / players[i].L))
        x_ = str(round(players[i].x / players[i].L))
        y_ = str(round(players[i].y / players[i].L))
        L_ = str(players[i].L)

        visible_balls[i] = [r_ + ' ' + x_ + ' ' + y_ + ' ' + L_] + visible_balls[i]
        otvets[i] = '<' + (','.join(visible_balls[i])) + '>'

    # отправляем новое состояние игрового поля
    for i in range(len(players)):
        if (players[i].conn != None) and (players[i].ready):
            try:
                players[i].conn.send(otvets[i].encode())
                players[i].errors = 0
            except:
                players[i].errors += 1

    # чистим список от отвалившихся игроков
    for player in players:
        if player.r == 0:
            if player.conn != None:
                player.dead += 1
            else:
                player.dead += 300

        if (player.errors == 500) or (player.dead == 300):
            if player.conn != None:
                player.conn.close()
            players.remove(player)

    # чистим список от съеденного корма
    for m in microbes:
        if m.r == 0: microbes.remove(m)

    if not work_on_server:
        # нарисуем состояние комнаты
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                server_works = False

        screen.fill('BLACK')
        for player in players:
            x = round(player.x * WIDTH_SERVER_WINDOW / WIDTH_ROOM)
            y = round(player.y * HEIGHT_SERVER_WINDOW / HEIGHT_ROOM)
            r = round(player.r * WIDTH_SERVER_WINDOW / WIDTH_ROOM)
            c = colours[player.colour]

            pygame.draw.circle(screen, c, (x, y), r)
        pygame.display.update()

pygame.quit()
main_socket.close()
