# Animation-Frog

### animation_frog.hpp
``` Cpp
#include "app/utility/animation_component.hpp"
#include "app/utility/level_information.hpp"

#include <unordered_map>
#include <vector>

using namespace std;

class AnimationFrog : public AnimationComponent {
  public:
    AnimationFrog(GameObject *game_object, const string &sap_file, int Frog_FPS);
    virtual ~AnimationFrog();

    void set_next_node();
    void update_animation_frog(int velocity_x_current, int velocity_x_previous, float delta_time);
    void update_time_passed(float delta_time);
    string get_sprite_name();

  protected:
    string animation_node;
    string transition;

    // we don't want the frog the change from animation node every frame
    // IDEA: time_treshold = 1 / FPS; time_passed is a sum of all previous delta_times and is reset to 0 when time_passed >= time_treshold --> then the animation node is updated
    float time_treshold;
    float time_passed;

    unordered_map<string, unordered_map<string, string>> animation_structure;
    unordered_map<string, string> node_to_sprite_structure;

    int velocity_x_current;
    int velocity_y_current;
    int velocity_x_previous;
    int velocity_y_previous;
    bool contact_to_ground_current;
    bool contact_to_ground_previous;
};
```

### animation_frog.cpp

``` Cpp
#include "app/utility/animation_frog.hpp"

AnimationFrog::AnimationFrog(GameObject *game_object, const string &sap_file, int Frog_FPS) : AnimationComponent(game_object) {
    // node initializeren op IDLE_0
    this->animation_node = "IDLE_0";

    // AnimationFrog bezit de methodes construct_animation_structure en construct_node_to_sprite_structure omdat AnimationFrog overerft van AnimationComponent
    // construct_animation_structure en construct_node_to_sprite_structure geven de woordenboeken terug
    // ik heb alle andere functies van AnimationComponent in commentaar gezet
    this->animation_structure = this->construct_animation_structure(sap_file);
    this->node_to_sprite_structure = this->construct_node_to_sprite_structure(sap_file);

    this->time_treshold = 1 / ((float)Frog_FPS);
    this->time_passed = 0;
}

// geen idee wat de destructor zal inhouden
AnimationFrog::~AnimationFrog() {}

void AnimationFrog::set_next_node() {

    if (this->velocity_x_previous == 0 && this->velocity_x_current != 0) {
        this->transition = "MOVING";
        this->animation_node = this->animation_structure[this->animation_node][this->transition];
        this->time_passed = 0;
    }

    else if (this->velocity_x_previous != 0 && this->velocity_x_current == 0) {
        this->transition = "IDLE";
        this->animation_node = this->animation_structure[this->animation_node][this->transition];
        this->time_passed = 0;
    }

    else {
        this->transition = "TIME";
        if (this->time_passed >= this->time_treshold) {
            // update de animation_node aan de hand van de transitie
            this->animation_node = this->animation_structure[this->animation_node][this->transition];
            this->time_passed = 0;
        }
    }
}

string AnimationFrog::get_sprite_name() {
    string sprite_name = this->node_to_sprite_structure[this->animation_node];
    return sprite_name;
}

void AnimationFrog::update_animation_frog(int velocity_x_current, int velocity_x_previous, float delta_time) {
    this->time_passed += delta_time;
    this->velocity_x_current = velocity_x_current;
    this->velocity_x_previous = velocity_x_previous;
}
```

### frog_test.hpp

``` Cpp
#ifndef FROG_TEST_HPP
#define FROG_TEST_HPP

#include "app/renderer/AnimationRenderer.hpp"
#include "app/utility/animation_frog.hpp"
#include "core/game_object/game_object.hpp"
#include "core/input/input_manager.hpp"

class FrogTest : public GameObject {
  public:
    FrogTest(int Frog_FPS);
    ~FrogTest();
    void update(float delta_time);
    void velocity_update();
    void position_update(float delta_time);

  private:
    int velocity_x_current;
    int velocity_x_previous;
    int speed;
    
    Vector2 position;
    string sprite_name;

    AnimationRenderer *frog_renderer;
    AnimationFrog *frog_animation;
};

#endif
```

### frog_test.cpp

``` Cpp
#include "app/game_object/frog_test.hpp"

FrogTest::FrogTest(int Frog_FPS) {
    this->velocity_x_current = 0;
    this->velocity_x_previous = 0;

    Vector2 size = {64, 64};
    this->position = {500, 500};

    this->frog_animation = add_component<AnimationFrog>("assets/anim/frog.sap", Frog_FPS);
    this->frog_renderer = add_component<AnimationRenderer>("assets/sprites/frog.png", "assets/sprites/frog.sht", false, size, this->position);
}

FrogTest::~FrogTest() {}

void FrogTest::update(float delta_time) {
    velocity_update();
    position_update(delta_time);

    this->frog_animation->update_animation_frog(this->velocity_x_current, this->velocity_x_previous, delta_time);
    this->sprite_name = this->frog_animation->get_sprite_name();

    this->frog_renderer->set_location(sprite_name);
    this->frog_renderer->set_position(this->position);

    this->velocity_x_previous = this->velocity_x_current;
}

void FrogTest::velocity_update() {
    if (InputManager::get_instance().is_action_down("LEFT")) {
        this->velocity_x_current = -250;
    } else if (InputManager::get_instance().is_action_down("RIGHT")) {
        this->velocity_x_current = 250;
    } else {
        this->velocity_x_current = 0;
    }
}

void FrogTest::position_update(float delta_time) {
    this->position.x += velocity_x_current * delta_time;
}
```

### main.cpp

``` Cpp
    const int windowWidth = 1080;
    const int windowHeight = 720;

    SetConfigFlags(FLAG_WINDOW_RESIZABLE);
    InitWindow(windowWidth, windowHeight, "Frog Test");
    SetWindowMinSize(320, 240);

    SetTargetFPS(240);

    Engine &e = Engine::get_instance();

    AnimationSystem *animation_system = new AnimationSystem("ANIMATION_SYSTEM");
    e.add_system_before("RENDERING_SYSTEM", animation_system);

    int Frog_FPS = 15;
    FrogTest frog_test(Frog_FPS);

    while (!WindowShouldClose()) { // Detect window close button or ESC key
        e.update();
    }

    CloseWindow(); // Close window and OpenGL context
```
