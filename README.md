# PP10

## Goal

In this exercise you will:

* Explore custom `struct` types and `typedef` in C.
* Link against existing system libraries (e.g., `-lm`).
* Create and evolve a custom C library from header-only to a precompiled static archive and install it system-wide.
* Install and use a third-party JSON library (`jansson`) via your package manager.
* Download, build, and install a GitHub-hosted library with a Makefile into standard include/lib paths.

**Important:** Start a stopwatch when you begin and work uninterruptedly for **90 minutes**. When time is up, stop immediately and record where you paused.

---

## Workflow

1. **Fork** this repository on GitHub.
2. **Clone** your fork locally.
3. Create a `solutions/` directory in the project root:

   ```bash
   mkdir solutions
   ```
4. For each task, add or modify source files under `solutions/`.
5. **Commit** and **push** your solutions.
6. **Submit** your GitHub repo link for review.

---

## Prerequisites

* GNU C compiler (`gcc`) and linker (`ld`).
* Make utility (`make`).
* `apt` or your distro’s package manager.

---

## Tasks

### Task 0: Exploring `typedef` and `struct`

**Objective:** Define and use a custom struct type with `typedef`.

1. Create `solutions/point.h` with:

   ```c
   typedef struct {
       double x;
       double y;
   } Point;
   ```
2. Create `solutions/point_main.c` that includes `point.h`, declares a `Point p = {3.0, 4.0}`, and prints its distance from origin using `sqrt(p.x*p.x + p.y*p.y)` (linking `-lm`).

#### Reflection Questions

1. What does typedef struct { ... } Point; achieve compared to struct Point { ... };?
Mit typedef struct { ... } Point; kannst du den Typ Point direkt verwenden, ohne jedes Mal das Schlüsselwort struct davor schreiben zu müssen.
2. **How does the compiler lay out a `Point` in memory?**
Der Compiler speichert einen Point einfach als zwei direkt hintereinander liegende double‑Werte: zuerst x, dann y.
Da ein double auf den meisten Systemen 8 Byte groß ist, besteht ein Point typischerweise aus 16 Byte, wobei die beiden Werte ohne Lücken hintereinander liegen
---

### Task 1: Linking the Math Library (`-lm`)

**Objective:** Compile and link a program against the math library.

1. In `solutions/`, compile `point_main.c` with:

   ```bash
   gcc -o solutions/point_main solutions/point_main.c -lm
   ```
2. Run `./solutions/point_main` and verify it prints `5.0`.

#### Reflection Questions

1. **Why is the `-lm` flag necessary to resolve `sqrt`?**
Die Funktion sqrt gehört nicht automatisch zum normalen C‑Standard‑Programm, sondern liegt in der separaten Math‑Library (libm). Wenn du dein Programm kompilierst, kennt der Compiler zwar die Funktionssignatur aus <math.h>, aber der Linker muss später noch den tatsächlichen Code finden, der die Wurzel berechnet. Das -lm‑Flag sagt dem Linker: „Binde die Math‑Library ein, dort steckt die Implementierung von sqrt.“
2. **What happens if you omit `-lm` when calling math functions?**
Ohne -lm findet der Linker die echte Implementierung von sqrt nicht — er weiß nur, dass es die Funktion geben sollte, aber nicht, wo sie definiert ist. Dadurch bricht der Build mit einer Fehlermeldung wie „undefined reference to sqrt“ ab, und das Programm lässt sich nicht ausführen.
---

### Task 2: Header-Only Library

**Objective:** Create a simple header-only utility library.

1. Create `solutions/libutil.h` with an inline function:

   ```c
   #ifndef LIBUTIL_H
   #define LIBUTIL_H
   static inline int clamp(int v, int lo, int hi) {
       if (v < lo) return lo;
       if (v > hi) return hi;
       return v;
   }
   #endif
   ```
2. Create `solutions/util_main.c` that includes `libutil.h`, calls `clamp(15, 0, 10)` and prints `10`.

3. Compile and run:

   ```bash
   gcc -o solutions/util_main solutions/util_main.c
   ./solutions/util_main
   ```

#### Reflection Questions

1. **What are the advantages and drawbacks of a header-only library?**
Eine Header‑Only‑Library hat den Vorteil, dass du keine extra .c‑Datei kompilieren oder linken musst – alles steckt direkt im Header, und der Compiler kann die Funktionen sofort einbinden. Das macht die Nutzung sehr einfach und reduziert den Build‑Aufwand.
Der Nachteil ist, dass der Code bei jedem Einbinden neu generiert wird, was zu größerem Binärcode führen kann, und dass Fehler im Header sofort alle Dateien betreffen, die ihn inkludieren.
2. **How does `static inline` affect linkage and code size?**
static inline sorgt dafür, dass die Funktion keine globale Symboldefinition erzeugt (also keine Link‑Konflikte entstehen) und dass der Compiler sie direkt an der Aufrufstelle einfügt, wenn es sinnvoll ist. Dadurch spart man Funktionsaufruf‑Overhead, aber der Code kann sich mehrfach wiederholen, was die Binärgröße etwas erhöht.
---

### Task 3: Precompiled Static Library

**Objective:** Convert the header-only utility into a compiled static library and link it manually.

