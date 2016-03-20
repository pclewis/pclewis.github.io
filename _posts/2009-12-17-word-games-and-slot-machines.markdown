---
author: pcl
comments: true
date: 2009-12-17 06:43:56+00:00
layout: post
permalink: /2009/12/word-games-and-slot-machines/
slug: word-games-and-slot-machines
title: Word Games and Slot Machines
wordpressid: 37
categories:
- second life
tags:
- autoit
- c/c++
- lsl
- second life
---

I have this thing with games.

For many simple games, especially word games, there is a pretty straightforward strategy to follow to play a "perfect" game. Scrabble is a particularly good example. The simplest strategy is to play the best word you can, which is easily quantifiable by points. Refinements are obvious: try to save high scoring letters for bonus squares, try to make the board worse for your opponent.

Once you've figured the basics out, the most effective way to improve your game is to expand your vocabulary. At first, this seems like a pretty "human" endeavor. However, anyone who's played Scrabble online or competitively is probably familiar with the nonsense Scrabble words you have to memorize to play effectively. Especially important are the ones that help you use Q, X, and Z, and 2 letter words that let you attach to another word: qi, za, qats, mbaqanga. Your spell-checker doesn't have those words, and your dictionary probably doesn't either. You will never use them in a sentence, and you probably won't ever encounter anyone else using them either - unless you're playing or talking about Scrabble.

Memorizing and searching through lists of arbitrary, otherwise meaningless items isn't something humans are very good at. Performing precise calculations isn't something humans are very good at either. They are, however, tasks that computers are particularly good at.

This drives me crazy.

I've been programming for most of my life, so for many games, coding something that can play is more interesting than actually playing myself. I've written bots for Scrabble, Boggle, Sudoku, Poker, and all manner of word/card/number games - many for money.


## Lexis


<img src="/wp-content/uploads/2009/12/lexis-165x300.jpg"
     alt="Lexis game machine"
     style="float: right; margin-left: 10px;">

Recently, a friend introduced me to a word game on Second Life called Lexis. Lexis is basically 10 rounds of single-word Scrabble. You get 7 letters, and 7 spots to place them in, with the familiar bonus tiles: double word, triple word, double letter, triple letter. Your word must start in the left-most spot, so there is no strategy in how you place the word. The only thing to do is choose the highest-scoring word, taking into account the bonus squares, and input it as fast as possible.

Definitely a game for computers.

With a cash prize.

On one machine, the jackpot was over L$10,000 - which is about USD$35.

<!-- more -->


## Solving




### 'Sup, Dawg


When dealing with word games, a very common data structure is the Directed Acyclic Word Graph (DAWG). This structure is basically a tree where a node's children represent all of the letters that can possibly follow it to form a valid word. These structures are extremely efficient, both in terms of time to traverse and storage size.

I have a DAWG library I wrote in C++ a long time ago, which I've reused without modification on Windows, OSX, Linux, and FreeBSD. It even looks like it might work on MSVC6. You can download it from [my github](http://github.com/xau/test).

Using a DAWG, determining if a word is in a dictionary is almost as simple as a linear search through a list of strings:

~~~ cpp
bool
is_word_in_list(const vector<string>& list, const string& word)
{
  vector<string>::const_iterator i = list.begin();

  while(i != list.end() && *i != word) { ++i; }

  return i != list.end();
}

bool
is_word_in_dawg(const DAWG::DAWG& dawg, const string& word)
{
  DAWG::Iterator         di  = dawg.root();
  string::const_iterator si  = word.begin();

  while(si != word.end() && di != dawg.end()) {
    di = di.child();
    while(di != dawg.end() && di->letter() != *si) { ++di; }
    ++si;
  }

  return di->end_of_word();
}~~~

All we do is find the first letter of the word in the root node's children, then find the next letter of the word in that node's children, and so on until we run out of nodes or letters. If we ran out letters, we just need to make sure we stopped at a valid word ending (since, for example, we will find "ABJ" on the path to "ABJECT", but "ABJ" by itself is not a word).

Searching through a list of 10,000 words would take 10,000 string comparisons, whereas searching through the equivalent DAWG requires a maximum of 26 comparisons (one per letter) for each letter in the word: 182 for a 7-letter word. In practice it takes even fewer, because it will only contain letters that are allowed to follow another letter in a specific position. If the previous letter was a Q, there's only a handful of letters we need to check to know if it was a valid word. The Official Scrabble Player's Dictionary revision 3 (OSPD3) has about 80,000 words.


### Putting it to Use


Armed with a DAWG, it's pretty trivial to create a recursive algorithm that finds the longest word we can possibly create:

