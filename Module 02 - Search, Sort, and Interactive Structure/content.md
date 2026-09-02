# Search, Sort, and Interactive Structure

**CPSI 27703: Intro to Game Programming**

This module covers four ideas you will reuse all semester:

1. **Searching** data (linear and binary)
2. **Sorting** data (thinking it through with bubble sort, then the STL)
3. **Random number generation** (`rand()` vs `<random>`)
4. **The game loop** (update/draw with polymorphic game objects)

> PREFERENCE: Good game code is about **readability**, **reusability**, and **abstraction**.

---

## Linear Search

**Linear search** walks from the start of a container until it finds the key or runs out of data.

1. Start at the beginning
2. Compare each value to the key
3. Stop when you find the key or reach the end

**Cost:** **O(n)** worst case.

Linear search works on **any** order of data. You do not need a sorted list.

```cpp
#include <iostream>
#include <vector>

int linearSearch(const std::vector<int>& data, int key)
{
    for (int i{0}; i < static_cast<int>(data.size()); ++i)
    {
        if (data.at(i) == key)
        {
            return i;
        }
    }

    return -1;
}

int main()
{
    std::vector<int> scores{88, 42, 95, 73, 42, 60};

    int index{linearSearch(scores, 95)};
    if (index != -1)
    {
        std::cout << "Found 95 at index " << index << "\n";
    }

    index = linearSearch(scores, 99);
    if (index == -1)
    {
        std::cout << "99 not found\n";
    }

    return 0;
}
```

### The STL way: `std::find`

`std::find` does the same job. It returns an **iterator** to the first match, or **`end()`** if nothing matches.

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> scores{88, 42, 95, 73, 42, 60};

    auto it{std::find(scores.begin(), scores.end(), 95)};

    if (it != scores.end())
    {
        int index{static_cast<int>(it - scores.begin())};
        std::cout << "Found 95 at index " << index << "\n";
    }
    else
    {
        std::cout << "95 not found\n";
    }

    return 0;
}
```

> PROTIP: Use your own loop to learn the idea. Use `std::find` in real projects so you do not rewrite the same loop every time.

---

## Binary Search

**Binary search** requires a **sorted** list. Each step throws away half the remaining data.

1. Start at the middle index
2. Compare middle value to the key
3. If the key is smaller, repeat on the **left half**; if larger, repeat on the **right half**
4. Stop when found or the range is empty

**Cost:** **O(log n)**.

With about one million sorted items, binary search needs roughly **20 comparisons**. Linear search might need a million.

```cpp
#include <iostream>
#include <vector>

int binarySearch(const std::vector<int>& sortedData, int key)
{
    int low{0};
    int high{static_cast<int>(sortedData.size()) - 1};

    while (low <= high)
    {
        int mid{low + (high - low) / 2};
        int middleValue{sortedData.at(mid)};

        if (middleValue == key)
        {
            return mid;
        }

        if (key < middleValue)
        {
            high = mid - 1;
        }
        else
        {
            low = mid + 1;
        }
    }

    return -1;
}

int main()
{
    std::vector<int> sortedLevels{2, 5, 8, 12, 16, 23, 38, 56, 72, 91};

    int index{binarySearch(sortedLevels, 23)};
    if (index != -1)
    {
        std::cout << "Level 23 is at index " << index << "\n";
    }

    index = binarySearch(sortedLevels, 40);
    if (index == -1)
    {
        std::cout << "Level 40 not in table\n";
    }

    return 0;
}
```

### The STL way: `std::binary_search` and `std::lower_bound`

The STL gives you two common tools:

- **`std::binary_search`** returns `true` or `false` (is the value there?)
- **`std::lower_bound`** returns an iterator to the first element **not less than** the key (useful for insert position or exact lookup)

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> sortedLevels{2, 5, 8, 12, 16, 23, 38, 56, 72, 91};

    if (std::binary_search(sortedLevels.begin(), sortedLevels.end(), 23))
    {
        std::cout << "23 is in the table\n";
    }

    auto it{std::lower_bound(sortedLevels.begin(), sortedLevels.end(), 23)};
    if (it != sortedLevels.end() && *it == 23)
    {
        int index{static_cast<int>(it - sortedLevels.begin())};
        std::cout << "23 is at index " << index << "\n";
    }

    return 0;
}
```

> NOTE: Binary search needs sorted data first. That is one reason sorting shows up everywhere in games (leaderboards, spawn tables, inventory sorted by name).

---

## Sorting: Think It Through with Bubble Sort

Before jumping to library calls, walk through sorting like you might invent it yourself.

