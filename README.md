# Morse trainer

## Idea:

I wanted a simple website for practicing Morse code with the Koch method on the go. Other websites lacked some of the customization options I wanted, so I built my own.

## Concept:

The Koch method is a widely used approach for learning Morse code. During practice, five characters are played in Morse code, and you try to immediately write them down. This makes the process more similar to language acquisition than consciously decoding symbols, allowing you to become fluent much more quickly. Additionally, not all characters are practiced from the beginning. Instead, you start with only two or three characters and gradually build your way up.

I wanted a website that gives me full control over which characters, speeds, and other settings I practice with. I ended up implementing three different modes for different types of training, each with a number of settings to optimize the learning experience.

* **Continuous** continuously plays five-character blocks of Morse code at the selected speed and pitch.
* **Block** works the same way, except that after each block it waits for a specified amount of time before revealing the correct characters.
* **Test** plays a block of five characters. The user has to type the decoded characters into an input field. They are then checked, so there is no need for paper.

I originally planned to implement another mode that would have worked similarly to the test mode, except that it would have played a specified number of blocks and tracked the error rate. However, since I ended up using only the block and test modes, I never implemented the additional mode.

Morse code timing is based on precisely defined ratios built upon the duration of the short signal (dit). The long signal (dah) is three times the duration of a dit. The pause between two signals within a character is one dit. The pause after a character is three dits. Lastly, the pause between two words is seven dits. That is the official timing and the timing I used in this project.

## Implementation:

The website is built with HTML/CSS and JavaScript. It uses the Bootstrap framework for styling. The application is designed around portability and low maintenance. That's why I used simple HTML/CSS instead of processed solutions such as SCSS, EJS, or similar technologies. I also use embedded CSS and inline JavaScript instead of external files for the same reason, even though it is not as clean as using separate files.

