#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <math.h>
#include <signal.h>
#include <time.h>
#include <unistd.h>
#include "libfreenect.h"

#if defined(__APPLE__)
#include <GLUT/glut.h>
#else
#include <GL/glut.h>
#endif

#define WIDTH 640
#define HEIGHT 480

const float MIN_DEPTH = 859.0f;
const float MAX_DEPTH = 949.0f;

freenect_context *f_ctx;
freenect_device  *f_dev;
pthread_t freenect_thread;
int die = 0;

uint16_t depth[WIDTH * HEIGHT];
uint8_t depth_rgb[WIDTH * HEIGHT * 3];
GLuint tex_id;
pthread_mutex_t depth_mutex = PTHREAD_MUTEX_INITIALIZER;
volatile int depth_rgb_updated = 0;

const char* note_files[] = {"do.mp3", "re.mp3", "mi.mp3", "fa.mp3", "sol.mp3", "la.mp3", "si.mp3"};
const char* cor_names[] = {"violeta", "anil", "azul", "verde", "amarelo", "laranja", "vermelho"};
int cor_quadrante_predominante[4] = {0};
int nota_alvo[4] = {0};
int countdown = 40;
int nota_tocada_no_step = 0;
int nota_tocada_eval = 0;
int esperando_comeco = 1; // 1 = aguardando espaÃ§o, 0 = jogando normalmente


typedef enum { STATE_INTRO, STATE_PLAYING, STATE_EVAL, STATE_FEEDBACK } GameState;
static GameState game_state = STATE_INTRO;

int intro_step = 0, eval_step = 0, feedback_ok = 0, feedback_flash_count = 0;
int flash_on = 1;
int phase_ms_accum = 0, last_idle_ms = 0;

const int FLASH_MS_ON = 600;
const int FLASH_MS_OFF = 300;
const int FEEDBACK_FLASHES = 6;
const float ALPHA_OVERLAY_TELA = 0.4f;

void sortear_alvos() {
    for (int q = 0; q < 4; q++) {
        nota_alvo[q] = rand() % 7;
    }
}

void tocar_nota_idx(int idx) {
    if (idx < 0 || idx > 6) return;
    char cmd[128];
    snprintf(cmd, sizeof(cmd), "afplay notas/%s &", note_files[idx]);
    system(cmd);
}
void get_rgb_from_index(int idx, float *r, float *g, float *b) {
    switch (idx) {
        case 0: *r = 148/255.0f; *g = 0.0f;       *b = 211/255.0f; break; // violeta
        case 1: *r = 75/255.0f;  *g = 0.0f;       *b = 130/255.0f; break; // anil
        case 2: *r = 0.0f;       *g = 0.0f;       *b = 1.0f;       break; // azul
        case 3: *r = 0.0f;       *g = 1.0f;       *b = 0.0f;       break; // verde
        case 4: *r = 1.0f;       *g = 1.0f;       *b = 0.0f;       break; // amarelo
        case 5: *r = 1.0f;       *g = 165/255.0f; *b = 0.0f;       break; // laranja
        case 6: *r = 1.0f;       *g = 0.0f;       *b = 0.0f;       break; // vermelho
        default: *r = *g = *b = 0.0f; break;
    }
}

void draw_text(const char* text, float x, float y, void* font) {
    glColor3f(0.0f, 0.0f, 0.0f);
    glRasterPos2f(x, y);
    while (*text) {
        glutBitmapCharacter(font, *text);
        text++;
    }
}

