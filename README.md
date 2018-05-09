## Welcome to Phillips Project #2
We are givin a battleship game emulator that we have to code a bot to play and win the game for us. The bot should not access where the ships are located (so it cant cheat) it should use set conditions to narrow down possible ship locations. We are givin 15 files which we are only allowed to change the bot file because that is the only file we submit to class. We compete against other classmates on who can make the most efficeint bot for the game inorder to get better project grades.

### original Bot file
```markdown
#include <cstdlib>
#include <iostream>
#include "bot.h"
#include "screen.h"

using namespace std;

int ROWS;
int COLS;

int iter = 0;

/* Initialization procedure, called when the game starts:

   init (rows, cols, num, screen, log) 
 
   Arguments:
    rows, cols = the boards size
    num        = the number of ships 
    screen     = a screen to update your knowledge about the game
    log        = a cout-like output stream
*/
void init(int rows, int cols, int num, Screen &screen, ostream &log) 
{
  ROWS = rows;
  COLS = cols;
  log << "Start." << endl;
}


/* The procedure handling each turn of the game:
 
   next_turn(sml, lrg, num, gun, screen, log)
 
   Arguments:
    sml, lrg = the sizes of the smallest and the largest ships that are currently alive
    num      = the number of ships that are currently alive
    gun      = a gun.
               Call gun.shoot(row, col) to shoot: 
                  Can be shot only once per turn. 
                  Returns MISS, HIT, HIT_N_SUNK, ALREADY_HIT, or ALREADY_SHOT.
    screen   = a screen to update your knowledge about the game
    log      = a cout-like output stream
*/
void next_turn(int sml, int lrg, int num, Gun &gun, Screen &screen, ostream &log)
{

  int r = iter / COLS;
  int c = iter % COLS;
  iter += 1;
  
  log << "Smallest: " << sml << " Largest: " << lrg << ". ";
  log << "Shoot at " << r << " " << c << endl;

  result res = gun.shoot(r, c);

  // add result on the screen
  if (res == MISS)
    screen.mark(r, c, 'x', BLUE); 
  else 
    if (res == HIT)
      screen.mark(r, c, '@', GREEN); 
  else 
    if (res == HIT_N_SUNK)
      screen.mark(r, c, 'S', RED); 

}
```
##bot.h
```markdown

#ifndef BOT_H
#define BOT_H

#include <vector>
#include <iostream>
#include "gun.h"
#include "screen.h"

/* Initialization procedure, called when the game starts:

   init (rows, cols, num, screen, log) 
 
   Arguments:
    rows, cols = the boards size
    num        = the number of ships 
    screen     = a screen to update your knowledge about the game
    log        = a cout-like output stream
*/
void init(int rows, int cols, int num, Screen &screen, std::ostream &log);


/* The procedure handling each turn of the game:
 
   next_turn(sml, lrg, num, gun, screen, log)
 
   Arguments:
    sml, lrg = the sizes of the smallest and the largest ships that are currently alive
    num      = the number of ships that are currently alive
    gun      = a gun.
               Call gun.shoot(row, col) to shoot: 
                  Can be shot only once per turn. 
                  Returns MISS, HIT, HIT_N_SUNK, ALREADY_HIT, or ALREADY_SHOT.
    screen   = a screen to update your knowledge about the game
    log      = a cout-like output stream
*/
void next_turn(int sml, int lrg, int num, Gun &gun, Screen &screen, std::ostream &log);


#endif

```
## common.h
```markdown

#ifndef COMMON_H
#define COMMON_H

enum result {MISS, HIT, HIT_N_SUNK, ALREADY_HIT, ALREADY_SHOT};

enum color {GRAY, RED, GREEN, BLUE};

const int log_line_length = 70; 
#endif 

```
## gun
```markdown

#include "gun.h"

Gun::Gun(result (*oracle)(int, int)) : oracle(oracle) { }

result Gun::shoot(int row, int col) {
  return (*oracle)(row, col);
}

```

## gun.h
```markdown

#ifndef GUN_H
#define GUN_H

#include "common.h"

class Gun {
  result (*oracle) (int row, int col);
public : 
  Gun(result (*oracle) (int, int));
  result shoot(int row, int col);
};

#endif

```

