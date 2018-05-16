---
title: Character width bug in Bitvise SSH Client
date: 2018-05-16 04:50:49
tags:
  - Linux
  - shell
category: essay
---

## 1. Bug encountered

Some time ago, I switched to ZSH, using [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)'s default theme, robbyrussell. On my very first attempt to use it, I discovered a mysterious bug. For example:

![](01.PNG)

When auto-completing with TAB, the current line will shift back one character. It's harmless in this occasion, however, if you are auto-completing in the middle of a long command line, this will probably mess thing up, and you have to rewrite the entire command.

After that, I tested this in GNOME terminal, and it worked fine.

![](02.PNG)

## 2. Bug located

I looked through the GitHub issue page, and found out that no one else seem to have encountered a similar issue. Everyone uses auto-complete, if there's really a bug, how come it's never mentioned? Perhaps there's a bug in the terminal emulator of the SSH client I was using.

The arrow character at the beginning of the line seems larger in Bitvise xterm than that in GNOME terminal, so I highlighted them and made a comparison.

![](03.PNG)

Now we see the difference. In Bitvise xterm the arrow character is displayed two characters wide. In GNOME terminal, it's the same width as ASCII characters. The difference in display was probably due to the two terminals using different fonts. However, this is not the cause of the bug, either.

When we `echo '➜' | xxd` we can see that the UTF-8 encoded right arrow is `0xE29E9C`. Now we `vim ~/.oh-my-zsh/themes/robbyrussell.zsh-theme` and replace arrows with a Chinese character "哈"(UTF-8 `0xE59388`), open a new terminal, and try again.

![](04.PNG)

Auto-complete behavior with the Chinese character is normal, while the arrow character(having the same size and display width) isn't. Why? Now type an arrow into the terminal, and try to delete it with backspace.

![](05.PNG)

Uh-oh, the cursor only shifted back one character, and the arrow is still there. Now we found the bug. Bitvise xterm treated the arrow character as one character wide, but display it with two characters wide. 

In our first case, tab auto-completing will relocate the line using an offset depending on how many characters are before it. The arrow character is treated as one character wide, so the offset is 5 instead of 6. That's why the command line is shifted back one character.

## 3. Workaround

Workaround to this bug is simple. Just prevent using such characters in command line. To stop auto-completing from going wrong, change the theme configuration file and replace the arrow with something else. For example, I changed it into `%(!.#.$)`.

![](06.PNG)

Now the problem is solved.
