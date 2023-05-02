Download Link: https://assignmentchef.com/product/solved-comp2300-assignment-3-part-2
<br>
In the final assignment, you’ll take the principles from the sequencer instrument you built in <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/02-sequencer/">assignment 2</a> and make it network-aware, so that it can receive messages from the outside world. That means that instead of playing a sound with a function call in your program you do it by sending a “message” as a change in voltage (an electrical signal) on the GPIO pins. This is a <strong>live wire</strong>—it’s your job to make sure these electrical signals are sent &amp; received to successfully play the sound. To illustrate<sup id="fnref:acdc" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#fn:acdc">1</a></sup>, here’s AC/DC live in Paris (1979) playing <em>Live Wire</em>:

As you’ve seen in several of the labs (e.g. the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/">blinky lab</a> and <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/09-input-through-interrupts/">lab 9</a>) the discoboard’s GPIO pins provide a lot of flexibility for transmitting data (<code>0</code>s and <code>1</code>s) as low and high voltages. In those labs, the pins were connected to LEDs &amp; joysticks, but you might have noticed there are also a bunch of little gold-coloured pins sticking up on your board connected to nothing in particular. Any of these which are marked <strong>P</strong>XY (where X is the <em>port number</em> from A to F and Y is the <em>pin number</em> from 0 to 15) are GPIO pins as well, and you can connect them up using jumper wires like so:

<img decoding="async" alt="Discoboard with networking wires" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/deliverables/03-networked-instrument/discoboard-wires.jpg?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/deliverables/03-networked-instrument/discoboard-wires.jpg?w=980&amp;ssl=1" alt="Discoboard with networking wires" data-recalc-dims="1">

 </noscript>

<p class="info-box">You have to be a little bit careful when configuring your GPIO pins as inputs and outputs like this—it is possible to fry your board with a <a class="acton-tabs-link-processed" href="https://youtu.be/1-yzqgwTVi8">short circuit</a>.

This assignment has two parts: in Part 1, you need to implement the <strong>P2300 communication protocol</strong>—a specific way of sending the <code>1</code>s and <code>0</code>s over the wire and interpreting them at the other end. In Part 2, you get to design your own protocol for playing sounds over the wire.

The <a class="acton-tabs-link-processed" href="https://gitlab.cecs.anu.edu.au/comp2300/2021/comp2300-2021-assignment-3">assignment 3 template repo</a> contains the usual starter code, including the utility libraries which have been introduced in the labs over the last few weeks. These libraries are there so that you can skip over the stuff you’ve already done in the previous assignments and spend your energy on actually implementing the P2300 protocol. It’s still crucial that you understand what this code does, though, so if you’re not sure about anything then ask a question on <a class="acton-tabs-link-processed" href="https://piazza.com/class/kl5njlf936c40y">the COMP2300 forum</a> using the <code>assignment3</code> tag.

<h2 id="p2300-protocol">Background: the P2300 protocol</h2>

<p class="info-box">This assignment requires a bit more explanation than the first two. Make sure you <strong>read this section carefully</strong> so that you understand the P2300 protocol.

The purpose of P2300 is to allow two devices to work together to play music (to generate a sound waveform). The overall goal is similar to the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/02-sequencer/">sequencer you wrote in assignment 2</a>, except that this time the “messages” to trigger a new note are sent over a wire.

To communicate over a wire (a network) it’s not enough to just connect two GPIO pins together with a jumper wire. You need a <em>protocol</em>: a way of interpreting the signals on the wire so that the sender and the receiver can understand one another. In Part 1 of this assignment you need to implement the <em>P2300 protocol</em>.

<h3 id="wires">Wires</h3>

The P2300 protocol (which from here we’ll refer to as just P2300) describes the way that a <strong>sender</strong> device can control the playback of a <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/02-sequencer/">sequence</a> of notes on a <strong>receiver</strong> device. P2300 is a simplex 2-wire protocol, which means:

<ul>

 <li>the information flows in one direction: <em>from</em> the device acting as the sender <em>to</em> the device acting as the receiver</li>

 <li>it requires two physical connections between the sender &amp; the receiver</li>