You have a list:

```
5  1  4  3  8  2  6  7
```

You look at the first two values. They are out of order (`5` before `1`). **Swap.**

```
1  5  4  3  8  2  6  7
```

Now what? **Check the next pair.** Compare `5` and `4`. Out of order. Swap.

```
1  4  5  3  8  2  6  7
```

Keep going: compare neighbors left to right, swap when the left one is bigger.

After **one full pass** across the list:

```
1  4  3  5  2  6  7  8
```

Something useful happened: the **biggest** value (`8`) ended up at the **end**. It bubbled there step by step.

Now what? **Do it again.** Run another pass. The next-largest values settle near the back.

```
1  3  2  4  5  6  7  8
```

Keep repeating until a full pass makes **zero swaps**. Then the list is sorted.

**How many passes?** In the worst case with `n` items, you may need about **`n - 1` passes**, and each pass scans most of the list. That nested work is **O(n²)**.

Computer scientists call this **bubble sort**. Large values bubble toward the end one pass at a time.

```cpp
#include <iostream>
#include <vector>

void bubbleSort(std::vector<int>& data)
{
    for (int pass{0}; pass < static_cast<int>(data.size()) - 1; ++pass)
    {
        bool swapped{false};

        for (int i{0}; i < static_cast<int>(data.size()) - 1 - pass; ++i)
        {
            if (data.at(i) > data.at(i + 1))
            {
                int temp{data.at(i)};
                data.at(i) = data.at(i + 1);
                data.at(i + 1) = temp;
                swapped = true;
            }
        }

        if (!swapped)
        {
            break;
        }
    }
}

int main()
{
    std::vector<int> values{5, 1, 4, 3, 8, 2, 6, 7};
    bubbleSort(values);

    for (int value : values)
    {
        std::cout << value << " ";
    }
    std::cout << "\n";

    return 0;
}
```

### Other classic sorts (short version)

**Selection sort:** each pass finds the minimum of the unsorted tail and swaps it into the next open slot. Still **O(n²)**, but it makes fewer swaps than bubble sort.

**Insertion sort:** take one unsorted item at a time and slide it left into its correct spot among the sorted prefix. Also **O(n²)** in the worst case, but it is fast on small or nearly sorted lists.

**Merge sort:** split the list in half recursively until pieces have one item, then merge pairs back together in order. **O(n log n)**.

**Quicksort:** pick a pivot, partition smaller values to one side and larger to the other, then recurse on each side. **O(n log n)** on average.

You do not need to code merge or quicksort by hand for this course. Know the idea and the cost.

### The STL way: `std::sort`

In real C++ you call the library:

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> values{5, 1, 4, 3, 8, 2, 6, 7};

    std::sort(values.begin(), values.end());

    for (int value : values)
    {
        std::cout << value << " ";
    }
    std::cout << "\n";

    return 0;
}
```

`std::sort` lives in `<algorithm>`. Pass **`begin()`** and **`end()`** to describe the range.

Most standard library implementations use a hybrid strategy (often **introsort**, based on quicksort). Last I checked, they switch to **insertion sort** for small sub-ranges because insertion sort is cheap on tiny lists.

> PREFERENCE: Write bubble sort once so the passes make sense. After that, call `std::sort`.

| Algorithm | Typical cost | Notes |
|-----------|--------------|-------|
| Bubble sort | O(n²) | Good for learning |
| Selection sort | O(n²) | Few swaps |
| Insertion sort | O(n²) | Fast on small lists |
| Merge sort | O(n log n) | Stable divide and conquer |
| Quicksort | O(n log n) avg | Fast in practice |
| `std::sort` | O(n log n) | What you use in projects |

---

## Random Number Generation (RNG)

### Why games need randomness

- **Statistical events** (critical hits, loot drops)
- **Variety** (enemy spawn positions, procedural levels)
- **Simulation** (particle spread, AI jitter)

Computers follow instructions. "Random" on a CPU is almost always **pseudo-random**: a formula that *looks* random unless you know the seed.

### Engine vs distribution

Whether you call **`rand()`** or **`std::mt19937`**, the raw output is the same kind of thing: a **large integer** from a formula. That number is rarely the value you actually want in a game.

You still have to **map** that raw value into the range or shape you need. The old way uses `%` and addition. The modern way uses a **distribution** object.

| Piece | Job |
|-------|-----|
| **Seed** | Starting point. Same seed, same sequence. |
| **Engine** | Produces raw integers (`rand()`, `mt19937`, etc.) |
| **Distribution** | Turns raw engine output into the range or curve you want |

---

## The old way: `rand()` and `%`

`rand()` from `<cstdlib>` is fine for toy programs. It is **not secure** (do not use it for passwords, gambling money, or anti-cheat). You also do extra work to squeeze values into a range:

```cpp
std::rand() % 10 + 1;   // hope this gives 1..10
```

Common problems:

- Seeding with the clock alone is guessable
- `%` can **bias** results when the modulus does not divide the generator range evenly

Suppose `rand()` only returns 1 through 5 (toy example):

```cpp
std::rand() % 3
```

| rand | % 3 |
|------|-----|
| 1    | 1   |
| 2    | 2   |
| 3    | 0   |
| 4    | 1   |
| 5    | 2   |

Value `0` appears **1/5** of the time. Values `1` and `2` each appear **2/5** of the time. Not uniform.

```cpp
#include <cstdlib>
#include <iostream>

