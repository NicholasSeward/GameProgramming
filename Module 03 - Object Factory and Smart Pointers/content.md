# Object Factory and Smart Pointers

**CPSI 27703: Intro to Game Programming**

This module covers three ideas that make a game engine grow without turning into a mess of `new` and `switch` statements:

1. **Object factories** (create a fully initialized object with one call)
2. **Asset / factory library** (look up the right factory by name)
3. **Smart pointers** (ownership without manual `delete`)

> PREFERENCE: Good game code is about **readability**, **reusability**, and **abstraction**.

---

## The Factory Pattern

A **factory** creates a fully initialized object and hands it back. Callers do not care how construction works. They ask for an object and get one.

What it buys you:

- **Readability:** one line instead of a pile of setup code
- **Abstraction:** callers talk to a base type, not concrete classes
- **Reusability:** the same factory can be reused wherever that object is needed

```cpp
#include <iostream>
#include <memory>
#include <string>

class Vehicle
{
public:
    virtual ~Vehicle() = default;
    virtual void describe() const = 0;
};

class Car : public Vehicle
{
public:
    Car(const std::string& color)
        : color{color}
    {
    }

    void describe() const override
    {
        std::cout << "Car (" << color << ")\n";
    }

private:
    std::string color;
};

std::unique_ptr<Vehicle> createCar(const std::string& color)
{
    return std::make_unique<Car>(color);
}

int main()
{
    std::unique_ptr<Vehicle> vehicle{createCar("red")};
    vehicle->describe();
    return 0;
}
```

That is the idea: **one call**, fully built object, ready to use.

---

## Factories Mirror the Object Hierarchy

If your game objects form a hierarchy, the factories usually form the **same** hierarchy:

| Objects | Factories |
|---------|-----------|
| `Object` | `ObjectFactory` |
| `Car` | `CarFactory` |
| `Plane` | `PlaneFactory` |

```
Object                    ObjectFactory
  ├── Car                   ├── CarFactory
  └── Plane                 └── PlaneFactory
```

Each concrete factory knows how to build **one** concrete type. The base factory type is what the rest of the engine talks to.

```cpp
#include <iostream>
#include <memory>
#include <string>

class Object
{
public:
    virtual ~Object() = default;
    virtual void update() = 0;
    virtual void draw() const = 0;
};

class Car : public Object
{
public:
    Car(const std::string& name)
        : name{name}
    {
    }

    void update() override
    {
        std::cout << name << " drives\n";
    }

    void draw() const override
    {
        std::cout << "Draw car: " << name << "\n";
    }

private:
    std::string name;
};

class Plane : public Object
{
public:
    Plane(const std::string& name)
        : name{name}
    {
    }

    void update() override
    {
        std::cout << name << " flies\n";
    }

    void draw() const override
    {
        std::cout << "Draw plane: " << name << "\n";
    }

private:
    std::string name;
};

class ObjectFactory
{
public:
    virtual ~ObjectFactory() = default;
    virtual std::unique_ptr<Object> create(const std::string& name) const = 0;
};

class CarFactory : public ObjectFactory
{
public:
    std::unique_ptr<Object> create(const std::string& name) const override
    {
        return std::make_unique<Car>(name);
    }
};

class PlaneFactory : public ObjectFactory
{
public:
    std::unique_ptr<Object> create(const std::string& name) const override
    {
        return std::make_unique<Plane>(name);
    }
};

int main()
{
    CarFactory carFactory{};
    PlaneFactory planeFactory{};

    std::unique_ptr<Object> car{carFactory.create("Roadster")};
    std::unique_ptr<Object> plane{planeFactory.create("Skyhawk")};

    car->update();
    plane->update();
    car->draw();
    plane->draw();

    return 0;
}
```

> NOTE: Each factory instance has its own memory. Later you can store configuration on a factory (default stats, sprite names, etc.) without changing call sites.

---

## How Do We Get the Right Factory?

You load a level file. It says `"Car"` or `"Plane"`. Somehow you must pick the matching factory.

### Approach 1: `switch` (messy)

One option is a big `switch` on a type code. That forces you to:

