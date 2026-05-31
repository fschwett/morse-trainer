# Morse trainer

## Idea:

I wanted a simple website for practicing Morse code with the Koch method on the go. Other websites lacked some of the customization options I wanted, so I built my own.

## Concept:

The Koch method is a widely used approach for learning Morse code. During practice, five characters are played in Morse code and you try to immediately write them down. This makes the process more similar to language acquisition than to consciously decoding symbols, allowing you to become fluent much more quickly. Additionally, not all characters are practiced from the beginning. Instead, you start with only two or three characters and gradually build your way up.

I wanted to have a website that gives me full control over which characters, speeds, and other settings I practice with. I ended up implementing three different modes for different types of training, each with a number of settings to optimize the learning experience.

* **Continous** just keeps playing five character blocks of Morse code at the speed and pitch you chose.
* **Block** is the same, except after each block it waits for a specified amount of time before revealing the correct characters.
* **Test** plays a block of five characters. The user has to type the decoded characters into an input field. They are then checked so theres no need for paper.

I planned to implement another mode that would've worked similarly to the test mode, except it would've played a specified number of blocks and tracked the error rate. But since I ended up only using the block and test modes so I never implement the last mode.

Morse code timing is a precisely defined ratio build upon the duration of the short signal (dit). The long signal (dah) is three times the duration of a dit. The pause between two signals within a character is one dit. the pause after a character is three dit. Lastly the pause between two words is seven dit. That is the official timing and the timing I used in this project.

## Implementation:

The website is built with HTML/CSS and JS. It uses the bootstrap framework for styling. The application is built on the idea of portability and low maintenance. That's why I used simple HTML/CSS instead of processed solutions like SCSS, EJS or anything similar. Also I use embedded CSS and inline JS instead of external files for the same reason, even though it's not as clean as external files. This way one file works as a standalone application and can be used on any device and provided by any server, even the [github html preview](https://github.com/fschwett/morse-trainer#preview).

The application is build around the `setTimeout()` function, that is called on each signal flank. It's callback (`handleTimeout`) essentially creates a self sustaining loop and that handles the Morse signal and creation of new blocks as well as the logic after each block (depending on the mode).

### Initialization:

The first function to be called is `init`. It creates a checkbox for each character so the user can choose which characters to practice.

```js
let fullAlphabet = "abcdefghijklmnopqrstuvwxyz1234567890";

for (let i = 0; i < fullAlphabet.length; i++) {
    ...
}
```

### Variables:

After that, the function adds event handlers to all the GUI elements like the pitch input, the dit duration input etc and initializes the AudioContext for the app.

The most important variables are `run` which stores the current state of the application, `timeoutID` which stores the ID of the forementioned timeout, `morse` which maps each character with it's corresponding Morse code, along with the variables that store user settings, like `showCode`, `playMorse` etc.

### Run status:

The run status is handled by the `runButtonClick()` function. It checks the current status and if the app is running, it clears the current timeout, calls `stopSound()`, sets `run` to false and resets all the elements. If it's not running, it starts the application. As mentioned before `handleTimeout()` creates a loop that also generates a new block and thus `runButtonClick()` just sets all the parameters as if a block finished and then call `handleTimeout()` which starts the app and reduces code.

### Timeout handling:

`timeoutHandler()` checks some settings and manipulates the document accordingly first. Then it determines the duration of the next timeout and decides whether to start or stop the sound. Therefore it uses the 



### Sound handling:

The Morse signal is created using the Web Audio API's `AudioContext`. I implemented the methods `startSound()` and `stopSound()`. They use an oscillator and a gain node to produce a sound. The oscillator produces a sine wave with a specific gain. If the oscillator is stopped abruptly, a clicking sound can occur because the audio signal is cut off at an arbitrary point in its waveform. This instantaneous discontinuity introduces a wide range of high-frequency components, which are perceived as a click. This is not a computer error or a limitation of the Web Audio API; it is a natural consequence of how the human auditory system processes sudden changes in sound.

There are essentially two ways to get rid of the click:

* gradually decrease gain
* stop exactly at a zero point crossing

The following graphic shows what the abrupt stop, fading out and the stop at a zero point crossing look like:

<img width="415" height="196" alt="waves-transparent" src="https://github.com/user-attachments/assets/99df00af-96bb-450a-baf2-92c9bcdc2cb2" />

The easiest way to implement a click free sine wave was by in and decreasing the gain. This is done by using the gain nodes `gain.setTargetAtTime()` method. It sets the gain to a certain value starting at a specified time and transitioning over a specified amount of time. To start the sound, the oscillator is started with a gain of 0. The gain then transitions to 1 over the course of 0.002 seconds which I found was the optimal time to avoid clicking and still give a clear start.

```js
gainNode.gain.setTargetAtTime(0, audioContext.currentTime, 0);

if (playMorse) oscillator.start();

gainNode.gain.setTargetAtTime(1, audioContext.currentTime, 0.002);
```

Same goes for stopping the oscillator. The gain is reduced to 0 over the course of 0.003 seconds and additionally a timeout is set which stops the oscillator after 0,01 seconds

```js
gainNode.gain.setTargetAtTime(0, audioContext.currentTime, 0.003);
setTimeout(() => { oscillator.close(); }, 10);
```

## Preview:

The project can be viewed at [html-preview.github.io](https://html-preview.github.io/?url=https://github.com/fschwett/morse-trainer/blob/main/index.html).