void draw_quadrant_text(int q, const char* label) {
    float xmin = (q % 2 == 0) ? -1.0f : 0.0f;
    float xmax = (q % 2 == 0) ?  0.0f : 1.0f;
    float ymin = (q < 2) ? 0.0f : -1.0f;
    float ymax = (q < 2) ? 1.0f : 0.0f;

    glMatrixMode(GL_MODELVIEW);
    glPushMatrix();
    glLoadIdentity();

    glDisable(GL_TEXTURE_2D);
    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);

    glColor4f(1.0f, 1.0f, 1.0f, 1.0f);
    glBegin(GL_QUADS);
        glVertex2f(xmin, ymin);
        glVertex2f(xmax, ymin);
        glVertex2f(xmax, ymax);
        glVertex2f(xmin, ymax);
    glEnd();

    glPopMatrix();

    float center_x = (xmin + xmax) / 2.0f;
    float center_y = (ymin + ymax) / 2.0f;
    float scale = 0.0010f;

    float total_width = 0;
    for (const char* c = label; *c; c++) {
        total_width += glutStrokeWidth(GLUT_STROKE_ROMAN, *c);
    }
    float offset_x = total_width * scale * 0.5f;

    float bold_offset = 0.0020f;

    for (float dx = -bold_offset; dx <= bold_offset; dx += bold_offset) {
        for (float dy = -bold_offset; dy <= bold_offset; dy += bold_offset) {
            glPushMatrix();
            glLoadIdentity();
            glTranslatef(center_x - offset_x + dx, center_y - 0.02f + dy, 0.0f);
            glScalef(scale, scale, 1.0f);
            glColor3f(0.0f, 0.0f, 0.0f);
            for (const char* c = label; *c; c++) {
                glutStrokeCharacter(GLUT_STROKE_ROMAN, *c);
            }
            glPopMatrix();
        }
    }
}
void draw_fullscreen_overlay(float r, float g, float b, float alpha) {
    glColor4f(r, g, b, alpha);
    glBegin(GL_QUADS);
        glVertex2f(-1, -1); glVertex2f(1, -1);
        glVertex2f(1, 1); glVertex2f(-1, 1);
    glEnd();
}

void DrawQuadrants() {
    glColor3f(1.0f, 1.0f, 1.0f);
    glLineWidth(6.0f);
    glBegin(GL_LINES);
        glVertex2f(0.0f, -1.0f); glVertex2f(0.0f, 1.0f);
        glVertex2f(-1.0f, 0.0f); glVertex2f(1.0f, 0.0f);
    glEnd();
}

void drawCountdown() {
    char texto[8];
    snprintf(texto, sizeof(texto), "%02d", countdown);
    glDisable(GL_TEXTURE_2D);
    float cx = 0.0f, cy = 0.0f, r = 0.05f;
    int num_segments = 40;
    float aspect = (float)WIDTH / HEIGHT;
    glColor3f(1.0f, 1.0f, 1.0f);
    glBegin(GL_POLYGON);
    for (int i = 0; i < num_segments; i++) {
        float theta = 2.0f * 3.1415926f * i / num_segments;
        float x = r * cosf(theta) / aspect;
        float y = r * sinf(theta);
        glVertex2f(x + cx, y + cy);
    }
    glEnd();
    glColor3f(0.0f, 0.0f, 0.0f);
    glRasterPos2f(-0.030f, -0.030f);
    void* fonte = GLUT_BITMAP_TIMES_ROMAN_24;
    for (int i = 0; texto[i] != '\0'; i++)
        glutBitmapCharacter(fonte, texto[i]);
    glEnable(GL_TEXTURE_2D);
}

