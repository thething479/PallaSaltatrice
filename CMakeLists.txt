#include "raylib.h"
#include <vector>
#include <algorithm>
#include <cmath>

const int LARGHEZZA = 800;
const int ALTEZZA = 450;

// Il sole, disegnato volutamente male: raggi storti, cerchi sfasati, faccia storta
void disegnaSole() {
    Vector2 c = {680.0f, 80.0f};
    const float angoli[] = {0, 38, 77, 110, 150, 185, 225, 262, 300, 335};
    const float lunghezze[] = {62, 55, 70, 52, 66, 58, 72, 50, 64, 56};
    for (int i = 0; i < 10; i++) {
        float a = angoli[i] * DEG2RAD;
        Vector2 inizio = {c.x + cosf(a) * 42, c.y + sinf(a) * 42};
        Vector2 fine = {c.x + cosf(a) * lunghezze[i], c.y + sinf(a) * lunghezze[i]};
        DrawLineEx(inizio, fine, 4, {255, 200, 0, 255});
    }
    DrawCircleV(c, 38, {255, 230, 40, 255});
    DrawCircleV({c.x + 4, c.y + 3}, 34, {255, 240, 90, 255});  // secondo cerchio fuori centro
    DrawCircle(668, 72, 4, BLACK);                              // occhi a altezze diverse
    DrawCircle(691, 77, 5, BLACK);
    DrawLineEx({664, 95}, {678, 102}, 3, BLACK);                // bocca storta
    DrawLineEx({678, 102}, {696, 92}, 3, BLACK);
}

// Tre colline verdi, tagliate in basso dal terreno che viene disegnato dopo
void disegnaColline(float terreno) {
    Color scuro = {70, 150, 70, 255};
    Color chiaro = {105, 190, 95, 255};
    DrawCircle(120, (int)terreno + 70, 170, scuro);
    DrawCircle(430, (int)terreno + 110, 230, chiaro);
    DrawCircle(740, (int)terreno + 60, 150, scuro);
}

// Un sasso con la faccia arrabbiata
void disegnaSasso(const Rectangle& o) {
    DrawRectangleRounded(o, 0.6f, 8, {120, 120, 125, 255});
    float ex1 = o.x + o.width * 0.3f;
    float ex2 = o.x + o.width * 0.7f;
    float ey = o.y + o.height * 0.38f;
    DrawCircle((int)ex1, (int)ey, 6, WHITE);                    // occhi
    DrawCircle((int)ex2, (int)ey, 6, WHITE);
    DrawCircle((int)ex1 - 1, (int)ey + 1, 3, BLACK);            // pupille
    DrawCircle((int)ex2 - 1, (int)ey + 1, 3, BLACK);
    DrawLineEx({ex1 - 7, ey - 9}, {ex1 + 5, ey - 5}, 3, BLACK);  // sopracciglia
    DrawLineEx({ex2 + 7, ey - 9}, {ex2 - 5, ey - 5}, 3, BLACK);
    DrawLineEx({o.x + o.width * 0.3f, o.y + o.height * 0.78f},   // bocca
               {o.x + o.width * 0.7f, o.y + o.height * 0.72f}, 3, BLACK);
}

int main() {
    InitWindow(LARGHEZZA, ALTEZZA, "Palla Saltatrice");
    SetTargetFPS(60);

    // Icona della finestra: sfondo giallo con una palla rossa al centro
    Image icona = GenImageColor(64, 64, {255, 220, 40, 255});
    ImageDrawCircle(&icona, 32, 32, 20, {210, 40, 40, 255});
    SetWindowIcon(icona);
    UnloadImage(icona);

    const float terreno = 350.0f;      // altezza del pavimento
    const float gravita = 1800.0f;
    const float forzaSalto = -650.0f;
    const float raggio = 20.0f;        // la palla rossa
    const float posX = 100.0f;         // la palla resta sempre qui: sono i sassi a muoversi

    Vector2 palla;                     // centro della palla
    float velY;
    bool aTerra;
    std::vector<Rectangle> ostacoli;
    float timerOstacolo;
    float velocita;
    float punteggio;
    bool gameOver;

    auto ricomincia = [&]() {
        palla = {posX, terreno - raggio};
        velY = 0.0f;
        aTerra = true;
        ostacoli.clear();
        timerOstacolo = 1.0f;
        velocita = 300.0f;
        punteggio = 0.0f;
        gameOver = false;
    };
    ricomincia();

    while (!WindowShouldClose()) {
        float dt = GetFrameTime();

        if (!gameOver) {
            // --- SALTO (spazio o freccia su) ---
            if (aTerra && (IsKeyPressed(KEY_SPACE) || IsKeyPressed(KEY_UP))) {
                velY = forzaSalto;
                aTerra = false;
            }
            velY += gravita * dt;
            palla.y += velY * dt;
            if (palla.y >= terreno - raggio) {
                palla.y = terreno - raggio;
                velY = 0.0f;
                aTerra = true;
            }

            // --- NUOVI SASSI ---
            timerOstacolo -= dt;
            if (timerOstacolo <= 0.0f) {
                float altezza = (float)GetRandomValue(36, 52);
                ostacoli.push_back({LARGHEZZA + 20.0f, terreno - altezza, 44.0f, altezza});
                timerOstacolo = GetRandomValue(9, 18) / 10.0f;
            }

            // --- I SASSI VENGONO VERSO LA PALLA ---
            for (auto& o : ostacoli) o.x -= velocita * dt;
            ostacoli.erase(
                std::remove_if(ostacoli.begin(), ostacoli.end(),
                               [](const Rectangle& o) { return o.x + o.width < 0; }),
                ostacoli.end());

            // --- COLLISIONI ---
            for (const auto& o : ostacoli) {
                if (CheckCollisionCircleRec(palla, raggio - 2.0f, o)) gameOver = true;
            }

            punteggio += dt * 10.0f;
            velocita += 5.0f * dt;
        } else if (IsKeyPressed(KEY_R)) {
            ricomincia();
        }

        // --- DISEGNO ---
        BeginDrawing();
        ClearBackground({135, 206, 235, 255});  // cielo
        disegnaSole();
        disegnaColline(terreno);
        DrawRectangle(0, (int)terreno, LARGHEZZA, ALTEZZA - (int)terreno, {80, 130, 55, 255});

        for (const auto& o : ostacoli) disegnaSasso(o);

        // Palla rossa con riflesso e occhietto
        DrawCircleV(palla, raggio, {210, 40, 40, 255});
        DrawCircle((int)palla.x - 7, (int)palla.y - 8, 5, {255, 160, 160, 255});
        DrawCircle((int)palla.x + 7, (int)palla.y - 3, 5, WHITE);
        DrawCircle((int)palla.x + 9, (int)palla.y - 3, 2, BLACK);

        DrawText(TextFormat("Punti: %d", (int)punteggio), 10, 10, 24, BLACK);

        if (gameOver) {
            DrawText("GAME OVER", LARGHEZZA / 2 - 110, 150, 40, MAROON);
            DrawText("Premi R per ricominciare", LARGHEZZA / 2 - 140, 200, 22, BLACK);
        }
        EndDrawing();
    }

    CloseWindow();
    return 0;
}