1. Split `clamp` into `solutions/util.c` & `solutions/util.h` (remove `inline` and `static`).
2. Compile:

   ```bash
   gcc -c solutions/util.c -o solutions/util.o
   ```
3. Create the executable linking manually:

   ```bash
   gcc -o solutions/util_main_pc solutions/util.o solutions/util_main.c
   ```
4. Run `./solutions/util_main_pc` to verify output.

#### Reflection Questions

1. **Why must you include `solutions/util.o` when linking instead of just the header?**
Der Header (util.h) enthält nur die Deklaration der Funktion, die eigentliche Implementierung steckt aber in util.c, und erst durch das Kompilieren entsteht die Objektdatei util.o, die den fertigen Maschinen-Code enthält.
Beim Linken muss der Linker diesen Code finden, sonst weiß er nicht, wo die Funktion wirklich definiert ist. Deshalb muss util.o beim Linken angegeben werden, damit die Funktion clamp tatsächlich eingebunden wird.
2. **What symbol resolution occurs at compile vs. link time?**
Compile-Time: Syntax, Typen, Signaturen

Link-Time: echte Funktionsdefinitionen zusammenführen
---

### Task 4: Packaging into `.a` and System Installation

**Objective:** Archive the static library and install it to system paths.

1. Create `libutil.a`:

   ```bash
   ar rcs libutil.a solutions/util.o
   ```
2. Move headers and archive:

   ```bash
   sudo cp solutions/util.h /usr/local/include/libutil.h
   sudo cp libutil.a /usr/local/lib/libutil.a
   sudo ldconfig
   ```
3. Compile a test program using system-installed lib:

   ```bash
   gcc -o solutions/util_sys solutions/util_main.c -lutil
   ```

   (assumes `#include <libutil.h>`)

#### Reflection Questions

1. **How does `ar` create an archive, and how does the linker find `-lutil`?**
ar packt die Objektdateien zu einer einzigen Datei libutil.a. Der Linker sucht bei -lutil automatisch nach einer Datei namens libutil.a in Systempfaden wie /usr/local/lib, daher wird die Library gefunden.
2. **What is the purpose of `ldconfig`?**
ldconfig aktualisiert den System‑Cache der Bibliotheken, damit neu installierte Libraries in /usr/local/lib vom Linker erkannt und genutzt werden können.
---

### Task 5: Installing and Using `jansson`

**Objective:** Install a third-party JSON library and link against it.

1. Install via `apt`:

   ```bash
   sudo apt update && sudo apt install libjansson-dev
   ```
2. Create `solutions/json_main.c`:

   ```c
   #include <jansson.h>
   #include <stdio.h>
   int main(void) {
       json_t *root = json_pack("{s:i, s:s}", "id", 1, "name", "Alice");
       char *dump = json_dumps(root, 0);
       printf("%s\n", dump);
       free(dump);
       json_decref(root);
       return 0;
   }
   ```
3. Compile and run:

   ```bash
   gcc -o solutions/json_main solutions/json_main.c -ljansson
   ./solutions/json_main
   ```

#### Reflection Questions

1. **What files does `libjansson-dev` install, and where?**
Das Paket installiert die Header (jansson.h) nach /usr/include und die kompilierten Bibliotheken (libjansson.so, libjansson.a) nach /usr/lib bzw. /usr/lib/x86_64-linux-gnu.
2. **How does the linker know where to find `-ljansson`?**
Der Linker sucht automatisch nach einer Datei namens libjansson.so oder libjansson.a in den Standard‑Library‑Verzeichnissen. Da das Paket die Dateien dort ablegt, kann -ljansson ohne zusätzliche Pfade aufgelöst werden.
---

### Task 6: Building and Installing a GitHub Library

**Objective:** Download, build, and install a library from GitHub using its Makefile.

1. Choose a small C library on GitHub (e.g., `sesh/strbuf`).
2. Clone and build:

   ```bash
   git clone https://github.com/sesh/strbuf.git
   cd strbuf
   make
   ```
3. Install to system paths:

   ```bash
   sudo make install PREFIX=/usr/local
   sudo ldconfig
   ```
4. Write `solutions/strbuf_main.c` that includes `strbuf.h`, uses its API, and prints a test string.
5. Compile and link:

   ```bash
   gcc -o solutions/strbuf_main solutions/strbuf_main.c -lstrbuf
   ./solutions/strbuf_main
   ```

#### Reflection Questions

1. **What does `make install` do, and how does `PREFIX` affect installation paths?**
make install kopiert die gebauten Dateien (Header, Libraries, ggf. Binaries) in Systemverzeichnisse. Der Wert von PREFIX bestimmt dabei die Basis der Installationspfade, z. B. /usr/local/include und /usr/local/lib. Dadurch landet die Library an Orten, die der Compiler und Linker automatisch durchsuchen.
2. **How can you inspect a library’s exported symbols to verify installation?**
Mit Werkzeugen wie nm, objdump -t oder readelf -s kann man die Symbole einer Library anzeigen. So lässt sich kontrollieren, ob Funktionen wie strbuf_* korrekt im Archiv oder der Shared Library vorhanden sind.
---

**Remember:** Stop after **90 minutes** and record where you stopped.
