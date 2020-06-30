+++
title = "Adventures in Rust: A Basic 2D Game"

[taxonomies]
tags = ["rustlang", "gamedev"]

[extra]
location = "San Francisco"
+++

In an effort to do more fun side projects, I've been learning [Rust][rust],
a wonderful systems programming language developed by the Mozilla
Foundation. It's been a while since I've touched a compiled language as my
day-to-day often deals with Python and Javascript variants. I was inspired
after seeing a lot of [interesting][rust-2017] [articles][quantum] about
Rust usage and decided to dive into learning Rust by creating a very basic 2D
game, inspired by the classic [Defender][defender] arcade game.

<!-- more -->

[rust]: https://rust-lang.org
[rust-2017]: https://blog.rust-lang.org/2017/12/21/rust-in-2017.html
[quantum]: https://blog.rust-lang.org/2017/11/14/Fearless-Concurrency-In-Firefox-Quantum.html
[defender]: https://en.wikipedia.org/wiki/Defender_(1981_video_game)

Note that to keep things relatively concise this blog post walks through
the major portions of the codebase but does skip over some of the
implementation details. If you're interested in walking through the code
yourself, the final result of this blog post is available on [here on
Github][repo].

[repo]: https://github.com/a5huynh/defender-game

And finally, if you're interested in jumping into Rust for yourself, here
are some resources I found immensely useful:
* “[The Rust Programming Language][rust-book]” Book (2nd Edition)
* [Rust Subreddit][subreddit].
* [Rust web playground][rust-play].

[rust-book]: https://doc.rust-lang.org/book/second-edition/ch01-00-introduction.html
[subreddit]: https://reddit.com/r/rust
[rust-play]: https://play.rust-lang.org/


## Getting Started

Alright, let's get down to business!

First off, I opted to use a framework geared towards game development
rather than build a lot of things from scratch and to abstract a lot of the
OS level windowing and input handling. There is a couple out there that
have varying degrees of bells and whistles and in the end I opted to use
[Piston][piston] which is being actively developed, has some basic
documentation, and has a modular architecture in case I want to use more
advance features later on.

[piston]: http://www.piston.rs/

I began with the Piston "[Getting Started][get-started]" example and made
some modifications to move much of the game logic to `lib.rs`.
Below is the modified `main.rs` file where all we do is configure the
window and start the event loop.

[get-started]: https://github.com/PistonDevelopers/Piston-Tutorials/tree/master/getting-started

``` rust
fn main() {
    // Original Defenders had a resolution of 320x256
    let mut app = App::new(GraphicsConfig::new("Defender", 960, 768));

	  // Poll for events from the window.
    let mut events = Events::new(EventSettings::new());
    while let Some(e) = events.next(&mut app.window.settings) {
        // Handle rendering
        if let Some(r) = e.render_args() {
            app.render(&r);
        }
        // Handle any updates
        if let Some(u) = e.update_args() {
            app.update(&u);
        }
    }
}
```
***Note:** There is an `App` struct that is not present in the "Getting
Started" example where I move useful game state, such as the square
position (`x`,`y`) and `rotation`.*

While in `lib.rs` we have the basic render and update loops to blank out
the screen, draw a red square, and then rotate it as seen below.

``` rust
// Handle rendering any objects on screen.
pub fn render(&mut self, args: &RenderArgs) {
    // Get the location of the "player" we want to render.
    let rotation = self.rotation;
    let x = self.x;
    let y = self.y;

    // Create a little square and render it on screen.
    let square = rectangle::square(0.0, 0.0, 50.0);
    self.window.gl.draw(args.viewport(), |c, gl| {
        // Clear the screen.
        clear(BLACK, gl);
        // Place object on screen
        let transform = c.transform.trans(x, y)
            // Handle any rotation
            .rot_rad(rotation)
            // Center object on coordinate.
            .trans(-25.0, -25.0);
        // Draw a box rotating around the middle of the screen.
        rectangle(RED, square, transform, gl);
    });
}

// Update any animation, etc.
pub fn update(&mut self, args: &UpdateArgs) {
    // Rotate 2 radians per second.
    self.rotation += 2.0 * args.dt;
}
```

Lets `cargo run` to build and run the project and see what we get! If
you're following along, you should a nice black screen with a rotating red
square in the middle.

![Rotating red square](/img/2018/window-loop-black.gif)

With the basic rendering and animation functionality set up, I moved onto
capturing input from the keyboard and translating that to movement on
screen.


## Getting Input

