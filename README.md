import time
import pygame
import numpy as np
from pygame.locals import *
import random
import os
os.environ['LIBGL_ALWAYS_SOFTWARE'] = '1'
pygame.init()
pygame.mixer.init()
width, height = 800, 600
screen = pygame.display.set_mode((width, height))
pygame.display.set_caption("Minecraft The 3D Block Game")
clock = pygame.time.Clock()

GRASS_GREEN = (34, 139, 34)
DIRT_BROWN = (139, 69, 19)
WHITE = (255, 255, 255)
GRAY = (169, 169, 169)
WOOD_BROWN = (139, 69, 19)
LEAF_GREEN = (34, 139, 34)
MOB_COLOR = (255, 0, 0)
SHADOW_COLOR = (0, 0, 0, 128)  
SKY_BLUE = (135, 206, 235)
NIGHT_BLUE = (25, 25, 112)

vertices = np.array([[-1, -1, -1], [1, -1, -1], [1, 1, -1], [-1, 1, -1],
                     [-1, -1, 1], [1, -1, 1], [1, 1, 1], [-1, 1, 1]])
faces = [(0, 1, 2, 3), (4, 5, 6, 7), (0, 1, 5, 4), (2, 3, 7, 6), (1, 2, 6, 5), (4, 0, 3, 7)]

block_positions = {}
mob_positions = []
mob_update_rate = 30
mob_update_counter = 0
render_distance = 20

pitch = 0
yaw = 0

day_time = True
day_night_cycle_speed = 0.0
camera_shake_intensity = 0

def add_block(x, y, z, color=None):
    block_positions[(x, y, z)] = color if color else DIRT_BROWN

def remove_block(x, y, z):
    if (x, y, z) in block_positions:
        del block_positions[(x, y, z)]

def project(x, y, z, pitch, yaw):
    fov = 256
    distance = 4
    cos_pitch = np.cos(pitch)
    sin_pitch = np.sin(pitch)
    cos_yaw = np.cos(yaw)
    sin_yaw = np.sin(yaw)

    x, y, z = (
        x * cos_yaw - z * sin_yaw,
        y * cos_pitch + z * sin_pitch,
        z * cos_pitch - y * sin_pitch
    )
    factor = fov / (z + distance + 0.1)
    x = x * factor + width / 2 + camera_shake_intensity
    y = -y * factor + height / 2 + camera_shake_intensity
    return (int(x), int(y), z)

def draw_shadow(pos, offset, pitch, yaw):
    shadow_offset = np.array([0, 0, 0.1]) 
    rotated_vertices = vertices + pos + offset + shadow_offset
    projected_vertices = [project(*v, pitch, yaw) for v in rotated_vertices]

    face_order = sorted(faces, key=lambda f: -sum(projected_vertices[v][2] for v in f) / len(f))

    for face in face_order:
        pygame.draw.polygon(screen, SHADOW_COLOR, [projected_vertices[v][:2] for v in face], 4)  

def draw_block(pos, offset, color, pitch, yaw):
    rotated_vertices = vertices + pos + offset
    projected_vertices = [project(*v, pitch, yaw) for v in rotated_vertices]

    face_order = sorted(faces, key=lambda f: -sum(projected_vertices[v][2] for v in f) / len(f))

    for face in face_order:
        face_color = color if color is not None else DIRT_BROWN
        pygame.draw.polygon(screen, face_color, [projected_vertices[v][:2] for v in face])
        pygame.draw.polygon(screen, WHITE, [projected_vertices[v][:2] for v in face], 1)

def add_tree(base_x, base_y):
    for z in range(3):
        add_block(base_x, base_y, z, WOOD_BROWN)
    leaf_positions = [(base_x, base_y, 3), (base_x-1, base_y, 3), (base_x+1, base_y, 3), 
                      (base_x, base_y-1, 3), (base_x, base_y+1, 3)]
    for pos in leaf_positions:
        add_block(*pos, LEAF_GREEN)

def add_mountain(base_x, base_y):
    for z in range(5):
        for x in range(base_x - z, base_x + z + 1):
            for y in range(base_y - z, base_y + z + 1):
                add_block(x, y, z)

