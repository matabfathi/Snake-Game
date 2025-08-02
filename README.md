#include <iostream>
#include <conio.h>
#include <windows.h>
#include <vector>
#include <ctime>
#include <fstream>

using namespace std;

enum Direction { STOP = 0, LEFT, RIGHT, UP, DOWN };

const int WIDTH = 20;
const int HEIGHT = 10;
const char SNAKE_HEAD = 'O';
const char SNAKE_BODY = 'o';
const char FOOD_CHAR = '*';
const char OBSTACLE_CHAR = '#';
struct Position {
    int x, y;
};
class Food {
public:
    Position pos;
    void generate(const vector<Position>& occupied) {
        bool valid = false;
        while (!valid) {
            pos.x = rand() % WIDTH;
            pos.y = rand() % HEIGHT;
            valid = true;
            for (auto& p : occupied)
                if (p.x == pos.x && p.y == pos.y)
                    valid = false;
        }
    }
};
class Snake {
public:
    Position head;
    vector<Position> tail;
    int length = 0;
    Snake() {
        head = { WIDTH / 2, HEIGHT / 2 };
    }
    void move(Direction dir) {
        if (length > 0) {
            tail.insert(tail.begin(), head);
            if (tail.size() > length)
                tail.pop_back();
        }
        switch (dir) {
        case LEFT:  head.x--; break;
        case RIGHT: head.x++; break;
        case UP:    head.y--; break;
        case DOWN:  head.y++; break;
        default: break;
        }
    }
    bool collisionWithSelf() {
        for (auto& part : tail)
            if (part.x == head.x && part.y == head.y)
                return true;
        return false;
    }
    vector<Position> getOccupiedPositions() const {
        vector<Position> all = tail;
        all.push_back(head);
        return all;
    }
};
class Game {
private:
    Snake snake;
    Food food;
    vector<Position> obstacles;
    Direction dir = STOP;
    bool gameOver = false;
    bool paused = false;
    int score = 0;
    int highScore = 0;
    int speed = 100;
    int speedIncreaseInterval = 30;
    int speedStep = 10;
public:
    Game() {
        srand(time(0));
        loadHighScore();
        food.generate(snake.getOccupiedPositions());
        generateObstacles();
    }
    void run() {
        while (!gameOver) {
            draw();
            input();
            if (!paused) logic();
            Sleep(speed);
        }
        system("cls");
        if (score == highScore) {
            cout << "🎉 NEW HIGH SCORE! 🎉\n\n";
        }
        cout << "🎮 Game Over!\n";
        cout << "Score: " << score << "\n";
        cout << "High Score: " << highScore << "\n";
    }
private:
    void draw() {
        system("cls");
        for (int i = 0; i < WIDTH + 2; i++) cout << "#";
        cout << "\n";
        for (int y = 0; y < HEIGHT; y++) {
            cout << "#";
            for (int x = 0; x < WIDTH; x++) {
                if (x == snake.head.x && y == snake.head.y) {
                    setColor(10); cout << SNAKE_HEAD; setColor(7);
                }
                else if (isTail(x, y)) {
                    setColor(2); cout << SNAKE_BODY; setColor(7);
                }
                else if (x == food.pos.x && y == food.pos.y) {
                    setColor(12); cout << FOOD_CHAR; setColor(7);
                }
                else if (isObstacle(x, y)) {
                    setColor(8); cout << OBSTACLE_CHAR; setColor(7);
                }
                else {
                    cout << " ";
                }
            }
            cout << "#\n";
        }
        for (int i = 0; i < WIDTH + 2; i++) cout << "#";
        cout << "\n";
        cout << "Score: " << score << "\tHigh Score: " << highScore;
        cout << "\tSpeed: " << (100 - speed) / 10 + 1;
        if (paused) cout << "  [PAUSED]";
        cout << "\nControls: W A S D | Pause: P | Exit: X\n";
    }
    void input() {
        if (_kbhit()) {
            switch (_getch()) {
            case 'a': if (dir != RIGHT) dir = LEFT; break;
            case 'd': if (dir != LEFT)  dir = RIGHT; break;
            case 'w': if (dir != DOWN)  dir = UP; break;
            case 's': if (dir != UP)    dir = DOWN; break;
            case 'p': paused = !paused; break;
            case 'x': gameOver = true; break;
            }
        }
    }
    void logic() {
        snake.move(dir);
        if (snake.head.x < 0 || snake.head.x >= WIDTH ||
            snake.head.y < 0 || snake.head.y >= HEIGHT)
            endGame();
        if (snake.collisionWithSelf())
            endGame();
        for (auto& o : obstacles)
            if (snake.head.x == o.x && snake.head.y == o.y)
                endGame();
        if (snake.head.x == food.pos.x && snake.head.y == food.pos.y) {
            score += 10;
            snake.length++;
            food.generate(snake.getOccupiedPositions());
            if (score > highScore) saveHighScore();
            
            if (score % speedIncreaseInterval == 0 && speed > 30) {
                speed -= speedStep;
            }
        }
    }
    void setColor(int color) {        SetConsoleTextAttribute(GetStdHandle(STD_OUTPUT_HANDLE), color);
    }
    bool isTail(int x, int y) {
        for (auto& part : snake.tail)
            if (part.x == x && part.y == y) return true;
        return false;
    }
    bool isObstacle(int x, int y) {
        for (auto& obs : obstacles)
            if (obs.x == x && obs.y == y) return true;
        return false;
    }
    void endGame() {
        gameOver = true;
    }
    void generateObstacles() {
        obstacles.clear();
        int obstacleCount = 5 + rand() % 5;        
        for (int i = 0; i < obstacleCount; i++) {
            Position obs;
            bool valid = false;
            while (!valid) {
                obs.x = rand() % WIDTH;
                obs.y = rand() % HEIGHT;
                valid = true;                
                if ((obs.x == snake.head.x && obs.y == snake.head.y) ||
                    (obs.x == food.pos.x && obs.y == food.pos.y)) {
                    valid = false;
                    continue;
                }                
                for (auto& part : snake.tail)
                    if (part.x == obs.x && part.y == obs.y)
                        valid = false;
                        
                for (auto& existing : obstacles)
                    if (existing.x == obs.x && existing.y == existing.y)
                        valid = false;
            }
            obstacles.push_back(obs);
        }
    }
    void loadHighScore() {
        ifstream file("highscore.txt");
        if (file >> highScore) {
            file.close();
        }
        else {
            highScore = 0;
        }
    }
    void saveHighScore() {
        highScore = score;
        ofstream file("highscore.txt");
        file << highScore;
        file.close();
    }
};
int main() {
    Game game;
    game.run();
    return 0;
}