This way, a single file works as a standalone application and can be used on any device and served by any web server, even through the [GitHub HTML preview](https://github.com/fschwett/morse-trainer#preview).

The application is built around the `setTimeout()` function, which is called on each signal transition. Its callback (`handleTimeout`) essentially creates a self-sustaining loop that handles the Morse signal, generates new blocks, and executes the logic after each block depending on the selected mode.

### Initialization:

The first function to be called is `init`. It creates a checkbox for each character so the user can choose which characters to practice.

```js
let fullAlphabet = "abcdefghijklmnopqrstuvwxyz1234567890";

for (let i = 0; i < fullAlphabet.length; i++) {
    ...
}
```

### Important variables:

After that, the function adds event handlers to all GUI elements, such as the pitch input and the dit-duration input, and initializes the `AudioContext` for the app.

The most important variables are `run`, which stores the current state of the application, `timeoutID`, which stores the ID of the aforementioned timeout, and `morse`, which maps each character to its corresponding Morse code. There are also variables that store user settings, such as `showCode`, `playMorse`, and others.

### Run status:

The run status is handled by the `runButtonClick()` function. It checks the current status and, if the app is running, clears the current timeout, calls `stopSound()`, sets `run` to `false`, and resets all UI elements. If the app is not running, it starts the application. As mentioned before, `handleTimeout()` creates a loop that also generates new blocks itself. Therefore, `runButtonClick()` simply sets all parameters as if a block had just finished and then calls `handleTimeout()`, which starts the app while reducing code duplication.

### Logic and timeouts:

`timeoutHandler()` handles most of the logic. It checks various settings and updates the document accordingly. It then determines the duration of the next timeout and decides whether to start or stop the sound. Other timeout-related functions are `timerHandler()` and `countIn()`, which handle the countdowns before revealing a block and before playing the next block.

To always play five characters per block, the app uses a zero-based counter called `charNumber`. The handlers always schedule another timeout after executing their code. The callback used for the next timeout depends on the value of `charNumber`.

**Continuous mode** always calls `timeoutHandler()` and never changes the callback because there are no separate phases.

In **block mode**, when the counter reaches 5, the block has finished, and `timerHandler()` becomes the next callback. It counts down, reveals the block, and increments `charNumber` again. This tells the function that the next timer should start, and it begins another countdown. When it finishes counting down, it schedules another timeout using `timeoutHandler()` as the callback.

**Test mode** breaks the loop after reaching the end of a block. When the user enters an answer, it is checked and `countIn()` is called. It works similarly to `timerHandler()`. After counting in, it switches back to `timeoutHandler()` for the next callback.

### Sound handling:

The Morse signal is generated using the Web Audio API's `AudioContext`. I implemented the methods `startSound()` and `stopSound()`. They use an oscillator and a gain node to produce sound. The oscillator generates a sine wave with a specific gain level.

If the oscillator is stopped abruptly, a clicking sound can occur because the audio signal is cut off at an arbitrary point in its waveform. This instantaneous discontinuity introduces a wide range of high-frequency components, which are perceived as a click. This is not a computer error or a limitation of the Web Audio API; it is a natural consequence of how the human auditory system processes sudden changes in sound.

There are essentially two ways to eliminate the click:

* Gradually decrease the gain.
* Stop exactly at a zero-crossing point.

The following graphic shows what an abrupt stop, a fade-out, and a stop at a zero-crossing point look like:

<img width="415" height="196" alt="waves-transparent" src="https://github.com/user-attachments/assets/99df00af-96bb-450a-baf2-92c9bcdc2cb2" />

The easiest way to implement a click-free sine wave was by gradually increasing and decreasing the gain. This is done using the gain node's `gain.setTargetAtTime()` method. It sets the gain to a specified value at a specified time and transitions toward that value over a specified time constant.

To start the sound, the oscillator is started with a gain of 0. The gain then transitions to 1 over the course of 0.002 seconds, which I found to be the optimal value for avoiding clicks while still providing a clear start.

```js
gainNode.gain.setTargetAtTime(0, audioContext.currentTime, 0);

if (playMorse) oscillator.start();

gainNode.gain.setTargetAtTime(1, audioContext.currentTime, 0.002);
```

The same principle is used when stopping the oscillator. The gain is reduced to 0 over the course of 0.003 seconds, and additionally a timeout is set that stops the oscillator after 0.01 seconds.

Since `stopSound()` may be called while the oscillator is not running, `oscillator.stop()` could throw an exception. This can happen, for example, if the user clicks the stop button during the pause between two Morse signals. The oscillator will not be running at that moment, but `stopSound()` will still be called. Therefore, I use a try-catch block.

```js
gainNode.gain.setTargetAtTime(0, audioContext.currentTime, 0.003);

setTimeout(() => {
    try {
        oscillator.stop();
    } catch (e) {
    }
}, 10);
```

## Text-to-speech feature

The block mode includes a feature that lets you hear the correct solution read aloud. This allows the mode to be used auditorily without looking at the screen.

The Web Speech API's `SpeechSynthesis` interface could have been used, but it did not work the way I wanted. At higher playback speeds, characters started to overlap. Therefore, I chose to create an MP3 file for each character. I create five `Audio` objects for each MP3 file because using only one object per character caused issues when the same character needed to be played twice in quick succession.

To maintain my "one-file-to-rule-them-all" approach, I wanted a way to embed the MP3 files directly into the website. The first version simply used a folder containing MP3 files that were loaded into a `Map` object when the feature was enabled. The current version also uses a `Map`, but instead of mapping each character to an audio file, it maps them to Base64-encoded audio data that is hardcoded into the HTML file.

They are stored in a separate `<script>` tag for better organization.

The reason I did not simply hardcode five maps is that most browsers only allow audio playback after a user interaction. I wrote a function that maps each character to its Base64-encoded audio file and then pushes the result into the `audios` array:

```html
<script>
    let audios = [];

    let createAudios = () => {
        let map = new Map([
            ["a", new Audio("data:audio/mpeg;base64,//NkxAAas ...")],
            ["b", new Audio("data:audio/mpeg;base64,//NkxAAZk ...")],
            ["c", new Audio("data:audio/mpeg;base64,//NkxAAa0 ...")]
        ]);
        
        audios.push(map);
    };
</script>
```

This function is then called five times by `playSolutionSwitched()` after user interaction. I added a guard clause because I only wanted five copies of the files:

```js
let playSolutionSwitched = () => {
    if (audios.length > 0) return;

    for (let i = 0; i < 5; i++) {
        createAudios();
    }
}
```

## Preview:

The project can be viewed at [html-preview.github.io](https://html-preview.github.io/?url=https://github.com/fschwett/morse-trainer/blob/main/index.html).