int main()
{
    std::srand(12345);

    for (int trial{0}; trial < 10; ++trial)
    {
        int value{std::rand() % 6 + 1};
        std::cout << "d6: " << value << "\n";
    }

    return 0;
}
```

---

## The modern way: `<random>`

The `<random>` header separates **engine** from **distribution**:

- Seed once with **`std::random_device`** when you want a different sequence each run
- Keep one **`std::mt19937 rng`** and reuse it
- Use **`dieDist(rng)`**, **`lootDist(rng)`**, etc. to get values in the shape you want

`rng()` and `rand()` both give you a big raw integer. The distribution is what turns that into a die roll, a percentage, or a bell-curve stat.

### Uniform integers (most common in games)

```cpp
#include <iostream>
#include <random>

int main()
{
    std::random_device seedSource;
    std::mt19937 rng{seedSource()};

    std::uniform_int_distribution<int> dieDist{1, 6};
    std::uniform_int_distribution<int> d20Dist{1, 20};
    std::uniform_int_distribution<int> lootDist{1, 100};

    for (int i{0}; i < 5; ++i)
    {
        std::cout << "d6: " << dieDist(rng) << "\n";
    }

    std::cout << "d20: " << d20Dist(rng) << "\n";
    std::cout << "loot: " << lootDist(rng) << "\n";

    return 0;
}
```

Use a **fixed seed** when you want the same sequence every run (debugging, tests, replays):

```cpp
std::mt19937 rng{12345};
```

### Normal distribution (one example)

Most values cluster near the mean. Good for damage spread, aim error, or stat generation.

```cpp
#include <iostream>
#include <random>

int main()
{
    std::mt19937 rng{42};
    std::normal_distribution<double> damageDist{25.0, 5.0};

    for (int i{0}; i < 5; ++i)
    {
        std::cout << "Hit for " << damageDist(rng) << " damage\n";
    }

    return 0;
}
```

### Main distributions at a glance

| Distribution | Typical use | Example |
|--------------|-------------|---------|
| `uniform_int_distribution<int>` | Fair integer in `[min, max]` | `dieDist{1, 6}` then `dieDist(rng)` |
| `uniform_real_distribution<double>` | Decimal in `[min, max)` | `unitDist{0.0, 1.0}` for 0..1 |
| `bernoulli_distribution` | True/false with probability | `critDist{0.05}` for 5% crit chance |
| `normal_distribution<double>` | Bell curve around a mean | `damageDist{25.0, 5.0}` |

> NOTE: Seed once, then reuse the same `rng`. Creating a new engine every frame wastes work and can produce ugly patterns.

> PREFERENCE: Name objects for what they mean in the game: `rng`, `dieDist`, `lootDist`, not `e` and `d`.

---

## The Game Loop

Interactive games share the same skeleton:

1. **Initialize** once (SDL, window, objects)
2. **Loop** until quit:
   - Read user input
   - Call **`update`** on each game object
   - Call **`draw`** on each game object
   - Present the frame and pause for a fixed frame rate
3. **Cleanup**

Store every object in a container of a **base type**. The base declares **`virtual update`** and **`virtual draw`**. Derived classes override behavior. Each frame, one loop updates and draws everything without knowing concrete types.

```cpp
#include <iostream>
#include <memory>
#include <vector>

class GameObject
{
public:
    virtual ~GameObject() = default;

    virtual void update(float deltaTimeSeconds) = 0;
    virtual void draw() const = 0;
};

class Player : public GameObject
{
public:
    void update(float deltaTimeSeconds) override
    {
        x += 120.0f * deltaTimeSeconds;
    }