~~~ cpp
Word
find_best_word(const DAWG::Iterator start, const DAWG::Iterator& end,
               const string& letters, const Word& prev_best_word,
               const Word& prefix)
{
  Word best_word = prev_best_word;
  for( DAWG::Iterator di = start; di != end; ++di ) {
    size_t pos = letters.find(di->letter());
    if(pos!=string::npos) {
      Word new_word = prefix + di->letter();
      if(di->end_of_word() && new_word.score() > best_word.score())
        best_word = new_word;

      string new_letters = letters;
      new_letters.erase(pos, 1);

      best_word = find_best_word(di.child(), end, new_letters,
                                 best_word, new_word);
    }
  }
  return best_word;
}
~~~

When we first call this function, we'll pass it the beginning node in our DAWG and a blank best word and prefix. It will look through every possible first letter, and see if it's in the list of letters we can use. If so, we tack it onto the end of our starting word (which will be blank in this instance), and then we'll check if it's the end of a word as well (ex: "A", "I"). Then we check if it's better than the best word we know of, and make it the new best word if it is.

After that, we just start again with the child node (i.e. the tree of letters allowed to follow the one we used), the pool of letters sans the one we used, our best word (if we found one), and the word we've built up so far. Since our words are only going to be a maximum of 7 letters long, we don't have to worry about the stack getting out of hand; there will only be one stack frame for each letter in our word so far.


### Scoring Words


Of course, the longest word isn't what we need to find. We need to take into account the value of each letter, as well as the bonus squares on the board, and find the word with the highest score.

First we need a way to get the score of each letter:

~~~ cpp
static const uint letter_values[27] = {
/* a  b  c  d  e  f  g  h  i  j  k  l  m  n  o  p  q   r  s  t */
   1, 3, 3, 2, 1, 4, 2, 4, 1, 8, 5, 1, 3, 1, 1, 3, 10, 1, 1, 1,
/* u  v  w  x  y   z */
   1, 4, 4, 8, 4, 10
};
uint get_letter_score( char letter ) { return letter_values[ letter - 'a' ]; }
~~~

Then, we just need a way to score an entire word. Since the score is a property of the word, we'll just subclass string and a score method. There is only one issue: the individual letter scores are always the same, but the total word score will depend on the bonus squares, which change every round. Fortunately, since every word is built off of the prefix we pass in, we can simply set up the bonuses on that object and make sure any new objects created from it copy them.

~~~ cpp
class Word : public string {
  typedef pair<uint, uint> Square;

  public:
    Word operator+(const char c) const {
      Word new_word(*this);
      new_word.push_back(c);
      return new_word;
    }

    int score() {
      uint score = 0, mult = 1;
      for(size_t i = 0; i < length() && i < squares_.size(); ++i) {
        score += get_letter_score(at(i)) * squares_[i].first;
        mult *= squares_[i].second;
      }
      return (score * mult) + length();
    }

    void add_square(uint letter_mult, uint word_mult) {
      squares_.push_back(make_pair(letter_mult, word_mult));
    }

  private:
    vector<Square> squares_;
};
~~~

We add the length of the word to the total score, because Lexis gives us a bonus point for each letter if we finish within 10 seconds.

With this class, we can just change our strings to Words in our original function, and compare the score() instead of the length(). That's all we have to do; we can now calculate the best possible word for any round of Lexis.


### A Word on Efficiency


Doing everything at a high level like this is incredibly inefficient, but this game is so simple that the logic at this level doesn't matter. Also, modern compilers can do an incredible job of optimization, so many improvements we might make are already being done behind-the-scenes by the compiler. I created a test driver that loads a saved DAWG of OSPD4 and finds the best word from the letters on the command line. Loading the file and finding the solution completes in less than 1/100th of a second, even if I give it 100 letters to use.

More complex games may require extensive optimization, but it's best to start with an understandable, high-level solution, create some unit tests, set up profiling, and then go crazy optimizing it _once you're sure it's necessary_.

That said, I started backwards, and created a bare metal solver in C before I came back and wrote this version to make sure it worked correctly.


### Slot Machines


With what we have so far, we can write a simple interface that lets us type in the available letters and bonus squares and gives us back the best possible word and the best possible score. If you try playing a game with it, or watch someone else play and just see what the best they could have done is, you'll notice something:

You can't win.

While Lexis masquerades as a skill-based game, the letters and bonus square layout is randomized, and those factors determine whether it's possible to reach the minimum score required to win.

Or is it really all luck?

After playing a few times, it certainly begins to seem like scoring very high early on increases your odds of getting impossible or low-scoring sets of letters later. Noting that the payout to the previous winner was over L$16,000, and that the game only added 50% of your bet to the pot, I hypothesized that it would not even be possible to win until the pot reached L$15,000.