- Store object names as `char` / `int`, or convert strings to an `enum`
- Add a new `case` for every new object type
- Touch the same function every time the game grows

```cpp
#include <iostream>
#include <memory>
#include <string>

enum class ObjectType
{
    Car,
    Plane
};

class Object
{
public:
    virtual ~Object() = default;
    virtual void describe() const = 0;
};

class Car : public Object
{
public:
    void describe() const override
    {
        std::cout << "Car\n";
    }
};

class Plane : public Object
{
public:
    void describe() const override
    {
        std::cout << "Plane\n";
    }
};

std::unique_ptr<Object> createBySwitch(ObjectType type)
{
    switch (type)
    {
    case ObjectType::Car:
        return std::make_unique<Car>();
    case ObjectType::Plane:
        return std::make_unique<Plane>();
    }

    return nullptr;
}

int main()
{
    std::unique_ptr<Object> object{createBySwitch(ObjectType::Car)};
    object->describe();
    return 0;
}
```

This works for a tiny demo. It does **not** scale. New object types mean new cases, more enums, and more conversion code from XML or JSON strings.

### Approach 2: `map<std::string, ObjectFactory*>`

Use a **string key** that matches the name in your data file:

```cpp
std::map<std::string, ObjectFactory*> factories;
```

Now the level file can say `"Car"` and you look up the factory by that string. No enum conversion. No giant `switch`.

Where do you put the map?

- Inside an `Engine` class? Sure.
- But you will need maps for **more than objects** later (textures, sounds, fonts, configs).

That is where a **Library** class comes in.

---

## The Library Class

A **library** owns a map of name → factory (or name → asset). Register once at startup. Look up by string forever after.

```cpp
#include <iostream>
#include <map>
#include <memory>
#include <string>
#include <vector>

class Object
{
public:
    virtual ~Object() = default;
    virtual void describe() const = 0;
};

class Car : public Object
{
public:
    Car(const std::string& name)
        : name{name}
    {
    }

    void describe() const override
    {
        std::cout << "Car: " << name << "\n";
    }

private:
    std::string name;
};

class Plane : public Object
{
public:
    Plane(const std::string& name)
        : name{name}
    {
    }

    void describe() const override
    {
        std::cout << "Plane: " << name << "\n";
    }

private:
    std::string name;
};

class ObjectFactory
{
public:
    virtual ~ObjectFactory() = default;
    virtual std::unique_ptr<Object> create(const std::string& name) const = 0;
};

class CarFactory : public ObjectFactory
{
public:
    std::unique_ptr<Object> create(const std::string& name) const override
    {
        return std::make_unique<Car>(name);
    }
};

class PlaneFactory : public ObjectFactory
{
public:
    std::unique_ptr<Object> create(const std::string& name) const override
    {
        return std::make_unique<Plane>(name);
    }
};

class Library
{
public:
    void add(const std::string& key, std::unique_ptr<ObjectFactory> factory)
    {
        factories[key] = std::move(factory);
    }

    std::unique_ptr<Object> create(const std::string& key, const std::string& name) const
    {
        auto it{factories.find(key)};
        if (it == factories.end())
        {
            return nullptr;
        }

        return it->second->create(name);
    }

private:
    std::map<std::string, std::unique_ptr<ObjectFactory>> factories;
};

int main()
{
    Library library{};
    library.add("Car", std::make_unique<CarFactory>());
    library.add("Plane", std::make_unique<PlaneFactory>());

    std::vector<std::unique_ptr<Object>> objects{};
    objects.push_back(library.create("Car", "Roadster"));
    objects.push_back(library.create("Plane", "Skyhawk"));

    for (const std::unique_ptr<Object>& object : objects)
    {
        object->describe();
    }

    return 0;
}
```

### The power play

Once the library is loaded, spawning from data looks like one idea:

```
objects.push_back(
    library.find(objectName)->second->create(objectData)
);
```

Or, cleaner with a helper on the library:

```cpp
objects.push_back(library.create(objectName, objectData));
```

That is the payoff:

1. Level file names an object type as a string
2. Library finds the factory
3. Factory builds a fully initialized object
4. You push it into the game object list

No `switch`. No hard-coded type list in the spawn loop.