void display() {
    glClear(GL_COLOR_BUFFER_BIT);
    glEnable(GL_TEXTURE_2D);
    glBindTexture(GL_TEXTURE_2D, tex_id);
    glColor3f(1.0f, 1.0f, 1.0f);

    pthread_mutex_lock(&depth_mutex);
    if (depth_rgb_updated) {
        glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, WIDTH, HEIGHT, GL_RGB, GL_UNSIGNED_BYTE, depth_rgb);
        depth_rgb_updated = 0;
    }
    pthread_mutex_unlock(&depth_mutex);

    glBegin(GL_QUADS);
        glTexCoord2f(0,1); glVertex2f(-1,-1);
        glTexCoord2f(1,1); glVertex2f(1,-1);
        glTexCoord2f(1,0); glVertex2f(1,1);
        glTexCoord2f(0,0); glVertex2f(-1,1);
    glEnd();

    glDisable(GL_TEXTURE_2D);
    DrawQuadrants();
    drawCountdown();

    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);

    if (game_state == STATE_INTRO && intro_step < 4) {
        draw_quadrant_text(intro_step, cor_names[nota_alvo[intro_step]]);
    } else if (game_state == STATE_EVAL && eval_step < 4) {
        draw_quadrant_text(eval_step, cor_names[cor_quadrante_predominante[eval_step]]);
    } else if (game_state == STATE_FEEDBACK && flash_on) {
        float r = feedback_ok ? 0.0f : 1.0f;
        float g = feedback_ok ? 1.0f : 0.0f;
        draw_fullscreen_overlay(r, g, 0.0f, ALPHA_OVERLAY_TELA);
    }

    glDisable(GL_BLEND);
    glutSwapBuffers();
}
void calcular_predominancia_frame_atual() {
    int counts[4][7] = {0};
    for (int y = 0; y < HEIGHT; y++) {
        for (int x = 0; x < WIDTH; x++) {
            int i = y * WIDTH + x;
            int q = (y < HEIGHT / 2 ? 0 : 2) + (x >= WIDTH / 2 ? 1 : 0);
            uint8_t r = depth_rgb[3*i];
            uint8_t g = depth_rgb[3*i+1];
            uint8_t b = depth_rgb[3*i+2];
            int idx = -1;
            if (r == 255 && g == 0 && b == 0) idx = 6;
            else if (r == 255 && g == 165 && b == 0) idx = 5;
            else if (r == 255 && g == 255 && b == 0) idx = 4;
            else if (r == 0 && g == 255 && b == 0) idx = 3;
            else if (r == 0 && g == 0 && b == 255) idx = 2;
            else if (r == 75 && g == 0 && b == 130) idx = 1;
            else if (r == 148 && g == 0 && b == 211) idx = 0;
            if (idx >= 0) counts[q][idx]++;
        }
    }
    for (int q = 0; q < 4; q++) {
        int max = 0;
        for (int i = 1; i < 7; i++)
            if (counts[q][i] > counts[q][max]) max = i;
        cor_quadrante_predominante[q] = max;
    }
}

void idle() {
    int now = glutGet(GLUT_ELAPSED_TIME);
    int delta = now - last_idle_ms;
    last_idle_ms = now;

    // Enquanto estiver esperando a tecla ESPAÃ‡O, nÃ£o executa a lÃ³gica do jogo
    if (esperando_comeco) {
        glutPostRedisplay();
        return;
    }

    phase_ms_accum += delta;

    switch (game_state) {
        case STATE_INTRO:
            if (!nota_tocada_no_step) {
                tocar_nota_idx(nota_alvo[intro_step]);
                nota_tocada_no_step = 1;
            }
            if (phase_ms_accum > 1500) {
                intro_step++;
                phase_ms_accum = 0;
                nota_tocada_no_step = 0;
                if (intro_step >= 4) {
                    game_state = STATE_PLAYING;
                    countdown = 40;
                    intro_step = 0;
                }
            }
            break;

        case STATE_PLAYING: {
            static int t = 0;
            t += delta;
            if (t >= 1000) {
                countdown--;
                t = 0;
            }
            if (countdown <= 0) {
                calcular_predominancia_frame_atual();
                game_state = STATE_EVAL;
                eval_step = 0;
                phase_ms_accum = 0;
            }
            break;
        }

        case STATE_EVAL:
            if (!nota_tocada_eval) {
                tocar_nota_idx(cor_quadrante_predominante[eval_step]);
                nota_tocada_eval = 1;
            }
            if (phase_ms_accum > 1500) {
                eval_step++;
                phase_ms_accum = 0;
                nota_tocada_eval = 0;
                if (eval_step >= 4) {
                    feedback_ok = 1;
                    for (int q = 0; q < 4; q++) {
                        if (cor_quadrante_predominante[q] != nota_alvo[q])
                            feedback_ok = 0;
                    }
                    game_state = STATE_FEEDBACK;
                    feedback_flash_count = 0;
                    flash_on = 1;
                    phase_ms_accum = 0;
                }
            }
            break;

        case STATE_FEEDBACK:
            if (phase_ms_accum > (flash_on ? FLASH_MS_ON : FLASH_MS_OFF)) {
                flash_on = !flash_on;
                phase_ms_accum = 0;
                if (!flash_on) feedback_flash_count++;
                if (feedback_flash_count >= FEEDBACK_FLASHES) {
                    sortear_alvos();
                    esperando_comeco = 1;  // Espera novamente a tecla espaÃ§o para prÃ³xima partida
                    game_state = STATE_INTRO;
                    intro_step = 0;
                    flash_on = 1;
                    phase_ms_accum = 0;
                }
            }
            break;
    }

    glutPostRedisplay();
}