## main
```markdown

#include <unistd.h>
#include <stdio.h>
#include <curses.h>
#include <locale.h>
#include <signal.h>
#include <sys/time.h>
#include <time.h>
#include <fcntl.h>

#include <stdlib.h>
#include <string.h>

#include "state.h"
#include "screen.h"
#include "output.h"

int update_from_input(state &s, Screen &screen, std::ostream &gamelog);
void finish(int sig);

/* Run the game */
void run (state &s) {

  buf buf;
  std::ostream gamelog (&buf); // game log
  Screen screen(s.rows, s.cols); // user's screen output

  output(s, screen, buf.data);
  
  { // Bot logic init
    init(s.rows, s.cols, 0, screen, gamelog);
  }

  int k = 0;
  int finished = 0;
  while( !finished ) {
    k++;
    if (k>10) { 
      
      k=0;
      // run the bot's logic and update the state
      if (s.play) {
        update(s, screen, gamelog);
      }

    }
    finished = update_from_input(s, screen, gamelog);
    output(s, screen, buf.data);
    usleep(10000);
  }
}

/* Helper function, puts the value v in the interval [min, max] */
int put_in_range(int v, int min, int max) {
  if (v < min)
    return min;
  else if (v > max)
    return max;
  else
    return v;
}

/* Main function. Mostly boilerplate code of setting up the terminal. */
int main(int argc, char* argv[]){

  /*       1    2    3   4   5   6    */
  /* ARGS: ROWS COLS SML LRG NUM SEED */
  /* Init the game */
  int rows = 20;
  int cols = 20;
  int sml = 1;
  int lrg = 5;
  int num = 50;
  // Random seed
  if (argc > 6) {
    srand(atoi(argv[6]));
  }
  else {
    srand(time(NULL));
    rows += -2 + rand() % 5;
    cols += -2 + rand() % 5;
    num += -5 + rand() % 11;
  }
  if (argc > 1) {rows = put_in_range(atoi(argv[1]), 5, 35);}
  if (argc > 2) {cols = put_in_range(atoi(argv[2]), 5, 35);}
  int max_size = (cols > rows) ? cols : rows; 
  if (argc > 3) {sml = put_in_range(atoi(argv[3]), 1, max_size);}
  if (argc > 4) {lrg = put_in_range(atoi(argv[4]), sml, max_size);}
  if (argc > 5) {num = put_in_range(atoi(argv[5]), 0, rows*cols);}
  // Skip the video simulation?
  bool fast = (argc > 7) && (strcmp(argv[7], "fast") == 0);
 
  // Init the game state
  state st;
  init(st, rows, cols, sml, lrg, num);

  if (!fast) 
  {
    /* User interface initialization */
    signal(SIGINT, finish);

    /* ncurses initialization */
    setlocale(LC_ALL, "");
    initscr();     /* initialize the library and screen */
    cbreak();      /* put terminal into non-blocking input mode */
    nonl();        /* no NL -> CRNL on output */
    noecho();      /* turn off echo */
    start_color();
    curs_set(0);   /* hide the cursor */
    timeout(0);

    use_default_colors();
    init_pair(0, COLOR_WHITE, COLOR_BLACK);
    init_pair(1, COLOR_WHITE, COLOR_BLACK);
    init_pair(2, COLOR_BLACK, COLOR_BLACK);
    init_pair(3, COLOR_RED, COLOR_BLACK);
    init_pair(4, COLOR_GREEN, COLOR_BLACK);
    init_pair(5, COLOR_BLUE, COLOR_BLACK);
    init_pair(6, COLOR_YELLOW, COLOR_BLACK);
    init_pair(7, COLOR_MAGENTA, COLOR_BLACK);
    init_pair(8, COLOR_CYAN, COLOR_BLACK);

    color_set(0, NULL);
    assume_default_colors(COLOR_WHITE, COLOR_BLACK);

    attrset(A_NORMAL | COLOR_PAIR(0));
    refresh();
    clear();
   
    /* Run the game */
    run(st);
  
    /* Restore the teminal state */
    echo();
    curs_set(1);
    clear();
    endwin();
  }
  else {

    buf buf;
    std::ostream gamelog (&buf); // game log
    Screen screen(st.rows, st.cols); // user's screen output

    // init bot logic
    init(st.rows, st.cols, 0, screen, gamelog);
    
    // run the game until it's over
    while(st.alive && st.ships > 0) { update(st, screen, gamelog); }
    
    printf("%i ", st.round);
    if (st.alive) 
      printf("success\n");
    else
      printf("failure\n");

  }

  return 0;
}

int update_from_input(state &s, Screen &screen, std::ostream &gamelog)
{
    int c;
    int finished=0;

    while ( !finished && (c=getch()) != ERR ) {

      switch(c){
        case 'q': case 'Q':
          finished = 1;
          break;
        case 'f': case 'F':
          for(int i = 0; i<500; ++i){ 
            update(s, screen, gamelog);
          }
          break;
        case 's': case 'S':
          s.play = false;
          update(s, screen, gamelog);
          break;
        case 'p': case 'P':
          s.play = !s.play;
          break;
        default:;
      }

    }                
    return finished;
}

/* SIGINT */
void finish(int sig)
{
  echo();
  curs_set(1);
  clear();
  endwin();
  exit(0);
}

```
## Makefile
```markdown
SHELL   = /bin/sh
CC      = g++
INSTALL = install
EXEC    = battleships 

# Sources
SRCS_0 = gun.cpp screen.cpp bot.cpp outstream.cpp state.cpp output.cpp
SRCS = $(SRCS_0) main.cpp
HDRS = $(SRCS_0:.cpp=.h) 
OBJS = $(SRCS:.cpp=.o)

EXECS = $(EXEC)

CFLAGS += -Wall -O2
LDLIBS += -lm -lncurses


.PHONY: all clean

# Build
all: $(EXECS)

clean:
	-rm -f $(OBJS) $(EXECS)

%.o: %.cpp $(HDRS)  
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $<

$(EXEC): $(OBJS) $(HDRS)
	$(CC) $(CFLAGS) $(LDFLAGS) $(OBJS) $(LDLIBS) -o $(EXEC)


```