> PREFERENCE: Keep registration in one place at startup. Keep spawning as a short lookup + `create` call.

---

## Asset Library (Same Idea, Different Payload)

An **asset library** uses the same pattern, but stores loaded resources instead of factories:

| Key | Value |
|-----|-------|
| `"player.png"` | texture pointer / handle |
| `"jump.wav"` | sound chunk |
| `"hud.ttf"` | font |

You load assets once, store them by name, and look them up when objects are created.

```cpp
#include <iostream>
#include <map>
#include <string>

class Texture
{
public:
    Texture(const std::string& path)
        : path{path}
    {
        std::cout << "Loaded texture: " << path << "\n";
    }

    const std::string& getPath() const
    {
        return path;
    }

private:
    std::string path;
};

class AssetLibrary
{
public:
    void add(const std::string& key, Texture texture)
    {
        textures.insert_or_assign(key, texture);
    }

    Texture* find(const std::string& key)
    {
        auto it{textures.find(key)};
        if (it == textures.end())
        {
            return nullptr;
        }

        return &it->second;
    }

private:
    std::map<std::string, Texture> textures;
};

int main()
{
    AssetLibrary assets{};
    assets.add("player", Texture{"assets/player.png"});
    assets.add("enemy", Texture{"assets/enemy.png"});

    Texture* playerTex{assets.find("player")};
    if (playerTex != nullptr)
    {
        std::cout << "Use " << playerTex->getPath() << "\n";
    }

    return 0;
}
```

Object factories often **ask the asset library** for textures and sounds while `create` runs. That keeps resource loading out of every game object class.

---

## Smart Pointers

Dynamic memory (`new` / `delete`) is dangerous:

- Forget `delete` → leak
- `delete` twice → crash
- Keep a dangling pointer → undefined behavior

**Smart pointers** wrap ownership so cleanup happens automatically. They also make ownership visible in the type system.

Types you will use:

| Type | Ownership | Copyable? |
|------|-----------|-----------|
| `std::unique_ptr` | Exactly one owner | No (move only) |
| `std::shared_ptr` | Shared, reference counted | Yes |
| `std::weak_ptr` | Observes a `shared_ptr` object, does not own | Yes |

> PREFERENCE: Start with `unique_ptr`. Reach for `shared_ptr` only when multiple owners must keep the object alive.

---

## Unique Pointers

A **`std::unique_ptr`** means: **only one thing owns this object**.

- Copy constructor is deleted (you cannot accidentally share ownership)
- Transfer ownership with **`std::move`**
- Borrow without transferring ownership with **`.get()`**
- Almost **no overhead** compared to a raw pointer

Use `unique_ptr` unless you truly need the object to survive after one owner dies.

```cpp
#include <iostream>
#include <memory>

class Sprite
{
public:
    Sprite(int id)
        : id{id}
    {
        std::cout << "Sprite " << id << " created\n";
    }

    ~Sprite()
    {
        std::cout << "Sprite " << id << " destroyed\n";
    }

    void draw() const
    {
        std::cout << "Drawing sprite " << id << "\n";
    }

private:
    int id;
};

void borrow(Sprite* sprite)
{
    if (sprite != nullptr)
    {
        sprite->draw();
    }
}

void takeOwnership(std::unique_ptr<Sprite> sprite)
{
    sprite->draw();
}

int main()
{
    std::unique_ptr<Sprite> sprite{std::make_unique<Sprite>(1)};
    sprite->draw();

    borrow(sprite.get());                 // still owned by sprite
    takeOwnership(std::move(sprite));     // ownership transferred

    if (!sprite)
    {
        std::cout << "sprite is empty after move\n";
    }

    return 0;
}
```

Creating one:

```cpp
#include <memory>

std::unique_ptr<Sprite> spritePtr = std::make_unique<Sprite>(constructorArgs);
```

> NOTE: `std::make_unique` is C++14. Prefer it over wrapping `new` by hand.
>
> PREFERENCE: Prefer `unique_ptr` for game objects in a scene list. It matches "the engine owns this object."

---

## Shared Pointers

A **`std::shared_ptr`** uses a small **manager** (control block) that tracks how many owners exist.