</ul>

Here’s a high-level diagram of the information flow:

<img decoding="async" alt="P2300 protocol connections" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/deliverables/03-networked-instrument/p2300-high-level.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/deliverables/03-networked-instrument/p2300-high-level.png?w=980&amp;ssl=1" alt="P2300 protocol connections" data-recalc-dims="1">

 </noscript>

The two lines<sup id="fnref:line" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#fn:line">2</a></sup> (wires) used in P2300 are:

<ul>

 <li>a <strong>note on/off</strong> line which the sender uses to tell the receiver to begin playing the sound at the current pitch (frequency); i.e. to turn the sequencer <em>on</em> or <em>off</em></li>

 <li>a <strong>pitch change</strong> line which the sender uses to tell the receiver to change the current pitch of the note being played</li>

</ul>

<p class="info-box">Don’t get confused between the terms <em>pitch</em> and <em>frequency</em>, in the context of the P2300 protocol they mean the same thing.

In principle the P2300 connections can be made on any pair of GPIO pins on your discoboard. However, so that we can test your program easily you <strong>must</strong> use these specific pins:

<ul>

 <li>connect the <strong>note on/off</strong> line from <strong>PE12</strong> (sender) to <strong>PH0</strong> (receiver)</li>

 <li>connect the <strong>pitch change</strong> line from <strong>PE13</strong> (sender) to <strong>PH1</strong> (receiver)</li>

</ul>

<h3 id="sender">Sender</h3>

Here is the full list of P2300 “messages” the sender may send to the receiver:

<ol>

 <li>a <em>rising edge</em> on the <strong>on/off line</strong> tells the receiver to begin making a sound (the pitch of which is determined by the current <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-pitch-sequence">pitch sequence index</a>)</li>

 <li>a <em>falling edge</em> on the <strong>on/off line</strong> tells the receiver to immediately stop making any sound at all (i.e. to go quiet)</li>

 <li>a <em>rising edge</em> on the <strong>pitch change</strong> line tells the receiver to increment the current <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-pitch-sequence">pitch sequence index</a> by 1 (regardless of whether the receiver is currently playing sound or not)</li>

 <li>a <em>falling edge</em> on the <strong>pitch change</strong> line has no effect</li>

</ol>

The sender device doesn’t generate the waveform itself (that’s kinda the point). It expects that there’s a <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#receiver">receiver</a> listening on the other end which will correctly interpret the “messages” (the voltage changes on the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#wires">on/off and pitch change lines</a>).

<h3 id="receiver">Receiver</h3>

The P2300 receiver is the device which actually makes the sound. The waveform must be a sawtooth wave (unlike in assignments <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/01-synth/">1</a> and <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/02-sequencer/">2</a> which were square waves).

As well as responding to the P2300 messages as described above, on startup the P2300 receiver device must:

<ol>

 <li>make no sound (as if the last <strong>note on/off</strong> message was an “off” message)</li>

 <li>initialise the starting <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-pitch-sequence">pitch sequence index</a> to 0 (i.e. the first element in the P2300 pitch sequence)</li>

</ol>

Finally, between messages<sup id="fnref:instantaneous" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#fn:instantaneous">3</a></sup> the receiver must either:

<ol>

 <li>continue to play a sawtooth wave at the current pitch (if the last <strong>note on/off</strong> message was an “on” message)</li>

 <li>remain silent (if the last <strong>note on/off</strong> message was an “off” message)</li>

</ol>

<h3 id="p2300-pitch-sequence">P2300 pitch sequence</h3>

In P2300 a pitch change message does not specify an actual pitch (either in Hz or as a MIDI note number). Instead, P2300 specifies that the receiver will play a pitch from a pre-arranged sequence of pitches. Each pitch change message increments the pitch sequence index by 1 (wrapping back to 0 when it is asked to increment from 7), so that the receiver will play the next pitch in the sequence.

This means that P2300 is a stateful protocol; you can’t look at any particular message and know what the receiver will do, you also need to know the current pitch sequence index.

Here is the P2300 pitch sequence (remember that on startup the pitch sequence index must be initialised to 0, i.e. the starting pitch is 220Hz):

<table id="note-table">

 <thead>

  <tr>

   <th>pitch sequence index</th>

   <th>pitch (Hz)</th>

  </tr>

 </thead>

 <tbody>

  <tr>

   <td>0</td>

   <td>220.00</td>

  </tr>

  <tr>

   <td>1</td>

   <td>246.94</td>

  </tr>

  <tr>

   <td>2</td>

   <td>261.63</td>

  </tr>

  <tr>

   <td>3</td>

   <td>293.66</td>

  </tr>

  <tr>

   <td>4</td>

   <td>329.63</td>

  </tr>

  <tr>

   <td>5</td>

   <td>369.99</td>

  </tr>

  <tr>

   <td>6</td>

   <td>392.00</td>

  </tr>

  <tr>

   <td>7</td>

   <td>440.00</td>

  </tr>

 </tbody>

</table>

<p class="attn-box">In Part 1 you cannot change the pitch sequence—e.g. by adding new entries to the end of the table or shuffling the order. If you do this, you’re changing the P2300 protocol, and <strong>you will lose marks accordingly</strong>.

If the current pitch sequence index is the last pitch in the pitch sequence (i.e. if the pitch sequence index is 7) and a <em>rising edge</em> is triggered on the <strong>pitch change</strong> line, then the receiver should set the current pitch back to the first entry in the table. In other words, the pitch sequence should loop back to the beginning.

In this way, the P2300 protocol can be used to play any sequence of pitches from the set in the above table<sup id="fnref:dorian" role="doc-noteref"><a class="footnote acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#fn:dorian">4</a></sup>, e.g. to “skip” a note in the sequence the sender should send a note off messages, then send two pitch change messages, then send the next note on message.

<h3 id="p2300-example">P2300 example</h3>

Here’s an example of the P2300 protocol in action.

<img decoding="async" alt="P2300 example timeline" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/deliverables/03-networked-instrument/p2300-example-timeline.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/deliverables/03-networked-instrument/p2300-example-timeline.png?w=980&amp;ssl=1" alt="P2300 example timeline" data-recalc-dims="1">

 </noscript>

The diagram shows an example “communication timeline “ for the P2300 protocol. It shows the way the voltage (either <code>0</code> or <code>1</code>) on both the <strong>note on/off</strong> wire (blue) and the <strong>pitch change</strong> wire (green) changes over time. It also shows the resulting “effects” of the P2300 communication; both the sound output (in orange—not to scale!) and the current pitch sequence index (in the pink boxes).

There are a few other things worth noting here:

<ul>

 <li>the required <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#receiver">“starting state”</a> is satisfied</li>

 <li>pitch changes only happen on a <em>rising edge</em>; it doesn’t matter when the subsequent <em>falling edge</em> happens</li>

 <li>between messages (i.e. when there are no voltage changes on either line) the receiver keeps on playing (or being silent)</li>

</ul>

<h2 id="part-1">Part 1</h2>




In this assignment, you need to program your discoboard to implement both the <strong>sender</strong> and the <strong>receiver</strong> parts of P2300. This means that the jumper wires will connect from your board to itself (as <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#wires">described above</a>). If you’re confused about how this works, <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#sender-and-receiver">see the FAQ</a>.

In Part 1 you need to write an assembly program which uses <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-protocol">P2300</a> to play the following song (the pitch sequence indexes that are shown come from the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-pitch-sequence">table above</a>) and indicate what the current value of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-pitch-sequence">pitch sequence index</a> should be. As in assignment 2 a gap (silence) between notes is represented by a <code>-</code> in the pitch and pitch sequence index columns. The song must loop: when it gets to the end, it must loop back to the beginning. You must stick to the P2300 protocol and pitch sequence <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-pitch-sequence">exactly as described above</a>—you cannot modify the receiver’s pitch sequence/table to play the sequence.

<table id="note-table">

 <thead>

  <tr>

   <th>pitch sequence index</th>

   <th>pitch (Hz)</th>

   <th>duration (s)</th>

  </tr>

 </thead>

 <tbody>

  <tr>

   <td>0</td>

   <td>220.00</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>2</td>

   <td>261.63</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>1</td>

   <td>246.94</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>3</td>

   <td>293.66</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>2</td>

   <td>261.63</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>4</td>

   <td>329.63</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>3</td>

   <td>293.66</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>5</td>

   <td>369.99</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>4</td>

   <td>329.63</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>6</td>

   <td>392.00</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>5</td>

   <td>369.99</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>7</td>

   <td>440.00</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>6</td>

   <td>392.00</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>5</td>

   <td>369.99</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>4</td>

   <td>329.63</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>3</td>

   <td>293.66</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>2</td>

   <td>261.63</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>1</td>

   <td>246.94</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>0</td>

   <td>220.00</td>

   <td>0.25</td>

  </tr>

  <tr>

   <td>–</td>

   <td>–</td>

   <td>0.25</td>

  </tr>

 </tbody>

</table>

If all goes well, it should sound like this:



 <audio controls="controls" data-mce-fragment="1"></audio>

Marks will be awarded for:

<ul>

 <li>playing the notes with the correct pitches (frequency)</li>

 <li>playing the notes at the correct tempo (timing)</li>

 <li>clean note transitions (i.e. no pitches other than those in the sequence above, even for a split second)</li>

 <li>code structure, readability &amp; modularity (including comments &amp; use of functions)</li>

 <li><strong>to be eligible for full marks in part-1</strong>, you must use a hardware timer to control the timing of the sender.</li>

</ul>

<p class="attn-box">For Part 1, you <strong>must</strong> play the sequence by sending messages over the wire using the P2300 protocol (<a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#is-it-working">we will test this</a>). If you try and play the sequence some other way (e.g. by playing it “directly” similar to your sequencer from assignment 2) you’ll get zero!

<p class="info-box">To help you out, there’s a bit more starter code in the <a class="acton-tabs-link-processed" href="https://gitlab.cecs.anu.edu.au/comp2300/2021/comp2300-2021-assignment-3">assignment 3 template</a> compared to assignments 1 and 2. In particular, have a look in <code>src/libcomp2300/tim7.S</code> for some helper functions for using the tim7 timer (for the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#sender">sender</a> part), <code>src/libcomp2300/wave.S</code> for some helper functions for playing the sawtooth wave (i.e. the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#receiver">receiver</a> part).In addition to this there are <em>plenty</em> of useful macros and functions in the <code>src/libcomp2300/macros.S</code> and <code>src/libcomp2300/utils.S</code> files respectively (e.g. setting up gpio pins, interrupts, priorities etc.).Please do take the time to check them out.

<p class="think-box">There is a lot going on in this assignment, so it’s understandable that this may seem a little overwhelming. To accommodate for this, the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#faq">FAQ</a> for this assignment is quite extensive, please do take the time to read it all.However, for your convenience here is a few of the points we’d like to highlight:—understand that interrupts <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#debugging-and-interrupts">may not work as expected when debugging</a>—following from the above, you <strong>NEED</strong> to be resetting your board (click in the black stick to the left of the joystick) before stepping through a debugging session—think about <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#how-to-decrement">how to traverse the table (up and down)</a> without <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#can-i-change-the-pitch-sequence-for-part-1">changing the pitch table in memory</a>—ensure that there is <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#separation">no implicit (or explicit) communication between sender and receiver</a> outside of the protocol, (no shared register or memory usage).—utilize the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#interrupts-lmao">common interrupt pitfalls</a> to check off possible issues with your code

<h2 id="part-2">Part 2</h2>

In part 2, you get to implement a more advanced network protocol for controlling the music over the wire. There are a few ways to do this:

<ul>

 <li>extend the P2300 protocol in a meaningful way</li>

 <li>implement a serial protocol using a <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#serial-shell">simple serial protocol outlined below</a></li>

 <li>design an all-new protocol</li>

 <li>implement an existing protocol (e.g., an industry standard such as <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/modi%20wikipedia%20hindi">MIDI</a>, or <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Open_Sound_Control">OSC</a>)</li>

 <li><em>some combination of the above</em> (e.g. modify P2300 using ideas from an existing industry-standard protocol)</li>

</ul>

You also no longer have to play the specific note sequence from <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#part-1">Part 1</a>, you can play whatever sequence you like—perhaps pick one which shows off the cool features of your new protocol.

If you decide to extend the P2300 protocol, you’re free to change any aspect of it. However, there is one caveat—if you make a really superficial change (e.g. changing only the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-pitch-sequence">pitch sequence</a>, or only changing rising edge triggers to falling edge triggers or vice versa) then that <strong>will not</strong> be enough to pass (it’d be difficult to write a good design document with such a superficial change anyway). If you’re unsure, check with your tutor or ask on <a class="acton-tabs-link-processed" href="https://piazza.com/class/kl5njlf936c40y">the COMP2300 forum</a>.

Here are some ideas for Part 2:

<ul>

 <li>extend P2300 to allow the <strong>sender</strong> to:

  <ul>

   <li>play multiple notes at once (i.e. <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Harmony">harmony</a>)</li>

   <li>change the <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Timbre">timbre</a> of the receiver’s waveform (e.g. square, triangle, sine)</li>

   <li>adjust the amplitude (or even <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Synthesizer#Attack_Decay_Sustain_Release_.28ADSR.29_envelope">amplitude envelope</a>) of the notes played by the receiver</li>

  </ul></li>

 <li>design a protocol where the <strong>sender</strong> can specify any pitch (frequency) for playback (instead of just choosing from a pre-determined <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#p2300-pitch-sequence">pitch sequence</a>), either by:

  <ul>

   <li>using a clock line or timing-based protocol to send a multi-bit “packet” over the wire one-bit-at-a-time (a <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Serial_communication">serial protocol</a>)</li>

   <li>using multiple wires to send all the bits of the packet simultaneously (a <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Parallel_communication">parallel protocol</a>)</li>

  </ul></li>

 <li>implement a real-world protocol like <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/1-Wire#Communication_protocol">1-wire</a>, <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/I2c">I2C</a>, <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus">SPI</a> or another pre-existing protocol (some of these are pretty hard—but they’re a great challenge if you want to stretch yourself)</li>

</ul>

<p class="attn-box">In <code>part-2</code> you will have a new file called <code>pins.yml</code>. This file contains the information about which pin connections you are using (what pins are wired together). This must be filled out correctly and up to date at the time of submission or you <strong>will</strong> lose marks.As mentioned above, you don’t <em>have</em> to use a different wiring configuration. See the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/deliverables/03-networked-instrument/#part-2-wires">FAQ for more details</a>.

Marks for Part 2 will be awarded for a <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/resources/design-document/"><strong>design document</strong></a> describing what you’re doing and how you implemented it in ARM assembly language. You need to explain the <strong>“what”, “how” and “why”</strong> (<a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/resources/design-document/">design, implementation, and analysis</a>) of what you have done. Although it’s ok if you don’t do something <em>super</em>-complex, we do take the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/resources/faq/#ambition">sophistication</a> of your protocol into account. <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/resources/faq/#images-in-dd">Using images/diagrams</a> is encouraged. Your design document must be in <strong>pdf</strong> format <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/resources/faq/#can-my-design-document-be-longer-than-two-pages">(2 pages content + appendix + references)</a> with the filename <code>design-document.pdf</code> in top-level folder on the <code>part-2</code> branch.

<p class="attn-box">The design document for Part 2 is based on the protocol <em>you</em> implement (what it is, why is it like that, how is it implemented). That means don’t write about the sequence of notes you choose to play, or how frequency etc. is calculated.For more help on writing the design document, <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/resources/design-document/">check out this page</a>.

<span class="kksr-muted">Rate this product</span>

[youtube https://www.youtube.com/watch?v=6tdiMdj164w]

The audio setup here is the same as assignment 2, you still need to run <code>bl init</code> once at the start of your program <strong>before</strong> you do anything else. If you fail to do this then <strong>you will lose marks.</strong>

In general the start of your program should look like this: