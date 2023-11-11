+++
title = "Making blazingly ⚡ fast Minecraft cheats"
date = 2023-11-11T14:07:29+02:00
images = ["/minecraft_pigs.jpg"]
tags = ["rust", "minecraft", "linux", "evdev", "uinput"]
categories = []
+++

If you're a fan of the Minecraft bedwars minigame, you know that the Hypixel server is the *de facto* place for cheaters. The whole fun of playing there is trying to set up the best cheats you can without getting banned, and whoever does - wins the round. It's a fine line, but that's what makes it fun, it allows for fair competition between different kinds of cheaters and hackers.

For a big fan of bedwars and Rust enjoyer like myself to compete fairly with other cheaters, the only choice was to make my own cheats...

![funny image](/minecraft_soyjak.png)

## Intro

Minecraft is a simple game and, at least in bedwars, a lot of the gameplay boils down to how fast you can click your mouse. You need to spam the mouse buttons both when attacking other players and when building bridges, which are the main parts of the game. I am not even joking when I say that before I started cheating I actually broke 2 (TWO!!) different mice in the span of about a year by playing bedwars intensely. This alone is enough reason to consider switching to an autoclicker. And that's exactly what I made - an autoclicker, but no ordinary one.

### Autoclickers

An autoclicker is a simple concept - you activate it some way when you need it and it simulates fast mouse clicking. But before making it, a few questions arise:
 - How do we want to activate it?
 - What kind of clicking we want it to do? Maybe it should be possible for it to simulate keyboard events too? That would allow to completely automate bridge building!

My answer to all of these was **YES**.

I wanted my autoclicker to be activate-able by ANY sequence or combination of keyboard/mouse input, and once it is activated to be able to simulate ANY sequences or combinations of keyboard/mouse input.
I wanted it to be possible to create something as simple as remapping a key to another, and something as complex as automatically and continuously building a bridge while a key is pressed and stopping when it's released.

A simple program with a configuration file wouldn't work obviously, so I decided to use an actual programming language for the configuration of each rule.
And I chose Rust.


## Loading the autoclicker rules

After a long time pondering (not really...) I decided to have the configuration as dynamic libraries (`.so`s and `.dll`s) that will be loaded at runtime and react to user input by simulating other input.
The reason was that I love unsafe Rust, FFI and can never pass up an opportunity to debug segmentation faults.

A great crate for that is [`libloading`](https://docs.rs/libloading/latest/libloading/), and for watching the filesystem to load and remove rules at runtime [`notify`](https://docs.rs/notify/latest/notify/) is perfect.

The design I settled on was each dynamic-library/rule/macro exporting a function like this:

{{< highlight rust "linenos=table" >}}
// to check if ABI of the loader and the loadee are compatible
#[no_mangle]
static INCRO_VERSION: u64 = $crate::INCRO_VERSION;

#[no_mangle]
extern "C" fn event(incro: Incro, event: evdev::InputEvent) -> bool {
    incro.emit(evdev::InputEvent::new(..))

    true
}
{{< /highlight >}}

Incro is the name of this project (**IN**put ma**CRO**s), `evdev` is an input event interface on Linux systems that allows to monitor all device input (and simulate it via `evdev-uinput`). Since I am personally using Linux, I am not going to bother with other platforms.

As you can see, the function is called each time a new input event is registered, then the dynamic-library/rule/macro can decide how to react to it, maybe ignore, maybe update some inner state, and maybe start SPAMMING the fuck out of your mouse. And then it returns a `bool`: `true` if the original event should be "absorbed", `false` if it should be forwarded and re-emited.

Being able to "absorb" events means that we have to GRAB the input devices, instead of just monitoring them. Grabbing means that no other applications will receive their events.

Anyone who has worked with programs that grab input devices know that it's a real pain in the ass and the first thing you have to do before starting to work is slap something like this in the beginning of your program:

{{< highlight rust "linenos=table" >}}
#[cfg(debug_assertions)]
thread::spawn(|| {
    thread::sleep(Duration::from_secs(5));

    std::process::exit(0);
});
{{< /highlight >}}

This is important if you don't want to reboot your computer every time you run the program and something goes wrong, because all input devices are grabbed you literally can't do anything not even close the program, and 5 seconds is usually more than enough to test whatever you need to test.

The reason I needed the ability to "absorb" events (and in turn to grab the devices) was so it would be possible to, for example, spam a mouse button while the same button in reality is pressed and held. If you hold a button on one mouse and try clicking it on another, nothing will happen, so you must hide the fact that the first (real) mouse is being held, by grabbing the device and absorbing the events.

## Incro

The structure of the project is starting to become clear:
 - Load all dynamic-libraries/rules/macros from a directory
 - Grab all input devices
 - Create a virtual input device
 - When input is registered, run the `event` callback from each rule/macro
 - If all of them return `false`, re-emit the event from the virtual device, otherwise just absorb it.

As for the rules/macros:
 - If you want to simply remap a key, just return `true` and emit the event you want.
 - If you want to simulate some sequence of events with delays, you spawn a thread and do it there.

That introduces a question - how do you stop the new thread once it's time?
For example, for a simple autoclicker that spams left mouse button while it is being held in reality, we would want the code to look something like this:

{{< highlight rust "linenos=table" >}}
match event.kind() {
    InputEventKind::Key(Key::BTN_LEFT) => {
        if event.value() == 1 { // BTN_LEFT Pressed
            thread::spawn(move || {
                loop {
                    // Click BTN_LEFT
                    incro.emit(&[InputEvent::new(EventType::KEY, Key::BTN_LEFT.code(), 1)]);
                    incro.emit(&[InputEvent::new(EventType::KEY, Key::BTN_LEFT.code(), 0)]);

                    sleep(Duration::from_millis(100));
                }
            });
        } else if event.value() == 0 {  // BTN_LEFT Released
            // stop the thread
        }
        true
    }

    _ => false,
}
{{< /highlight >}}

![funny image](/chad_thread_cancellation.png)

These are wise words. And Rust doesn't provide a way to easily "cancel" a thread. The closest way is to periodically check an atomic bool in the thread and if its true then end execution. But we don't want every other line in our code to be a check, so I made a wrapper around `thread::spawn`.

{{< highlight rust "linenos=table" >}}
impl Incro {
    /// Spawns a thread
    pub fn thread<F: FnOnce(Incro) -> ControlFlow<()> + Send + 'static>(
        &self,
        f: F,
    ) -> ThreadHandle {
        let mut incro = self.clone();

        let atomic = Arc::new(AtomicBool::new(false));
        let handle = ThreadHandle {
            should_stop: Arc::clone(&atomic),
        };

        incro.parent_thread = Some(handle.clone());
        thread::spawn(move || f(incro));

        handle
    }
}
{{< /highlight >}}