## output
```markdown

#include <curses.h>
#include <string>
#include "output.h"
#include "screen.h"

#define X(r,c) (c)*2 + 6
#define Y(r,c) (r)*1 + 3

#define GRAY_COLOR (A_NORMAL | COLOR_PAIR(1))
#define WATER_COLOR (A_NORMAL | COLOR_PAIR(5))

void print_hint(std::string s, char c) {
  bool needs_highlight = true;
  attrset(A_NORMAL | COLOR_PAIR(1));
  addch('[');
  for(unsigned int i = 0; i<s.size(); ++i) {
    if (needs_highlight && s[i] == c) {
      attrset(A_BOLD | COLOR_PAIR(1));
      addch(c);
      attrset(A_NORMAL | COLOR_PAIR(1));
      needs_highlight = false;
    }
    else
      addch(s[i]);
  }
  addch(']');
}

void output(state &s, Screen &screen, std::vector<std::string> &logbufdata) {
 
  int rows = screen.get_rows();
  int cols = screen.get_cols();


  // draw the labels
  for(int r = 0; r < rows; ++r) {
    move(Y(r,-1), X(r,-1));
    printw("%i", r%10);
    
    if (r/10 > 0) {
      move(Y(r,-1), X(r,-1)-1);
      printw("%i", r/10);
    }
  }

  for(int c = 0; c < cols; ++c) {
    move(Y(-1,c), X(-1,c));
    printw("%i", c%10);
    
    if (c/10 > 0) {
      move(Y(-1,c)-1, X(-1,c));
      printw("%i", c/10);
    }
  }

  // draw the map
  for(int r = 0; r < rows; ++r) {
    for(int c = 0; c < cols; ++c) {
      move(Y(r,c), X(r,c));
      
      switch(screen.read_color(r, c)) {
        case GRAY: 
          attrset(GRAY_COLOR);
          break;
        case BLUE: 
          attrset(WATER_COLOR);
          break;
        case GREEN: 
          attrset(A_BOLD | COLOR_PAIR(4));
          break;
        case RED: 
          attrset(A_BOLD | COLOR_PAIR(3));
          break;
        default:
          attrset(GRAY_COLOR);
          break;
      }

      addch(screen.read(r, c));
      addch(' ');
    }
  }

  // Draw the interface

  int yy = Y(rows + 1, 0);
  int xx = 2; //X(rows + 1, 0);

  attrset(A_NORMAL | COLOR_PAIR(1));
  move(yy, xx);
  printw("Ships: %i / %i  ", s.ships, s.initial_ships);
  move(yy, xx + 20);
  printw("Round: %i ", s.round);
  
  if (!s.alive) {
    move(yy, xx + 36);
    attrset(A_BOLD | COLOR_PAIR(3));
    printw("GAME OVER");
    attrset(A_NORMAL | COLOR_PAIR(1));
  }
  if (s.alive && s.ships == 0) {
    move(yy, xx + 36);
    printw("YOU WON!");
  }

  move(yy+2, xx);
  print_hint("Quit", 'Q');
  
  move(yy+2, xx+10);
  if (s.play)
    print_hint("Pause", 'P');
  else {
    print_hint("Play", 'P');
    addch(' ');
  }

  move(yy+2, xx+20);
  print_hint("Step", 'S') ;
  
  move(yy+2, xx+30);
  print_hint("Fast-forward", 'F') ;
 
  /* Log */
  attrset(GRAY_COLOR);
  move(yy+4, xx);
  for (unsigned int i = 0; i < log_line_length; i++) {
    addch('-');
  }
  move(yy+4+9, xx);
  for (unsigned int i = 0; i < log_line_length; i++) {
    addch('-');
  }
  move(yy+4, xx + log_line_length - 20);
  addch('|');
  printw(" Captain's log ");
  addch('|');

  for (unsigned int i = 0; i < logbufdata.size(); ++i) {
    move(yy+5 + i, xx);
    clrtoeol();
  }

  attrset(A_NORMAL | COLOR_PAIR(1));
  for (unsigned int i = 0; i < logbufdata.size(); ++i) {
    move(yy+5 + i, xx);
    printw("%s", logbufdata[i].c_str());
  }

  refresh();
}

```

