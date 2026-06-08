#include <stdio.h>
#include <stdlib.h>
#include <math.h>

#define MAX_WIDTH 80
#define MAX_HEIGHT 25
#define MAX_OBJECTS 50

typedef enum {
    SHAPE_LINE,
    SHAPE_RECTANGLE,
    SHAPE_CIRCLE,
    SHAPE_TRIANGLE
} ShapeType;

typedef struct {
    int id;
    ShapeType type;
    int x1, y1;
    int x2, y2;
    int x3, y3;
    int radius;
    char stroke;
    char fill;
} Object;

static char canvas[MAX_HEIGHT][MAX_WIDTH];
static Object objects[MAX_OBJECTS];
static int object_count = 0;

void init_canvas(void) {
    for (int y = 0; y < MAX_HEIGHT; ++y) {
        for (int x = 0; x < MAX_WIDTH; ++x) {
            canvas[y][x] = ' ';
        }
    }
}

void display_canvas(void) {
    printf("\n");
    for (int x = 0; x < MAX_WIDTH + 2; ++x) putchar('-');
    putchar('\n');
    for (int y = 0; y < MAX_HEIGHT; ++y) {
        putchar('|');
        for (int x = 0; x < MAX_WIDTH; ++x) {
            putchar(canvas[y][x]);
        }
        putchar('|');
        putchar('\n');
    }
    for (int x = 0; x < MAX_WIDTH + 2; ++x) putchar('-');
    putchar('\n');
}

void set_pixel(int x, int y, char ch) {
    if (x >= 0 && x < MAX_WIDTH && y >= 0 && y < MAX_HEIGHT) {
        canvas[y][x] = ch;
    }
}

void draw_line(const Object *obj) {
    int x0 = obj->x1;
    int y0 = obj->y1;
    int x1 = obj->x2;
    int y1 = obj->y2;
    int dx = abs(x1 - x0);
    int dy = abs(y1 - y0);
    int sx = x0 < x1 ? 1 : -1;
    int sy = y0 < y1 ? 1 : -1;
    int err = dx - dy;

    while (1) {
        set_pixel(x0, y0, obj->stroke);
        if (x0 == x1 && y0 == y1) break;
        int e2 = err * 2;
        if (e2 > -dy) {
            err -= dy;
            x0 += sx;
        }
        if (e2 < dx) {
            err += dx;
            y0 += sy;
        }
    }
}

void draw_rectangle(const Object *obj) {
    int x0 = obj->x1 < obj->x2 ? obj->x1 : obj->x2;
    int x1 = obj->x1 < obj->x2 ? obj->x2 : obj->x1;
    int y0 = obj->y1 < obj->y2 ? obj->y1 : obj->y2;
    int y1 = obj->y1 < obj->y2 ? obj->y2 : obj->y1;

    for (int x = x0; x <= x1; ++x) {
        set_pixel(x, y0, obj->stroke);
        set_pixel(x, y1, obj->stroke);
    }
    for (int y = y0; y <= y1; ++y) {
        set_pixel(x0, y, obj->stroke);
        set_pixel(x1, y, obj->stroke);
    }
    for (int y = y0 + 1; y < y1; ++y) {
        for (int x = x0 + 1; x < x1; ++x) {
            set_pixel(x, y, obj->fill);
        }
    }
}

void draw_circle(const Object *obj) {
    int cx = obj->x1;
    int cy = obj->y1;
    int r = obj->radius;
    if (r <= 0) return;

    for (int y = cy - r; y <= cy + r; ++y) {
        for (int x = cx - r; x <= cx + r; ++x) {
            int dx = x - cx;
            int dy = y - cy;
            double distance = sqrt((double)dx * dx + (double)dy * dy);
            if (distance <= r + 0.4 && distance >= r - 0.4) {
                set_pixel(x, y, obj->stroke);
            } else if (distance < r - 0.4) {
                set_pixel(x, y, obj->fill);
            }
        }
    }
}

static int point_in_triangle(int px, int py, int x0, int y0, int x1, int y1, int x2, int y2) {
    int dX = px - x2;
    int dY = py - y2;
    int dX21 = x2 - x1;
    int dY12 = y1 - y2;
    int D = dY12 * (x0 - x2) + dX21 * (y0 - y2);
    int s = dY12 * dX + dX21 * dY;
    int t = (y2 - y0) * dX + (x0 - x2) * dY;
    if (D < 0) return s <= 0 && t <= 0 && s + t >= D;
    return s >= 0 && t >= 0 && s + t <= D;
}

void draw_triangle(const Object *obj) {
    draw_line(obj);
    Object edge = *obj;
    edge.x1 = obj->x2;
    edge.y1 = obj->y2;
    edge.x2 = obj->x3;
    edge.y2 = obj->y3;
    draw_line(&edge);
    edge.x1 = obj->x3;
    edge.y1 = obj->y3;
    edge.x2 = obj->x1;
    edge.y2 = obj->y1;
    draw_line(&edge);

    int minx = obj->x1;
    int maxx = obj->x1;
    int miny = obj->y1;
    int maxy = obj->y1;
    int xs[3] = {obj->x1, obj->x2, obj->x3};
    int ys[3] = {obj->y1, obj->y2, obj->y3};
    for (int i = 1; i < 3; ++i) {
        if (xs[i] < minx) minx = xs[i];
        if (xs[i] > maxx) maxx = xs[i];
        if (ys[i] < miny) miny = ys[i];
        if (ys[i] > maxy) maxy = ys[i];
    }
    for (int y = miny; y <= maxy; ++y) {
        for (int x = minx; x <= maxx; ++x) {
            if (point_in_triangle(x, y, obj->x1, obj->y1, obj->x2, obj->y2, obj->x3, obj->y3)) {
                set_pixel(x, y, obj->fill);
            }
        }
    }
}

void redraw_canvas(void) {
    init_canvas();
    for (int i = 0; i < object_count; ++i) {
        const Object *obj = &objects[i];
        switch (obj->type) {
            case SHAPE_LINE: draw_line(obj); break;
            case SHAPE_RECTANGLE: draw_rectangle(obj); break;
            case SHAPE_CIRCLE: draw_circle(obj); break;
            case SHAPE_TRIANGLE: draw_triangle(obj); break;
        }
    }
}

int add_object(const Object *obj) {
    if (object_count >= MAX_OBJECTS) return 0;
    objects[object_count++] = *obj;
    redraw_canvas();
    return 1;
}

int remove_object(int id) {
    for (int i = 0; i < object_count; ++i) {
        if (objects[i].id == id) {
            for (int j = i; j < object_count - 1; ++j) {
                objects[j] = objects[j + 1];
         scanf("%d %d %d %d", &obj->x1, &obj->y1, &obj->x2, &obj->y2);
            break;