def add_house(base_x, base_y):
    for x in range(base_x, base_x + 5):
        for y in range(base_y, base_y + 5):
            add_block(x, y, 0, GRAY)
            add_block(x, y, 1, GRAY)
            if x == base_x or x == base_x + 4 or y == base_y or y == base_y + 4:
                add_block(x, y, 2, GRAY)
                add_block(x, y, 3, GRAY)
    for z in range(4, 6):
        for x in range(base_x + 1, base_x + 4):
            for y in range(base_y + 1, base_y + 4):
                add_block(x, y, z, GRAY)

def add_mob_shape(position, shape, color):
    for offset in shape:
        pos = [position[0] + offset[0], position[1] + offset[1], position[2] + offset[2]]
        block_positions[tuple(pos)] = color

def create_mobs():
    mob_shapes = [
        [(0, 0, 0), (1, 0, 0), (0, 1, 0)],
        [(0, 0, 0), (0, 1, 0), (0, 2, 0)],
        [(0, 0, 0), (1, 0, 0), (1, 1, 0)],
        [(0, 0, 0), (0, 1, 0), (1, 1, 0)],
        [(0, 0, 0), (1, 0, 0), (1, 1, 0), (1, 0, 1)]
    ]

    for _ in range(10):
        shape = random.choice(mob_shapes)
        color = MOB_COLOR
        position = [random.randint(-40, 40), random.randint(-40, 40), 0]
        mob_positions.append(position)
        add_mob_shape(position, shape, color)


for x in range(-100, 101):
    for y in range(-100, 101):
        add_block(x, y, 0)

for _ in range(30):
    add_tree(random.randint(-100, 101), random.randint(-100, 101))
for _ in range(10):
    add_mountain(random.randint(-60, 40), random.randint(-40, 40))

add_house(-10, -10)
add_house(20, 20)

create_mobs()

running = True
player_pos = [0, 0, 0]

while running:
    for event in pygame.event.get():
        if event.type == QUIT:
            running = False
        elif event.type == KEYDOWN:
            if event.key == K_a:
                add_block(player_pos[0], player_pos[1], player_pos[2])
            elif event.key == K_r:
                remove_block(player_pos[0], player_pos[1], player_pos[2])
            elif event.key == K_LEFT:
                player_pos[0] -= 1
            elif event.key == K_RIGHT:
                player_pos[0] += 1
            elif event.key == K_UP:
                player_pos[1] += 1
            elif event.key == K_DOWN:
                player_pos[1] -= 1
            elif event.key == K_w:
                player_pos[2] += 1
            elif event.key == K_s:
                player_pos[2] -= 1
            elif event.key == K_SPACE:
                remove_block(player_pos[0], player_pos[1], player_pos[2])
            elif event.key == K_q:
                pitch -= 0.1  
            elif event.key == K_e:
                pitch += 0.1 
            elif event.key == K_z:
                yaw -= 0.1   
            elif event.key == K_c:
                yaw += 0.1  

    screen.fill(SKY_BLUE if day_time else NIGHT_BLUE)

    offset = np.array([-player_pos[0], -player_pos[1], -player_pos[2]])

    #
    for pos, color in block_positions.items():
        if abs(pos[0] - player_pos[0]) <= render_distance and abs(pos[1] - player_pos[1]) <= render_distance and abs(pos[2] - player_pos[2]) <= render_distance:
            draw_shadow(np.array(pos), offset, pitch, yaw)  # Draw shadow first
            draw_block(np.array(pos), offset, color, pitch, yaw)  # Draw the block

    if mob_update_counter >= mob_update_rate:
        mob_update_counter = 0
        for mob in mob_positions:
            mob[0] += random.choice([-1, 0, 1])
            mob[1] += random.choice([-1, 0, 1])
    else:
        mob_update_counter += 1

    for mob in mob_positions:
        draw_shadow(np.array(mob), offset, pitch, yaw)  
        draw_block(np.array(mob), offset, MOB_COLOR, pitch, yaw) 

    pygame.display.flip()
    clock.tick(60)

pygame.quit()
