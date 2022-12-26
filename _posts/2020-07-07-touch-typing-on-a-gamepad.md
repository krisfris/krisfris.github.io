---
layout: post
title:  Touch typing on a gamepad
date:   2020-07-07
---

Typing on a keyboard requires a fixed sitting or standing position.
Gamepads, on the other hand, are handheld and compact. You can walk
around the room or lie on the couch while using them.

Due to the low number of buttons on a gamepad people haven't seriously considered
them for typing-heavy activities such as programming.

The analog sticks (of which most gamepads have two) however have the potential to provide
an infinite number of inputs. It all comes down to choosing the right gestures for
maximum efficiency and minimum strain on the thumbs.

There is a number of existing text input methods for gamepads. If you have played console games at all you have surely seen and used one or the other.

{% include image.html
url="/assets/images/nes-legendofzelda-zeldacode-5c518de44cedfd0001ddb79f.jpg"
description="On-screen text input method used by Legend of Zelda"
%}

In Legend of Zelda you had to pick letters one by one using the arrow buttons
and press the confirm button each time to add the letter to the text input field.

More efficient input methods have been developed since then. If you are interested in
exploring them, check out [this article by Gamasutra](https://www.gamasutra.com/blogs/CharlieDeck/20170721/301392/Towards_Better_Gamepad_Text_Input.php).

Unfortunately, all input methods I found have 2 major flaws that make them unsuitable for
serious work:
* They are not fast enough
* They require visual feedback

The need for speed is obvious. Depending on visual feedback is unacceptable because
it takes up precious screen space and diverts your attention from your work to the
visual input aid thereby destroying your flow and ultimately slowing you down.

No feedback however means that all gestures will need to be memorized
and practiced until they can be executed with sufficient accuracy.
While a video game can hardly ask the user to spend a couple weeks practising a
text input method, for a generic text input method that can be applied to any
program, this is an acceptable cost and in fact similar to learning touch typing.

In the rest of this post I will outline the steps I took to create a text input system for
gamepads suitable as a keyboard replacement for typing-heavy activities.

## Defining the gestures

First, I created a tool for visualizing gamepad analog stick movements using python's
`pygame` package. For the purpose of this post, I adjusted the tool to not only show
the sticks' current positions but also the past positions in increasingly
lighter shades in order for you to see the paths taken by the sticks.

The following image represents the simultaneous movement of both analog
sticks inward, up, outward, down, inward again and back to the center.

{% include image.html
url="/assets/images/dov-screenshot-1592672335.5193367.png"
description="Visualization of analog stick movement patterns"
%}

The first observation I made was that since both sticks being in the center
was the neutral state of the gamepad, all inputs had to be reachable from that neutral state
and all inputs had to be completed when both sticks returned to the center.

Given these restrictions I found that the simplest possible input was
moving one of the sticks in any direction and back to the center.

{% include image.html url="/assets/images/dov-screenshot-1592678095.1293876.png"
description="The left stick is moved up and back to the center" %}

How many and which directions could be accurately targeted blindly? Consider the
following example.

{% include image.html url="/assets/images/dov-screenshot-1592681104.3781724.png"
description="The left stick is moved up, down, to the center, left and right while the right stick is moved diagonally" %}

A few minutes of experimentation revealed that directions aligned
with the axes were easy to target accurately while inputs targeting other directions
were much less accurate (as visible in the previous image).

The next simplest inputs I found were one step and two step dialing movements.

{% include image.html
url="/assets/images/dov-screenshot-1592682528.5321155.png"
description="The left stick is moved up, to the left and back to the center"%}

{%include image.html url="/assets/images/dov-screenshot-1592682896.4498007.png"
description="The left stick is moved up, to the left, down and back to the center"%}

Considering the possible gestures so far, there were 4 + 8 + 8 = 20 inputs
on each stick.

Of course, both sticks could be moved simultaneously to form combined inputs.

{%include image.html
url="/assets/images/dov-screenshot-1592684506.4475646.png"
description="Both sticks move up and back to the center simultaneously"%}

With combined gestures there were a total of 20 * 20 + 20 + 20 = 440 inputs
which I found to be more than sufficient.

## Encoding the gestures

I divided the input space of each stick into 4 sectors and assigned a number to each sector.

{%include image.html url="/assets/images/dov-screenshot-1592983580.7373.png"
description="Input spaces divided into sectors"%}

Next, I defined a threshold around the center that would help determine whether
a stick position was in neutral state or in one of the sectors.

{%include image.html url="/assets/images/dov-screenshot-1592983964.0614512.png"
description="Circular threshold around the center"%}

As you can see, the threshold radius was quite high. I found the best radius
that would allow me to make the fewest mistakes through experimentation.

Whenever one of the sticks left the center by crosssing the threshold,
an input sequence was started. As soon as both sticks were back in the center
within the threshold, the sequence was considered completed and transformed into
a pair of tuples that described the movement of the sticks.

{%include image.html url="/assets/images/dov-screenshot-1592984344.4616897.png"
description="Stick movements for input ((0,), (2, 3))"%}

## Mapping gestures to actions

Actions in this case were simply the keys you have on a keyboard.
The shoulder buttons of the gamepad could be mapped to the Shift, Ctrl, Alt
and Super keys which would be convenient as those were the keys used for
combinations (e.g. Ctrl-C for copy).

In order to determine the optimal mapping from input gesture to key,
I recorded all keystrokes on my system using a keylogger for several weeks
and analyzed the frequency of each key.

The most frequently typed keys should be mapped to the simplest (and therefore fastest)
gestures. I measured the difficulty of a gesture by adding the lengths of the inputs for each stick together. For example, the above input ((0,), (2, 3)) would have a difficulty of 1 + 2 = 3.

Assuming that, when entering single-stick inputs,
alternating between the two sticks would be faster than
multiple single-stick inputs on the same stick,
keys that were frequently typed together should preferably be mapped to different sticks.

Following this logic, I first generated all possible single stick inputs and
grouped them by difficulty. For each difficulty I counted the inputs in that group
and took that number of keys from the sorted list of most frequent keys.

The goal was now to split these keys into two groups, one to be mapped to the inputs
on the left stick, the other to the right.
To find the ideal groups, I created a graph with the nodes being the keys and
the weighted edges representing the frequencies of key combinations.

I repeatedly removed the edge with the lowest weight, until the graph
was bipartite. If the graph became disconnected, I would recursively apply
the partitioning algorithm on the connected components and finally merge
the groups of independent sets.

Consider the following example. The first diffiulty group consists of all inputs
with difficulty 1, which are `((0,), ()), ((1,), ()), ((2,), ()), ((3,), ()),
((), (0,)), ((), (1,)), ((), (2,)), ((), (3,))`.

There are 8 inputs in this group, thus take the 8 most frequent keys from the sorted list.
These are `'e', 'o', 't', 'a', 'i', 's', 'j', 'r'`. Create a graph with these keys as nodes
and assign weights to the edges between these nodes that correspond to the frequency of
each key combination.

{% include image.html url="/assets/images/dov-graph-1.png"
description="The keys e and r are combined most frequently thus should be mapped to different sticks" %}

As I remove weak edges from the graph it eventually becomes disconnected.

{% include image.html url="/assets/images/dov-graph-2.png"
description="The key j is frequent but isolated." %}

You might wonder why the key `j` is one of the 8 most frequent keys but has such weak connections
to the other frequent keys. The reason is that `j` is used heavily when editing with VIM plus
it's part of a shortcut to switch between windows on my system. Thus it's used more in isolation
than within text.

Since the graph is disconnected I continue to apply the algorithm on the connected
components. The subgraph consisting only of the node `j` is already bipartite (`j` + empty set).
I recursively apply the algorithm to the other component.

{% include image.html url="/assets/images/dov-graph-3.png"
description="Component is bipartite after removing the weakest edges"
%}

This component can now be successfully split into 2 groups with no edges between nodes
in a group.

{% include image.html url="/assets/images/dov-graph-4.png"
description="Bipartite drawing of the component"
%}

Finally, I merge the bipartite sets.

{% include image.html url="/assets/images/dov-graph-5.png"
description="Final grouping for the first 8 keys"
%}

As you can see, the strongest links (most frequent key combinations)
are between nodes on different sides which is exactly what I want.

I repeated this process for the other difficulty groups (only single stick inputs.)
Then I generated all possible combined inputs, grouped them again by difficulty
and assigned the remaining keys to those inputs. Since combined inputs require
the use of both sticks the problem of splitting up keys into two groups is irrelevant there.

I used python's `pyautogui` package to generate keyboard events when actions
were triggered.

## Practice

I used the same touch typing trainer `ktouch` that I had already used for
learning to type on a keyboard nearly two decades ago. I created custom lessons for this purpose.

{% include image.html
url="/assets/images/dov-ktouch.png"
description="Practising gamepad touch typing in ktouch"
%}

## Observations

* While the python process running this input system usually stayed below 10% CPU usage,
if it was to run in the background continuously I would have preferred to reimplement and optimize
it in a lower level language to leave the CPU available for more intensive foreground tasks.

* After I upgraded my gamepad to the DualShock4, I realized I could make diagonal inputs
relatively accurately. Integrating diagonal inputs would reduce the number of more
complex inputs required thus increasing speed.

* After weeks of practice I could type about half as fast on the gamepad as on the keyboard.
Considering my insane typing speed on the keyboard that was not bad at all.
For additional gains, I would use the remaining available inputs and map them to custom
actions such as shortcuts or program-dependent behavior.

* Dialing movements seem to strain the thumbs the most. Replacing them with longer but
more straightforward inputs might be worth a shot.

* The strain on the thumbs recedes with increasing practice. However, in the beginning
practice should be limited to a few minutes at a time.

## Conclusion

In just a couple days I put together an efficient typing system for gamepads.
There are numerous possible improvements but this setup serves as a proof of concept
which demonstrates that efficient text input on a gamepad is achievable.
The code for this project can be found on [GitHub](https://github.com/krisfris/pydov).

## Discuss

Join the [Matrix chat room](https://matrix.to/#/#dov:matrix.org) for this project or comment below.