Piston's [event loop][events] makes it incredibly easy to poll for input
events. In the game, we're interested in keyboard presses that would lead
to our space ship moving around or firing a projectile at the enemy while
everything else we can safely ignore. I modified the event loop from the
Piston example to listen for and process keyboard events as we see below:

[events]: http://docs.piston.rs/piston/event_loop/struct.Events.html

``` rust
pub fn input(&mut self, button: &Button) {
    if let Button::Keyboard(key) = *button {
        match key {
            // For simplicity's sake we directly
            // modify the player struct here.
            Key::Up => self.player.y -= UNIT_MOVE,
            Key::Down => self.player.y += UNIT_MOVE,
            Key::Left => self.player.x -= UNIT_MOVE,
            Key::Right => self.player.x += UNIT_MOVE,
            Key::Space => (), // Fire bullets!
            _ => (), // Ignore all other
        }
    }
}
```
***Note:** In the final code, this input loop is a little more complicated since
I listen for events on button press and release, but the concept stays the same.*

## Rendering Enemies & Other Objects

As we render more and more objects on screen, I thought it best to
standardize some of the functionality we want to exist in each drawable
object. This also happened to be a great place to start playing around with
Rust [traits][traits]! Below we have a basic `GameObject` trait stating
that every object which implements that trait must have a `render` function
and optionally an `update` function.

[traits]: https://doc.rust-lang.org/book/second-edition/ch10-02-traits.html

``` rust
// Every object that needs to be rendered on screen.
pub trait GameObject {
    fn render(&self, ctxt: &Context, gl: &mut GlGraphics);
    fn update(&mut self, _: f64) {
        // By default do nothing in the update function
    }
}
```

+++

**What are traits?**

Traits in Rust allow us to abstract shared behavior that different types
may have in common. For instance, rather then make a single struct with all
attributes that may or may not be used when representing the `Player` and
`Enemy` objects, we can extract these attributes out into their own structs
and only implement the shared behavior amongst each object as a trait. For example,
from the Rust standard library things such as how to display an object as a
string are implemented as traits.

Traits also simplifies the logic that occurs in the shared behavior. For
instance, in our non-trait example perhaps we want to render enemies in a
certain way and must check a flag to see if the object is an enemy or not.
Traits eliminates this, allowing us to have separate `render` logic for the
enemy while still having compile time checks for types that are expected to
have this trait.

+++

With this `GameObject` trait, we can move the `render` and `update` logic
for the player into `models/player.rs`. This simplifies the main render and
update loop in `lib.rs` and sets the stage for future drawable objects such
as enemies, bullets, and other things.

``` rust
impl GameObject for Player {
    fn render(&self, ctxt: &Context, gl: &mut GlGraphics) {
        // Render the player as a little square
        let square = rectangle::square(0.0, 0.0, self.size);
        // Set the x/y coordinate for "spaceship"
        let transform = ctxt.transform.trans(self.x, self.y)
        // Draw the player on screen.
        rectangle(color::RED, square, transform, gl);
    }

    fn update(&mut self, dt: f64) {
        // Handle updates here. Adjusting animation/movement/etc.
    }
}
```

If you're interested in seeing how enemies and bullets are rendered and
animated, check out the `models/enemy.rs` and `models/bullet.rs`. At the
time of writing, enemies are completely harmless and only move in random
directions. Bullets are a little more complicated as they have a lifetime
value that determines when we remove them from screen.


## Handling Collisions

At this point, we should be able to render any sort of object on screen. If
the objective of the game is for the player to clear the screen of enemies
using bullets, we'll need to detect whether a bullet has in fact collided
with an enemy and handle that event accordingly. I added some additional
functions to the `GameObject` trait to handle all of this logic. In the
end, the only thing each object on screen needs to know is its current
position and the radius of its bounding circle.

Below is the snippet from the `GameObject` trait that handles collision
detection:

``` rust
fn collides(&self, other: &GameObject) -> bool {
    // Two circles intersect if the distance between their centers is
    // between the sum and the difference of their radii.
    let x2 = self.position().x - other.position().x;
    let y2 = self.position().y - other.position().y;
    let sum = x2.powf(2.0) + y2.powf(2.0);

    let r_start = self.radius() - other.radius();
    let r_end = self.radius() + other.radius();

    return r_start.powf(2.0) <= sum && sum <= r_end.powf(2.0);
}

// Use to determine position of the object
fn position(&self) -> &Position;
fn radius(&self) -> f64;
```

To run the actual collision check, we loop through each bullet and enemy
during the update loop and check to see if they've collided. If so, we
remove both from the screen and update our score.