Basically it takes a closure just like `thread::spawn` would and returns a `ThreadHandle` which is basically the atomic with a `Drop` implementation that sets it to `true`. It also gives the closure a different `Incro` instance - one that is aware of the thread it's for (it also has access to the same atomic). Now we can modify our `Incro::emit` function to check the atomic every time it's called:

{{< highlight rust "linenos=table" >}}
impl Incro {
    #[must_use]
    /// Emits fake events
    pub fn emit(&self, events: &[InputEvent]) -> ControlFlow<()> {
        if let Some(parent_thread) = &self.parent_thread {
            if parent_thread.should_stop.load(Ordering::SeqCst) {
                return ControlFlow::Break(());
            }
        }
        // Actually emit the events
        // ...

        ControlFlow::Continue(())
    }
}
{{< /highlight >}}

We also make it return `ControlFlow` which is perfect for this situation, since it works with the try `?` operator.

Now our thread code can look like this:

{{< highlight rust "linenos=table" >}}
match event.kind() {
    InputEventKind::Key(Key::BTN_LEFT) => {
        if event.value() == 1 { // BTN_LEFT Pressed
            let handle = incro.thread(move |incro| {
                loop {
                    // Click BTN_LEFT
                    // now we just add ? to the emit() invocations
                    incro.emit(&[InputEvent::new(EventType::KEY, Key::BTN_LEFT.code(), 1)])?;
                    incro.emit(&[InputEvent::new(EventType::KEY, Key::BTN_LEFT.code(), 0)])?;

                    sleep(Duration::from_millis(100));
                }
            });

            thread = Some(handle); // store the handle somewhere
        } else if event.value() == 0 {  // BTN_LEFT Released
            thread.take(); // drop the handle, stopping the thread
        }
        true
    }

    _ => false,
}
{{< /highlight >}}

Now we can easily stop the spawned thread by dropping it's handle, and we know on what points it can stop (the `emit()?`s).

If we have a situation where we press a key and then release it after a delay, it's likely that at some point the thread will need to stop during that brief moment in-between, and we dont want to leave the key pressed, so we can do this:

{{< highlight rust "linenos=table" >}}
incro.thread(move |incro| {
    // Press SPACE for example
    incro.emit(&[InputEvent::new(EventType::KEY, Key::KEY_SPACE.code(), 1)])?;

    // delay
    sleep(Duration::from_millis(100));

    // Release SPACE
    incro.force_emit(&[InputEvent::new(EventType::KEY, Key::KEY_SPACE.code(), 0)]);

    sleep(Duration::from_millis(100));
});
{{< /highlight >}}

`force_emit` is the same as `emit` but without the check we just added (original version). So now space is pressed it will be released regardless, and the only point where our thread can stop is before pressing space.

And there you have it! A BLAZINGLY FAST ⚡ autoclicker with FEARLESS CONCURRENCY 🚀

Not only can you cheat in Minecraft with it, you can cheat in any game, or just program wild macros for your keyboard.

### Sad news

As fast our autoclicker is, the rest of the programs on your computer are much slower for some reason, and you can't really simulate very precise input. I tried really hard to automate "god-bridging" in Minecraft, which requires very precisely timed mouse clicks, to a precision of a few milliseconds, and even though Incro can do that easily, the rest of the stack doesn't seem to react correctly - random delays appear. I don't know if this limitation is introduced by my desktop environment (I use KDE, tried both wayland and X11), or maybe Minecraft itself. If anyone knows how to fix this pls tell me.

## Closing remarks

Be sure to check out [`Incro`](https://github.com/PonasKovas/incro) and give it a try maybe. Hope you found this post interesting and fun!