## output.h
```markdown

#ifndef OUTPUT_H
#define OUTPUT_H

#include "state.h"

void output(state &st, Screen &screen, std::vector<std::string> &logbufdata);

#endif

```

## outstream
```markdown

#include "outstream.h"
#include "common.h"

#include <cstdio> // for EOF

int buf::overflow(int c) {
  if (c == EOF) {
    return EOF;
  }
  else {
    unsigned int max_line_len = log_line_length;
    int last = data.size()-1;
    if (c == '\n' || data[last].size() >= max_line_len) {
      std::string s;
      if (c != '\n') s.push_back(c);
      
      if (data.size() >= (unsigned int)max_size) {
        for(int j = 0; j < last; ++j) {
          data[j] = data[j+1];
        }
        data.pop_back();
      }

      data.push_back(s);
    }
    else{
      data[last].push_back(c);
    }
    
    return c;
  }
}

int buf::sync() {
  return 0;
}

buf::buf() {
  max_size = 8;
  data.resize(1);
  data[0] = "";
}

```
## outstream.h
```markdown

#ifndef OUTSTREAM_H
#define OUTSTREAM_H

#include <streambuf>
#include <iostream>
#include <vector>

class buf : public std::streambuf {
 
  private:
    int max_size;
  public:  
    std::vector<std::string> data;
 
  protected:
    virtual int overflow(int c);
    virtual int sync();

  public: 
    buf();
};

#endif

```

## screen

```markdown

#include "screen.h"

bool Screen::in_range (int r, int c) {
  return (r >= 0 && c >= 0 && r < (int)symbols.size() && c < (int)symbols[0].size());
}
  
int Screen::get_rows() {
  return (int)symbols.size();
}

int Screen::get_cols(){
  if (symbols.size() > 0) 
    return (int)symbols[0].size();
  else
    return 0;
}

Screen::Screen(int rows, int cols) {
  if (!(rows > 0 && cols > 0 && rows < 100 && cols < 100)) {
    return;
  }

  symbols.resize(rows);
  for(int r = 0; r < rows; ++r) {
    symbols[r].resize(cols);
    for(int c = 0; c < cols; ++c) {
      symbols[r][c].ch = '-';
      symbols[r][c].clr = BLUE;
    }
  }
}

void Screen::mark(int row, int col, char ch, color clr) {
  if (in_range(row, col)) {
    symbols[row][col].ch = ch;
    symbols[row][col].clr = clr;
  }
}

char Screen::read(int row, int col) {
  if (in_range(row, col)) {
    return symbols[row][col].ch;
  }
  return ' ';
}

color Screen::read_color(int row, int col) {
  if (in_range(row, col)) {
    return symbols[row][col].clr;
  }
  return GRAY;
}


```

## screen.h
```markdown

#ifndef SCREEN_H
#define SCREEN_H

#include <vector>
#include "common.h"

struct symbol { 
  char ch;
  color clr;
};

class Screen {
  std::vector< std::vector <symbol> > symbols; 
  bool in_range(int row, int col);

public: 
  Screen(int rows, int cols);
  int get_rows ();
  int get_cols ();
  void mark(int row, int col, char ch, color clr = GRAY);
  char read(int row, int col);
  color read_color(int row, int col);
};

#endif

```
## state