void depth_cb(freenect_device *dev, void *v_depth, uint32_t timestamp) {
    uint16_t *depth_data = (uint16_t *)v_depth;
    pthread_mutex_lock(&depth_mutex);
    for (int i = 0; i < WIDTH * HEIGHT; i++) {
        uint16_t d = depth_data[i];
        float norm = 1.0f - ((float)d - MIN_DEPTH) / (MAX_DEPTH - MIN_DEPTH);
        if (norm < 0.0f) norm = 0.0f;
        if (norm > 1.0f) norm = 1.0f;

        float r = 0, g = 0, b = 0;

        // Agora a escala estÃ¡ invertida corretamente (mais prÃ³ximo = violeta)
        if (norm < 0.16f)        { r = 148/255.0f; g = 0.0f;       b = 211/255.0f; } // violeta
        else if (norm < 0.33f)   { r = 75/255.0f;  g = 0.0f;       b = 130/255.0f; } // anil
        else if (norm < 0.50f)   { r = 0.0f;       g = 0.0f;       b = 1.0f;       } // azul
        else if (norm < 0.66f)   { r = 0.0f;       g = 1.0f;       b = 0.0f;       } // verde
        else if (norm < 0.83f)   { r = 1.0f;       g = 1.0f;       b = 0.0f;       } // amarelo
        else if (norm < 0.97f)   { r = 1.0f;       g = 165/255.0f; b = 0.0f;       } // laranja
        else                     { r = 1.0f;       g = 0.0f;       b = 0.0f;       } // vermelho

        depth_rgb[3 * i + 0] = (uint8_t)(r * 255);
        depth_rgb[3 * i + 1] = (uint8_t)(g * 255);
        depth_rgb[3 * i + 2] = (uint8_t)(b * 255);
    }
    depth_rgb_updated = 1;
    pthread_mutex_unlock(&depth_mutex);
}


void *freenect_threadfunc(void *arg) {
    while (!die && freenect_process_events(f_ctx) >= 0);
    return NULL;
}

void keyPressed(unsigned char key, int x, int y) {
    if (key == 27) {
        die = 1;
        pthread_join(freenect_thread, NULL);
        freenect_stop_depth(f_dev);
        freenect_close_device(f_dev);
        freenect_shutdown(f_ctx);
        exit(0);
    } else if (key == 32 && esperando_comeco) { // tecla ESPAÃ‡O
        esperando_comeco = 0;
        game_state = STATE_INTRO;
        intro_step = 0;
        phase_ms_accum = 0;
        nota_tocada_no_step = 0;
    }
}

int main(int argc, char **argv) {
    srand(time(NULL));
    if (freenect_init(&f_ctx, NULL) < 0) return 1;
    if (freenect_open_device(f_ctx, &f_dev, 0) < 0) return 1;

    glutInit(&argc, argv);
    glutInitDisplayMode(GLUT_RGB | GLUT_DOUBLE);
    glutInitWindowSize(WIDTH, HEIGHT);
    glutCreateWindow("Genius da Areia");
    glutFullScreen();

    glutDisplayFunc(display);
    glutIdleFunc(idle);
    glutKeyboardFunc(keyPressed);

    glEnable(GL_TEXTURE_2D);
    glGenTextures(1, &tex_id);
    glBindTexture(GL_TEXTURE_2D, tex_id);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, WIDTH, HEIGHT, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

    freenect_set_depth_callback(f_dev, depth_cb);
    freenect_set_depth_mode(f_dev, freenect_find_depth_mode(FREENECT_RESOLUTION_MEDIUM, FREENECT_DEPTH_MM));
    freenect_start_depth(f_dev);

    pthread_create(&freenect_thread, NULL, freenect_threadfunc, NULL);
    sortear_alvos();
    last_idle_ms = glutGet(GLUT_ELAPSED_TIME);
    glutMainLoop();
    return 0;
}