``` rust
for bullet in self.bullets.iter_mut() {
    // Did bullet collide with any enemies
    for enemy in self.enemies.iter_mut() {
        if bullet.collides(enemy) {
            // Setting the bullet ttl will remove it from screen.
            bullet.ttl = 0.0;
            // Setting the enemy health to 0 will remove it from screen.
            enemy.health = 0;
            // Keep track of kills
            self.score += 10;
        }
    }
}
```

And there it is in action!

![Red triangle shoots green square](/img/2018/collisions.gif)

All that's missing are explosions and bits of green square scattered
across the screen.

## Keeping Score & End Game

In the previous section, you'll notice that every time a player hits an
enemy square, we update the score. Currently there is no way for the player
to know what score they have and no way for the game to end. Let's render
the score on screen and handle an end-game status.

Rendering text ended up taking a huge chunk of my time to figure out.
Mostly because some of the documentation and example projects using text
render code were out of date. In the end, I was able to dive into the
source code for the `piston2d-opengl_graphics` library to figure out the
exact incantation. In the end, it was a simple two lines of code to load
the font we want and use it to render the score.

First and foremost, we need to load the font we want to use. I chose a fantastic [old school IBM font][old-school] to really accentuate the nostalgic look.

[old-school]: https://int10h.org/oldschool-pc-fonts/readme/

``` rust
let glyph_cache = GlyphCache::new(
    "./assets/fonts/PxPlus_IBM_VGA8.ttf",
    (),
    TextureSettings::new()
).expect("Unable to load font");
```

And finally in our render loop, draw the current score on screen.

``` rust
text::Text::new_color(::color::WHITE, 16)
    .draw(
        format!("Score: {}", score).as_str(),
        glyph_cache,
        &DrawState::default(),
        c.transform.trans(0.0, 16.0),
        gl
    ).unwrap();
```
***Note**: In the final code, I created some utility functions to help render the
text. You can find these in `gfx/utils.rs`*

To detect end game states, I add a `GameStatus` enum with different states
and check for the appropriate status at the start of the render and update
loops. For example, in the snippet below we check that we've reached either
a `GameStatus::Died` or `GameStatus::Win` and render the appropriate
message to show the user.

``` rust
let viewport = [size.width as f64, size.height as f64];
match state.game_status {
    GameStatus::Died => {
        draw_center("YOU DIED!", 32, viewport, gc, &c, gl);
        return;
    },
    GameStatus::Win => {
        draw_center("YOU WIN!", 32, viewport, gc, &c, gl);
        return;
    },
    _ => (),
}
```


## Final Result

Finally, here is the final result!

![](/img/2018/final-result.gif)

Running into enemies will insta-kill the player and shooting down the harmless
wiggly green squares leads to winning the game. While a far cry from the Defender
arcade game, we have all the workings of a basic 2d game that I set out to build
as an excuse to deep dive into Rust.


## Appendix: Notes & Learnings

Since this was my first dive into a full project in Rust, I took notes on
what worked and what didn't work throughout the entire dev process.

* Rust is familiar but different (in a good way).

It's been a while since I've thought about pointers and references, heap
allocation vs. stack allocation, and even types but jumping into Rust felt
really good. It brings a lot of familiar syntax from C-based languages
(such as [structs][structs], [references, and pointers][rust-refs]) while
also introducing a lot of compelling new functionality such as
[closures][rust-closures], [traits][traits], and [lifetimes][rust-lifetimes].

[structs]: https://en.wikipedia.org/wiki/Struct_(C_programming_language)
[rust-closures]: https://rustbyexample.com/fn/closures.html
[rust-lifetimes]: https://doc.rust-lang.org/book/second-edition/ch10-03-lifetime-syntax.html
[rust-refs]: https://doc.rust-lang.org/book/second-edition/ch04-02-references-and-borrowing.html

* Cargo is an excellent tool.

[Cargo] is Rust's package manager that is a sort of mixture between
traditionally package managers and a Makefile. Cargo can handle
dependencies while also building and running your code as well. There is
also a flourishing community of [third-party subcommands][third-party] that do
everything from watching for changes and auto-building to auditing for
security vulnerabilities.

[cargo]: https://doc.rust-lang.org/cargo/
[third-party]: https://github.com/rust-lang/cargo/wiki/Third-party-cargo-subcommands

* Rust game dev is still nascent.

While [Piston][piston] made a lot of things straightforward to use,
the framework is not 100% stable quite yet. There's still plenty
of development going on that may create breaking changes from version
to version. However, it's exciting to get up and running as quickly as
I could in a scripting language such as Python.

*Edited (2018-02-08): Added missing links in appendix.*