- Copying a `shared_ptr` increases the count
- Destroying a `shared_ptr` decreases the count
- Memory is deleted when the **shared count hits 0**
- That manager adds a little **overhead**

Use shared pointers only when the object should die only after **all** owners are gone.

```cpp
#include <iostream>
#include <memory>

class Resource
{
public:
    Resource()
    {
        std::cout << "Resource created\n";
    }

    ~Resource()
    {
        std::cout << "Resource destroyed\n";
    }
};

int main()
{
    std::shared_ptr<Resource> a{std::make_shared<Resource>()};
    std::cout << "count: " << a.use_count() << "\n";

    {
        std::shared_ptr<Resource> b{a};
        std::cout << "count: " << a.use_count() << "\n";
    }

    std::cout << "count after b died: " << a.use_count() << "\n";
    return 0;
}
```

Creating one:

```cpp
#include <memory>

std::shared_ptr<Resource> sharedPtr = std::make_shared<Resource>(constructorArgs);
```

### Weak pointers

A **`std::weak_ptr`** can observe an object managed by `shared_ptr` **without** keeping it alive.

- Does **not** increase the shared count
- Use when you need access, but **not ownership**
- Call **`lock()`** before use to get a temporary `shared_ptr` (or empty if the object is gone)

```cpp
#include <iostream>
#include <memory>

int main()
{
    std::shared_ptr<int> sharedValue{std::make_shared<int>(42)};
    std::weak_ptr<int> weakValue{sharedValue};

    if (std::shared_ptr<int> locked{weakValue.lock()})
    {
        std::cout << "alive: " << *locked << "\n";
    }

    sharedValue.reset();

    if (weakValue.expired())
    {
        std::cout << "object is gone\n";
    }

    return 0;
}
```

> PREFERENCE: Use `weak_ptr` for back-links (parent pointer, observer lists) so you do not create ownership cycles.

---

## Which Smart Pointer Should I Use?

| Situation | Choice |
|-----------|--------|
| One clear owner (engine owns objects) | `unique_ptr` |
| Need to hand temporary access | `.get()` or a reference |
| Multiple owners must keep it alive | `shared_ptr` |
| Want to observe without owning | `weak_ptr` |

In this course, most scene objects should live in:

```cpp
std::vector<std::unique_ptr<Object>> objects;
```

Factories should usually **return** `std::unique_ptr<Object>`. That makes ownership transfer explicit and automatic.

---

## Putting It Together

A typical spawn path for this course:

1. Register factories in a **Library** at startup
2. Read an object name from config / XML
3. Look up the factory
4. `create(...)` returns a `unique_ptr<Object>`
5. Push it into the game object container
6. Game loop calls `update` / `draw` through the base type

```cpp
objects.push_back(library.create(objectName, objectData));
```

That one line is the whole point of this module.

---

## Try it now

### Exercise 1: Add a third factory

Prompt: Add a `Boat` class and a `BoatFactory`. Register it in the library under `"Boat"`, then create one and call `describe()`.

:::details Hint

Mirror `CarFactory`. The library lookup string must match the key you register.

:::

### Exercise 2: Move ownership

Prompt: Create a `unique_ptr<Car>`, pass it into a function that takes `std::unique_ptr<Car>` by value, and confirm the original pointer is empty afterward.

:::details Solution

**Reasoning:** `unique_ptr` cannot be copied. `std::move` transfers ownership into the function parameter.

```cpp
#include <iostream>
#include <memory>

class Car
{
public:
    void honk() const
    {
        std::cout << "beep\n";
    }
};

void takeCar(std::unique_ptr<Car> car)
{
    car->honk();
}

int main()
{
    std::unique_ptr<Car> car{std::make_unique<Car>()};
    takeCar(std::move(car));

    if (!car)
    {
        std::cout << "ownership moved\n";
    }

    return 0;
}
```

:::

### Exercise 3: Shared then weak

Prompt: Make a `shared_ptr<int>`, copy it so `use_count()` is 2, store a `weak_ptr`, reset both shared owners, then print whether the weak pointer is expired.

:::details Hint

After both shared owners are gone, `weakValue.expired()` should be `true`.

:::
