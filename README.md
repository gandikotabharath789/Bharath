#include <stdio.h>
#include <stdlib.h>
#include <time.h>

/* ---- Part 1: small differences between operating systems ----
   We only need three things: read a key without waiting,
   wait a few milliseconds, and clear the screen.            */
#ifdef _WIN32
    #include <conio.h>
    #include <windows.h>
    #define CLEAR "cls"
    #define delay(ms) Sleep(ms)
    int getKey()                       /* returns -1 if no key pressed */
    {
        if (_kbhit())
            return _getch();
        return -1;
    }
    void setup() {}
    void restore() {}
#else
    #include <termios.h>
    #include <unistd.h>
    #define CLEAR "clear"
    #define delay(ms) usleep((ms) * 1000)
    struct termios oldSettings;
    void restore()                     /* put the terminal back to normal */
    {
        tcsetattr(0, TCSANOW, &oldSettings);
    }
    void setup()                       /* read keys instantly, no Enter */
    {
        struct termios t;
        tcgetattr(0, &oldSettings);
        t = oldSettings;
        t.c_lflag &= ~(ICANON | ECHO);
        t.c_cc[VMIN] = 0;
        t.c_cc[VTIME] = 0;
        tcsetattr(0, TCSANOW, &t);
        atexit(restore);
    }
    int getKey()                       /* returns -1 if no key pressed */
    {
        unsigned char c;
        if (read(0, &c, 1) == 1)
            return c;
        return -1;
    }
#endif

/* ---- Part 2: game data ---- */
#define W 40                           /* board width  */
#define H 20                           /* board height */

int x[W * H], y[W * H];                /* x[i], y[i] = position of body part i (0 = head) */
int length;                            /* current length of the snake */
int dx, dy;                            /* direction: dx = -1/0/1, dy = -1/0/1 */
int foodX, foodY;                      /* position of the food */
int score;
int gameOver;

/* put the food on a random empty cell */
void placeFood()
{
    int i, onSnake;
    do
    {
        foodX = rand() % W;
        foodY = rand() % H;
        onSnake = 0;
        for (i = 0; i < length; i++)
            if (x[i] == foodX && y[i] == foodY)
                onSnake = 1;
    } while (onSnake);                 /* try again if it landed on the snake */
}

/* set everything to the starting values */
void init()
{
    int i;
    length = 3;
    for (i = 0; i < length; i++)       /* snake starts in the middle, going right */
    {
        x[i] = W / 2 - i;
        y[i] = H / 2;
    }
    dx = 1;
    dy = 0;
    score = 0;
    gameOver = 0;
    placeFood();
}

/* ---- Part 3: draw the board ---- */
void draw()
{
    int r, c, i;
    char cell;

    system(CLEAR);
    printf("SNAKE GAME     Score: %d\n", score);
    for (c = 0; c < W + 2; c++)        /* top wall */
        putchar('#');
    putchar('\n');

    for (r = 0; r < H; r++)
    {
        putchar('#');                  /* left wall */
        for (c = 0; c < W; c++)
        {
            cell = ' ';
            if (c == foodX && r == foodY)
                cell = '*';
            for (i = 0; i < length; i++)
                if (x[i] == c && y[i] == r)
                    cell = (i == 0) ? 'O' : 'o';   /* O = head, o = body */
            putchar(cell);
        }
        putchar('#');                  /* right wall */
        putchar('\n');
    }

    for (c = 0; c < W + 2; c++)        /* bottom wall */
        putchar('#');
    printf("\nW A S D = move     X = quit\n");
}

/* ---- Part 4: keyboard ---- */
void input()
{
    int key;
    while ((key = getKey()) != -1)     /* read all keys waiting */
    {
        if (key >= 'A' && key <= 'Z')  /* accept capital letters too */
            key = key + 32;
        /* the snake may not turn straight back on itself */
        if (key == 'w' && dy != 1)  { dx = 0;  dy = -1; }
        if (key == 's' && dy != -1) { dx = 0;  dy = 1;  }
        if (key == 'a' && dx != 1)  { dx = -1; dy = 0;  }
        if (key == 'd' && dx != -1) { dx = 1;  dy = 0;  }
        if (key == 'x')
            gameOver = 1;
    }
}

/* ---- Part 5: move the snake and check what happened ---- */
void update()
{
    int i, newX, newY, ate;

    newX = x[0] + dx;                  /* where the head will be */
    newY = y[0] + dy;

    if (newX < 0 || newX >= W || newY < 0 || newY >= H)   /* hit a wall */
    {
        gameOver = 1;
        return;
    }
    for (i = 0; i < length; i++)                           /* hit itself */
        if (x[i] == newX && y[i] == newY)
        {
            gameOver = 1;
            return;
        }

    ate = (newX == foodX && newY == foodY);
    if (ate && length < W * H)
    {
        length++;                      /* grow: the tail stays where it was */
        score += 10;
    }

    for (i = length - 1; i > 0; i--)   /* every part moves to where the one before it was */
    {
        x[i] = x[i - 1];
        y[i] = y[i - 1];
    }
    x[0] = newX;                       /* move the head */
    y[0] = newY;

    if (ate)
        placeFood();
}

/* ---- Part 6: main game loop ---- */
int main()
{
    int speed;

    srand((unsigned)time(NULL));       /* different food positions each game */
    setup();
    init();

    while (!gameOver)
    {
        draw();
        input();
        update();
        speed = 130 - length * 2;      /* faster as the snake grows */
        if (speed < 50)
            speed = 50;
        if (dy != 0)                   /* rows are taller than columns, so slow down vertically */
            speed = speed + 40;
        delay(speed);
    }

    draw();
    printf("\nGAME OVER!   Final score: %d\n", score);
    return 0;
}