    void draw() const override
    {
        std::cout << "Player at x=" << x << "\n";
    }

private:
    float x{0.0f};
};

class Enemy : public GameObject
{
public:
    void update(float deltaTimeSeconds) override
    {
        x -= 80.0f * deltaTimeSeconds;
    }

    void draw() const override
    {
        std::cout << "Enemy at x=" << x << "\n";
    }

private:
    float x{400.0f};
};

int main()
{
    std::vector<std::unique_ptr<GameObject>> objects{};
    objects.push_back(std::make_unique<Player>());
    objects.push_back(std::make_unique<Enemy>());

    const float deltaTime{1.0f / 60.0f};

    for (int frame{0}; frame < 3; ++frame)
    {
        std::cout << "--- frame " << frame << " ---\n";

        for (std::unique_ptr<GameObject>& object : objects)
        {
            object->update(deltaTime);
        }

        for (const std::unique_ptr<GameObject>& object : objects)
        {
            object->draw();
        }
    }

    return 0;
}
```

> PREFERENCE: Put **`virtual`** on the base functions and **`override`** on derived versions. Give the base a **`virtual` destructor** when objects live behind base pointers.

---

## SDL2 game loop

The console example above is the same pattern with SDL2: poll events, update objects, draw objects, cap the frame rate.

This demo uses two colored circles (player and wanderer) stored in a `std::vector<std::unique_ptr<GameObject>>`.

```sdl2
#include <SDL2/SDL.h>
#include <SDL2/SDL2_gfxPrimitives.h>
#include <memory>
#include <random>
#include <vector>

class GameObject
{
public:
    virtual ~GameObject() = default;

    virtual void update(float deltaTimeSeconds) = 0;
    virtual void draw(SDL_Renderer* renderer) const = 0;

protected:
    float x{320.0f};
    float y{240.0f};
};

class Player : public GameObject
{
public:
    Player()
    {
        x = 120.0f;
        y = 240.0f;
    }

    void update(float deltaTimeSeconds) override
    {
        const float speed{220.0f};

        if (moveLeft)
        {
            x -= speed * deltaTimeSeconds;
        }
        if (moveRight)
        {
            x += speed * deltaTimeSeconds;
        }
        if (moveUp)
        {
            y -= speed * deltaTimeSeconds;
        }
        if (moveDown)
        {
            y += speed * deltaTimeSeconds;
        }

        if (x < 20.0f)
        {
            x = 20.0f;
        }
        if (x > 620.0f)
        {
            x = 620.0f;
        }
        if (y < 20.0f)
        {
            y = 20.0f;
        }
        if (y > 460.0f)
        {
            y = 460.0f;
        }
    }

    void draw(SDL_Renderer* renderer) const override
    {
        filledCircleRGBA(
            renderer,
            static_cast<Sint16>(x),
            static_cast<Sint16>(y),
            18,
            80,
            220,
            120,
            255);
    }

    void handleKeyDown(SDL_Keycode key)
    {
        if (key == SDLK_LEFT || key == SDLK_a)
        {
            moveLeft = true;
        }
        if (key == SDLK_RIGHT || key == SDLK_d)
        {
            moveRight = true;
        }
        if (key == SDLK_UP || key == SDLK_w)
        {
            moveUp = true;
        }
        if (key == SDLK_DOWN || key == SDLK_s)
        {
            moveDown = true;
        }
    }

    void handleKeyUp(SDL_Keycode key)
    {
        if (key == SDLK_LEFT || key == SDLK_a)
        {
            moveLeft = false;
        }
        if (key == SDLK_RIGHT || key == SDLK_d)
        {
            moveRight = false;
        }
        if (key == SDLK_UP || key == SDLK_w)
        {
            moveUp = false;
        }
        if (key == SDLK_DOWN || key == SDLK_s)
        {
            moveDown = false;
        }
    }

private:
    bool moveLeft{false};
    bool moveRight{false};
    bool moveUp{false};
    bool moveDown{false};
};

class Wanderer : public GameObject
{
public:
    Wanderer(std::mt19937& rngRef)
        : rng{rngRef}
        , unitDist{0.0f, 1.0f}
    {
        x = 480.0f;
        y = 180.0f;
    }

    void update(float deltaTimeSeconds) override
    {
        x += (unitDist(rng) - 0.5f) * 260.0f * deltaTimeSeconds;
        y += (unitDist(rng) - 0.5f) * 260.0f * deltaTimeSeconds;

        if (x < 20.0f)
        {
            x = 20.0f;
        }
        if (x > 620.0f)
        {
            x = 620.0f;
        }
        if (y < 20.0f)
        {
            y = 20.0f;
        }
        if (y > 460.0f)
        {
            y = 460.0f;
        }
    }