```markdown

#include "state.h"

bool in_range(state &s, int r, int c) {
  return (r >= 0 && r < (int)s.map.size() && c >= 0 && c < (int)s.map[0].size());
}

void block(state &s, int r, int c) {
  if (in_range(s, r, c)) {
    s.map[r][c] = BLOCKED;
  }
}

struct loc {
  int r;
  int c;
};

bool place_ship(state &s, int size) {
  std::vector<struct loc> hor; // horizontal
  std::vector<struct loc> ver; // vertical

  // horizontal
  for (int r = 0; r < s.rows; ++r) {
    for (int c = 0; c < s.cols - (size-1); ++c) {
      bool okay = true;
      for(int i = 0; okay && (i < size); ++i) { if(s.map[r][c+i] != EMPTY) okay = false; }
      if (okay) {
        struct loc loc = {r, c};
        hor.push_back(loc);
      }
    }
  }
  // vertical
  for (int c = 0; c < s.cols; ++c) {
    for (int r = 0; r < s.rows - (size-1); ++r) {
      bool okay = true;
      for(int i = 0; okay && (i < size); ++i) { if(s.map[r+i][c] != EMPTY) okay = false; }
      if (okay) {
        struct loc loc = {r, c};
        ver.push_back(loc);
      }
    }
  }

  int num = hor.size() + ver.size();
  if (num > 0) {
    int x = rand() % num;
    if (x < (int)hor.size()) { // if horizontal
      struct loc loc = hor[x];
      for(int i = 0; i < size; ++i) {
        int r = loc.r;
        int c = loc.c + i;
        s.map[r][c] = SHIP; 
        block(s, r-1, c);
        block(s, r+1, c);
      }
      block(s, loc.r, loc.c - 1);
      block(s, loc.r, loc.c + size);
    }
    else { // if vertical
      struct loc loc = ver[x - hor.size()];
      for(int i = 0; i < size; ++i) { 
        int r = loc.r + i;
        int c = loc.c;
        s.map[r][c] = SHIP; 
        block(s, r, c - 1);
        block(s, r, c + 1);
      }
      block(s, loc.r - 1, loc.c);
      block(s, loc.r + size, loc.c);
    }
    return true;
  }
  else 
    return false;

}

void init(state &s, int rows, int cols, int smallest, int largest, int number) {
  s.rows = rows;
  s.cols = cols;
  
  // initialize the map
  s.map.resize(rows);
  for(int r = 0; r < rows; ++r) {
    s.map[r].resize(cols);
    for(int c = 0; c < cols; ++c) {
      s.map[r][c] = EMPTY;
    }
  }
  
  s.round = 1;
  
  s.play = true;
  s.alive = true;

  // place the fleet
  s.ships = 0;
  int can_continue = true;
  while(s.ships < number && can_continue) {
    int size = smallest + rand()%(largest + 1 - smallest);
    bool success = place_ship(s, size);
    if (success)
      s.ships ++;
    else
      can_continue = size > smallest;
  }
  s.initial_ships = s.ships;
 
  // replace BLOCKED by EMPTY
  for(int r = 0; r < rows; ++r) {
    for(int c = 0; c < cols; ++c) {
      if (s.map[r][c] == BLOCKED)
        s.map[r][c] = EMPTY;
    }
  }
}
  

bool ship_alone(state &s, int r, int c) {
  return
    in_range(s,r,c) && (s.map[r][c] != EMPTY)
      && (!in_range(s,r-1,c) || s.map[r-1][c] == EMPTY)
      && (!in_range(s,r+1,c) || s.map[r+1][c] == EMPTY)
      && (!in_range(s,r,c-1) || s.map[r][c-1] == EMPTY)
      && (!in_range(s,r,c+1) || s.map[r][c+1] == EMPTY);
}


// find the largest and the smallest ship sizes
void find_ships(state &s, int &sml, int &lrg) {

  if (s.ships <= 0) {
    sml = 0;
    lrg = 0;
    return;
  }

  lrg = 0;
  sml = s.rows + s.cols;

  // scan horizontally
  for(int r = 0; r < s.rows; ++r) {

    int len = 0;
    bool not_sunk = false;
    
    for(int c = 0; c < s.cols+1; ++c) {
      if (in_range(s,r,c) && s.map[r][c] != EMPTY) {
        len ++;
        if (s.map[r][c] == SHIP)
          not_sunk = true;
      }
      else {
        if (not_sunk && (len > 1 || ship_alone(s,r,c-1))) {
          // ship found
          if (len > lrg) lrg = len;
          if (len < sml) sml = len;
        }
        not_sunk = false;
        len = 0;
      }
    }
  }
  
  // scan vertically
  for(int c = 0; c < s.cols; ++c) {

    int len = 0;
    bool not_sunk = false;
    
    for(int r = 0; r < s.rows+1; ++r) {
      if (in_range(s,r,c) && s.map[r][c] != EMPTY) {
        len ++;
        if (s.map[r][c] == SHIP)
          not_sunk = true;
      }
      else {
        if (not_sunk && (len > 1 || ship_alone(s,r-1,c))) {
          // ship found
          if (len > lrg) lrg = len;
          if (len < sml) sml = len;
        }
        not_sunk = false;
        len = 0;
      }
    }
  }
  
}

namespace { 
  std::vector <std::vector <result> > xrm;
  bool xloaded = false;
  int xrow = -1;
  int xcol = -1;
  
  result xshoot(int row, int col) {

    if (xloaded){
      if (row >= 0 && row < (int)xrm.size() && col >= 0 && col < (int)xrm[0].size()) {
        result res = xrm[row][col];
        xloaded = false;
        xrow = row;
        xcol = col;
        return res;
      }
      else {
        xloaded = false;
        return MISS;
      }
    }

    return ALREADY_SHOT;

  }
  
  bool xalone_alive(state &s, int r, int c) {
    for (int rr = r-1; rr >= 0 && s.map[rr][c] != EMPTY; rr--) { if (s.map[rr][c] == SHIP) return false; }
    for (int rr = r+1; rr < s.rows && s.map[rr][c] != EMPTY; rr++) { if (s.map[rr][c] == SHIP) return false; }
    for (int cc = c-1; cc >= 0 && s.map[r][cc] != EMPTY; cc--) { if (s.map[r][cc] == SHIP) return false; }
    for (int cc = c+1; cc < s.cols && s.map[r][cc] != EMPTY; cc++) { if (s.map[r][cc] == SHIP) return false; }

    return true;
  }

  void xreload (state &s) {
    xloaded = true;
    xrow = -1;
    xcol = -1;
    xrm.resize(s.rows);
    for (int r = 0; r < s.rows; ++r) {
      xrm[r].resize(s.cols);
      for (int c = 0; c < s.cols; ++c) {
        switch (s.map[r][c]) {
          case SHIP:
            if (xalone_alive(s,r,c))
              xrm[r][c] = HIT_N_SUNK;
            else
              xrm[r][c] = HIT;
            break;
          case DMG:
            xrm[r][c] = ALREADY_HIT;
            break;
          default:
            xrm[r][c] = MISS;
            break;
        }
      }
    }
  }
}

void update(state &s, Screen &screen, std::ostream &gamelog) { 
  // don't do anything if the game has finished
  if (!s.alive || s.ships <= 0) return;

  int number = s.ships;
  int smallest = 0;
  int largest = 0;
  find_ships(s, smallest, largest);

  // prepare the gun
  xreload(s);
  Gun gun(xshoot);

  next_turn(smallest, largest, number, gun, screen, gamelog);

  // if shot during the turn
  if (xrow != -1) { 
    switch (s.map[xrow][xcol]) {
      case SHIP:
        if (xalone_alive(s,xrow,xcol)) {
          s.ships--;
        }
        s.map[xrow][xcol] = DMG;
        break;
      default: // otherwise miss
        if (s.ships > 0) { // round increment
          s.round = s.round + 1;
        }
        break;
    }
  }
  else { // if didn't shoot or shot out of bounds
    s.round = s.round + 1;
  }

  if (s.round > s.rows * s.cols) {
    s.alive = false;
  }
}



```

## state.h
```markdown

#ifndef STATE_H
#define STATE_H

#include <cstdlib>
#include <iostream>
#include <vector>

#include "bot.h"
#include "outstream.h"

enum tile {EMPTY, BLOCKED, SHIP, DMG};

struct state {
  // dimensions
  int rows;
  int cols;
  // map
  std::vector <std::vector<tile> > map;

  int initial_ships;
  int ships;
  int round;

  // UI
  bool play;
  bool alive;
};

void init(state &s, int rows, int cols, int smallest, int largest, int number);

void update(state &s, Screen &screen, std::ostream &gamelog);

#endif

```
