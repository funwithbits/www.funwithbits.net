---
layout: post
title: "Handling game states"
date: 2017-01-28 22:52:39 -0200
comments: true
categories: [game programming]
---

_Prerequisite_: basic C++ and game programming knowledge

When you start any game, you expect to see a loading screen, followed by the
main menu which has a button that allows you to play the game. When you start
playing the game, it's also expected that you'll be able to go back to main
menu. All these different stages of the game are known as game states.

Handling game states is a very difficult task, specially to newbies to game
programming like myself. Today, I was looking for an efficient way to switch
back and forth between all available states of my simple game.

The simplest way to do it would be using a switch statement, as follow:
``` cpp
enum game_states {
    MENU,
    GAME,
    ...
}

void game::update() {
    switch(_current_game_state) {
        case MENU:
            // show game options
            // update current state to PLAY if user clicks on play button.
            ...
            break;
        case PLAY:
            ...
    }
}

```

That's indeed very simple, but it would be a nightmare to maintain this code
when the number of states increases considerably. It turned out that a
[Finite State Machine (FSM)](http://en.wikipedia.org/wiki/Finite-state_machine)
is exactly what I was looking for. It's said that FSM is an abstract machine
that can be in exactly one of a finite number of states at any given time.

To implement a state machine that handle different types of game states, I took
advantage of polymorphism. Each game state will derive from an abstract class
called game_state; follow its definition:
``` cpp
class game_state {
public:
    virtual void on_enter() = 0;
    virtual void on_exit() = 0;
    virtual void update() = 0;
};
```

The first two methods will be used for loading and cleaning the game state,
respectively. **game_state::update()** will be used for a given state to react
to user's input and possibly switch to another state. For example, when an user
clicks on play button, the state machine will switch from menu to play state.

Now our state machine will be able to work with all different types of game
states in the same way. To make the transition between stages more efficient,
the machine will work in the same way a stack does. For example, a new state
will be pushed to the back of container storing the game states. And more
importantly, the active state is the one that was last pushed to the container.
That's how my implementation of game state machine turned out:

``` cpp
class game_state_machine {
    std::vector<std::unique_ptr<game_state>> _game_states;
public:
    void push(std::unique_ptr<game_state> state) {
        state->on_enter();
        _game_states.push_back(std::move(state));
    }

    void pop() {
        if (!_game_states.empty()) {
            _game_states.back()->on_exit();
            _game_states.pop_back();

            if (!_game_states.empty()) {
                _game_states.back()->on_enter();
            }
        }
    }

    void update() {
        if(!_game_states.empty()) {
            _game_states.back()->update();
        }
    }
};
```

Note that **game_state_machine::update()** will only call update on behalf of
the active state, and that's essential for the machine to work as expected.

I showed the implementation of abstract class for game state, but it's also
important to understand how an actual game state could be implemented by
deriving from it. Check it out:
``` cpp
class menu : public game_state {
public:
    void on_enter() override {
        // load menu sprites
        ...
    }
    void on_exit() override {
        // clean up goes here
        ...
    }

    void update() override {
        // if user clicked on play button, switch to play state.
        if ( ... ) {
            ...
            game_state_machine.push(std::make_unique<play>());
        }
    }
};
```

Very easy, no?! If we're in play state, and want to go back to menu, all we
need to do is to call **game_state_machine::pop()**.

This was the most efficient way I found to handle game states in my own game.
If you know a better way to do it, please leave a comment!

**PS**: the comment section only shows up when you click on the blog post's
title.


See you,