    void draw(SDL_Renderer* renderer) const override
    {
        filledCircleRGBA(
            renderer,
            static_cast<Sint16>(x),
            static_cast<Sint16>(y),
            14,
            240,
            180,
            60,
            255);
    }

private:
    std::mt19937& rng;
    std::uniform_real_distribution<float> unitDist;
};

int main(int, char**)
{
    SDL_Init(SDL_INIT_VIDEO);

    SDL_Window* window = SDL_CreateWindow(
        "Module 02 loop",
        SDL_WINDOWPOS_CENTERED,
        SDL_WINDOWPOS_CENTERED,
        640,
        480,
        SDL_WINDOW_SHOWN);

    SDL_Renderer* renderer = SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED);

    std::random_device seedSource;
    std::mt19937 rng{seedSource()};

    std::vector<std::unique_ptr<GameObject>> objects{};
    objects.push_back(std::make_unique<Player>());
    objects.push_back(std::make_unique<Wanderer>(rng));

    bool running{true};
    SDL_Event event{};
    Uint32 lastTicks{SDL_GetTicks()};

    while (running)
    {
        Uint32 now{SDL_GetTicks()};
        float deltaTime{static_cast<float>(now - lastTicks) / 1000.0f};
        lastTicks = now;

        while (SDL_PollEvent(&event))
        {
            if (event.type == SDL_QUIT)
            {
                running = false;
            }
            else if (event.type == SDL_KEYDOWN)
            {
                if (auto* playerObject = dynamic_cast<Player*>(objects.at(0).get()))
                {
                    playerObject->handleKeyDown(event.key.keysym.sym);
                }
            }
            else if (event.type == SDL_KEYUP)
            {
                if (auto* playerObject = dynamic_cast<Player*>(objects.at(0).get()))
                {
                    playerObject->handleKeyUp(event.key.keysym.sym);
                }
            }
        }

        for (std::unique_ptr<GameObject>& object : objects)
        {
            object->update(deltaTime);
        }

        SDL_SetRenderDrawColor(renderer, 24, 28, 38, 255);
        SDL_RenderClear(renderer);

        for (const std::unique_ptr<GameObject>& object : objects)
        {
            object->draw(renderer);
        }

        SDL_RenderPresent(renderer);
        SDL_Delay(16);
    }

    SDL_DestroyRenderer(renderer);
    SDL_DestroyWindow(window);
    SDL_Quit();

    return 0;
}
```

Use **arrow keys** or **WASD** to move the green player. The gold circle wanders using `unitDist(rng)`.

### Frame loop checklist

Each pass through the `while (running)` loop is one **frame**:

1. Compute **`deltaTime`** (seconds since last frame)
2. **Poll events** (`SDL_PollEvent`) and update input state
3. Call **`update(deltaTime)`** on every object in the container
4. **Clear** the back buffer, call **`draw(renderer)`** on every object, then **`SDL_RenderPresent`**
5. **`SDL_Delay(16)`** to cap near 60 FPS and yield in browser playgrounds

> PROTIP: Movement uses `speed * deltaTime` so behavior stays consistent even if the frame rate changes.

---

## Try it now

### Exercise 1: Search a high-score table

Prompt: Fill a sorted `std::vector<int>` with ten scores. Use `std::binary_search` to check whether a score exists, then `std::lower_bound` to print its index.

:::details Hint

Sort once with `std::sort`, then call the STL functions from `<algorithm>`.

:::

### Exercise 2: Fair loot roll

Prompt: Create `std::mt19937 rng` and `std::uniform_int_distribution<int> lootDist{1, 100}`. Roll ten times with `lootDist(rng)`.

:::details Solution

**Reasoning:** The engine owns the sequence. The distribution owns the range. Call `lootDist(rng)` each time instead of re-seeding.

```cpp
#include <iostream>
#include <random>

int main()
{
    std::mt19937 rng{12345};
    std::uniform_int_distribution<int> lootDist{1, 100};

    for (int i{0}; i < 10; ++i)
    {
        std::cout << lootDist(rng) << "\n";
    }

    return 0;
}
```

:::

### Exercise 3: Add a third object

Prompt: In the SDL2 demo, derive a new class from `GameObject` (for example a pulsing dot) and push it into the `objects` vector. It should update and draw without changing the main loop.

:::details Hint

Only override `update` and `draw`. The existing loops already call those on every `unique_ptr<GameObject>`.

:::