Note that [gambling is forbidden in Second Life](https://blogs.secondlife.com/community/features/blog/2007/07/26/wagering-in-second-life-new-policy), where gambling is defined as placing wagers that:


<blockquote>(1) (a) rely on chance or random number generation to determine a winner, OR (b) rely on the outcome of real-life organized sporting events,
AND
(2) provide a payout in
(a) Linden Dollars, OR
(b) any real-world currency or thing of value.</blockquote>




## Automating


Another thing you will eventually notice is that there is an aspect of Lexis that practically requires a computer: the speed bonus. As mentioned above, Lexis gives you a bonus point for each letter if you finish in under 10 seconds. Some of the machines will let you type your answer in chat, which makes this reasonably possible. The machine with the high jackpot requires a much more cumbersome method: click on a letter, click where it goes, and then click the timer when you're done.

Network latency and the performance of the Second Life client and server software make it take upwards of 300ms for each click to register. At two clicks per letter, we're looking at a minimum of 6 seconds to input a 5 letter word. Minus another second or two for the letters to even show up on our screen, that's not a lot of time to think. It's not even much time to relay the letters to a computer that can solve it a hundredth of a second.

Since the longest and most tedious part is clicking the letters, it's a natural place to start automating.


### Software


Having used the Second Life client for all three operating systems, I can say pretty confidently that the Windows client performs the best. Of the available clients, I prefer the third-party [Emerald viewer](modularsystems.sl/index.php).

To automate interactions with the client, I'll use a tool for Windows called [AutoIt](http://www.autoitscript.com/autoit3/index.shtml), which provides a BASIC-like scripting language to perform all sorts of UI interaction. It is useful for far more than cheating at silly word games, and should definitely be in the toolbox of any Windows power user or administrator.

Getting set up to compile C++ is easier on Linux than Windows, so I just used [putty](http://www.chiark.greenend.org.uk/%7Esgtatham/putty/download.html) to connect to a Linux server and run the solver application there. Another possibility would be using something like [Dev-C++](http://www.bloodshed.net/devcpp.html) and creating a native Windows app.


### Communication


AutoIt isn't very good at getting much information from a putty window. One thing that's easy to set, and that AutoIt is really good at reading, is the window title. We can set the title just by sending an escape code to stdout:

~~~ cpp
static void
set_title(const char* str)
{
    std::cout << "\033]0;" << str << "\007";
}
~~~


### Where to Click


We could go crazy setting up some kind of complicated screen scraping to automatically figure out where we need to click, but that's pretty excessive. AutoIt comes with a [Window Information Tool](http://www.autoitscript.com/autoit3/docs/intro/au3spy.htm) which we can use to manually find the right mouse coordinates. All we need is a script in Second Life to set our camera to a fixed position (squished for brevity):

~~~ lsl
list stored_params = [];
save_cam() {
    stored_params = [CAMERA_POSITION, llGetCameraPos(),
        CAMERA_FOCUS, llGetCameraPos()+llRot2Fwd(llGetCameraRot())]; }
set_cam() {
    llSetCameraParams(stored_params + [CAMERA_ACTIVE, 1,
        CAMERA_POSITION_LOCKED, 1, CAMERA_FOCUS_LOCKED, 1]); }
get_perms() {
    llRequestPermissions(llGetOwner(),
        PERMISSION_CONTROL_CAMERA|PERMISSION_TRACK_CAMERA); }
default {
    state_entry() {
        get_perms(); }
    attach(key id) {
        if(id==NULL_KEY) llClearCameraParams();
        else get_perms(); }
    run_time_permissions(integer perms) {
        if(stored_params != []) set_cam(); }
    touch_start(integer num) {
        save_cam(); set_cam(); }
}
~~~

We can drop this in a HUD object and then click it to lock our camera position. Detach to unlock, and re-attach to restore it to the locked position. This way we can be sure we'll only have to get the coordinates for the buttons once.

We'll need the coordinates for each of the letters, the spots to place them in the word, and the timer at the bottom. Since they're lined up in rows, we only need 3 different Y coordinates. These will be in our AutoIt script:

~~~ vb
Dim Const $Y_LETTER    = 219;
Dim Const $Y_WORD      = 297;
Dim Const $Y_DONE      = 355;
Dim Const $X_DONE      = 805;
Dim Const $X_LETTER[7] = [716, 755, 784, 832, 860, 894, 932];
Dim Const $X_WORD[7]   = [701, 741, 781, 821, 861, 901, 941];
~~~

Finally, we need our solver to tell the AutoIt script what order to click the letters in. The script doesn't know anything about the letters themselves, so we can't just give it the word and except it to figure out what to click. We'll set the window title to a comma-separated list of letter indexes, plus a special string to let AutoIt find our window:

~~~ cpp
string get_order(const string& word, string letters) {
  ostringstream stream;
  for(size_t i = 0; i < word.length(); ++i) {
    size_t letter_pos = letters.find(word[i]);
    letters.replace(letter_pos, 1, 1, '_');
    if(i>0) stream << ",";
    stream << letter_pos;
  }
  return stream.str();
}
/* ... */
set_title("_READY_," + get_order( best_word, letters ));
~~~

Our AutoIt script will just wait until it sees a window with our special _READY_ string, then interpret the rest of the title as a comma-separated list of letter indexes. It will click the letter, then the next position in the word, and when there are none left, it will click the timer to submit the word.

~~~ vb
WinWait( "_READY_" );
$title = WinGetTitle( "_READY_" );
$ary = StringSplit($title, ",", 2);
$pos = 0;
for $el in $ary
  if StringIsInt($el) then
    Sleep(300);
    MouseClick( "left", $X_LETTER[$el], $Y_LETTER, 1, 0 );
    Sleep(300);
    MouseClick( "left", $X_WORD[$pos], $Y_WORD, 1, 0 );
    $pos += 1;
  endif
next
Sleep(700);
MouseClick( "left", $X_DONE, $Y_DONE, 1, 0 );
WinActivate("_READY_");
Send("{ENTER}");
~~~

This script also switches back to the putty window at the end, so that we can just start typing the next word when the time comes. I've also made it press enter for me; I have the solver app set up to clear the title when a blank line is entered, so that I can put this script in a loop without it trying to submit the same word over and over.


### Input


<img src="/wp-content/uploads/2009/12/lexis2-300x169.jpg"
     alt="Solver in action"
     width="300"
     style="display: block; margin: 0 auto;">

I created a simple command-line interface to the solver, which expects lines containing 14 characters: 7 for the letters that are available, and 7 to specify the bonus squares. I use 1 for normal squares, 2 and 3 for letter multiples, and @ (shift+2) and # (shift+3) for word multipliers. This works well enough, but I could only reliably get the speed bonus for words up to 4 letters.

I'd usually stumble trying to input the bonus squares correctly, so I tried making them a little easier to input. I made it assume any squares I didn't specify were normal squares, and made a hyphen in the middle skip however many squares needed to be there. So, for example, I could type "12-21" instead of "1211121". This actually ended up slowing me down, since I'd end up hesitating to decide if I should use a hyphen or type out the full thing.

So the next candidate for automation is figuring out the bonus squares.


### The Colors


We already have the coordinates of the bonus squares on the screen, and they're color coded. Using the AutoIt Window Information tool we can figure out what the color for each kind of bonus is, and then use the [PixelGetColor](http://www.autoitscript.com/autoit3/docs/functions/PixelGetColor.htm) function to check them in our script.

~~~ vb
Func GetValue($v)
  $r = BitShift(BitAnd($v, 0xFF0000), 16)
  $g = BitShift(BitAnd($v, 0x00FF00), 8)
  $b = BitAnd($v, 0x0000FF)
  If ($r>0x70) And ($r<0x90) And ($g>0xD0) And ($b>0xD0) Then Return "2"
  If ($r>0xD0) And ($g>0xA0) And ($g<0xC0) And ($b>0xA0) And ($b<0xC0) Then Return "@"
  If ($r>0x50) And ($r<0x70) And ($g>0x50) And ($g<0x70) And ($b>0xD0) Then Return "3"
  If ($r>0xC0) And ($g<0x20) And ($b<0x20) Then Return "{#}"
  Return "1"
EndFunc

Func GetPixels()
  $v = ""
  For $x in $X_WORD
    $v &= GetValue(PixelGetColor($x,$Y_WORD))
  Next
  Send($v & "{ENTER}")
EndFunc

HotKeySet( "/", "GetPixels" );
~~~

With this active, simply pressing / will automatically type in the characters representing the bonus squares. Note that sometimes the pixels will be blurred or faded slightly, so the script checks for a range in each component rather than looking for an exact match.

After enabling this shortcut, I was able to get the speed bonus on every word unless lag prevented the clicks from going through correctly in time.


## The Results


The minimum score to win was 415. After about 20 games, when the jackpot was L$12,350, I got a very nice 120 point word (wizened) in the last round. Unfortunately that only boosted my score to 389. As the jackpot got higher and higher I started consistently scoring in high 300s, but never more than that round.

Remember, my hypothesis was that it was not possible to win until the jackpot exceeded L$15,000.

After around 90 games, pumping over L$9,000 into the machine, playing a perfect game every single round, I won with a score of 429.

The jackpot was at exactly L$14,950.
