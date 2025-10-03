# UNIX System Programming Main Programming Assignment, Part 1
* 說明
此程式是csh script以及sed來擴充原有sed的功能，將其稱為msed，而擴充的內容主要是增加新的sed 指令，或是修改原有sed指定的功能。

* 執行方式
和原有sed指令執行方式相同，要提供msed程式一個input以及sed指令，而他會輸出對input處理後的結果
```
% echo ABC | ./msed 's/$2/\$3/;/$4/p' B b C -n
A$3C
%
```
* Assignment overview:

This assignment will extend the behavior of sed.

New commands:
f: Sets the flag that is used by t.
t能夠在flag被設為1時，跳到指定的flag，而f做到set flag

F: Resets the flag that is used by t.
F則進行reset flag

Z: Similar to how "z" zaps the Pattern Space (PS), "Z" will only zap up to
   (and including) the first \n. If there's no \n, then zap all of the PS.
Z把pattern space中，從開頭到第一個\n的內容都zap掉

W: Consider that "p" prints the whole PS, but "P" prints just the first line.
   Similarly, "w" writes the whole PS to a file, and so you'd think "W" would
   print just one line to the file. Well, now that is what "W" does.
w把所有pattern space內容寫到file中，W只寫到第一個\n為止

C: Changes the PS to the text the follows the C. But unlike "c", "C" does not
   have any control flow side effect.
只有做到，把pattern space內容換成C之後的內容。注意和c的不同

D: This modifies the behavior of traditional "D", because it now has consisent
   behavior regardless of whether there is a "\n" in the PS: First, one line 
   is deleted from the PS (regardless of whether that line is offset by a "\n"
   or is just the entire PS). Second, the sed program restarts - without
   loading in the next line. In other words, D has been changed so that it
   behaves consistently, and never behaves exactly like a d.
   Hint: If, before you do the D, you firt put a "\n" in the front of the PS
   (if there is no \n there already), then you'll get this desired behavior.
無論pattern space內有沒有\n，都是刪掉一行以後，以目前的pattern space進行重啟。原本的
D是，如果沒有\n，就如同d一樣，刪掉以後，會重啟並重讀一個input

i,a,c,C,w,W,r: all of these take the REST of the line as its argument. But not
   in this new version. Now a ";" will end the argument.
功能如同原本，但改成如果碰到;就會停止抓取資料作為他的argument，不然原本要碰到newline才會

$-1,$-2,etc: $ is the last line of the file. But what about the 2nd-to-last?
   It is $-1. 3rd-to-last? $-2. Etc.
代表倒數第幾行，$是倒數第一行，$-1是倒數第二行，$-2是倒數第三行

$2,$3,etc: These now refer to passed-in arguments. This needs clarification,
   in the footnotes below.


 -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -


A second file posted with this README file is msed_template. If you look in
that file, you will see the __________ parts which you need to fill in.

You will also see a few place where that file references "footnotes in the
README file". Here they are:

Footnote 1: Why are the arguments $2, $3, ... defined, but not $1? And why
            does the msed template file say that $1 gets store into t5?
            The answer is that I have made a simplification to the assignment;
            the first argument is always the program. Moreover, the input file
            is never an argument. So, in our version of sed, it can ONLY be
            run like this:
                % cat infile | ./msed '<program>' flags or arguments.
         $n所代表的參數是從2開始算，而$2是第一個參數

Footnote 2: What do the $2, $3, etc really mean? Well, consider this:
                % echo ABC | ./msed 's/$2/$3/;/$4/p' B b C -n
                AbC
                %
            So, as you see, the argument is replaced by its value.

            Now, there is a need to be careful with quoting. For example:
                % echo ABC | ./msed 's/$2/\$3/;/$4/p' B b C -n
                A$3C
                %

            The way this is accomplished is to not just check for the
            expression $[0-9]\{1,\}, but to also make sure that the character
            before the "$" is not a "\". But there is a little detail: what
            if the $ is the first character, so that you can't check the
            character before it? The answer is: to make sure that there IS a
            character before it -- which explians line 8 of the msed_template
            file.
            參數的運行，相當於是做$[0-9]\{1,\}($+0~9被match到一次以上)的檢查，但還要
            檢查$前面是否有\，如果有，就不會被當成參數
            但如果$是在開頭，就沒辦法檢查$前面是否有\，所以範例程式中的第八行，才在開頭
            加上一個space


Footnote 3: What is the strategy to handle backquoting?
            We need to be careful about "\;" or "\$3", etc -- and to not be 
            fooled by "\\;" or "\\$3".
            The solution is to:
            - First, convert all the "\\" into "\h" (because "\h" is a special
              character that will not occur in the infiles that test your 
              program with). But note: you need to use "\\\\" to get "\\".
            - Second, convert "\;" to "\f" (another special character that
              will not occur in the infiles).
            - Third, put a "\n" before all remaining ";".
              (Now, when you do this, there will never be two commands on one
              line anymore -- but there might be a single command spread over
              multiple lines (and this gets cleaned up by later parts of the
              msed_template file).
            要做反斜線的檢查，如果遇到\;不能把它當成指令的結束，同樣，如果遇到\$3也
            不能把它當成參數，等等，總之\之後的不會被當成指令。
            但有可能會出現\\;或\\$3的狀況，這時\會被當成一個符號，而非反斜線的功能，
            所以仍會展現出;或$3的功能
            作法:
            1. 把\\轉成\h，且\\\\會被視為\\
            2. 把\;轉成\f
            3. 在所有仍存在的;前面放上\n(這會讓一個command被分成多行，但範例程式後面會處理這個問題)
            

Footnote 4: What is the strategy to handle "F"?
            The "F" command needs to reset the flag. This is accomplished
            by do a branch. If you don't want to branch anywhere (you just
            want to reset the flag), then branch to the place where you are
            already (eg: ...;F;... => ...;bL;:L;...).
            But there is a big issue: You can have more than one "F", and
            they will need unique labels. And that is why Line 63 of the
            msed_template file adds a counter to the label.
            reset flag可以用branch來做到
