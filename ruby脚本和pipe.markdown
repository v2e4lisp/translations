
**原文: ** [Writing Ruby Scripts That Respect Pipelines](http://www.jstorimer.com/blogs/workingwithcode/7766125-writing-ruby-scripts-that-respect-pipelines) |
**作者: ** Jesse Storimer

# [翻译]ruby脚本和pipe

在命令行里面管道是一个很重要的概念。

利用管道你可以把一些小的命令拼接起来，形成一个pipeline。这个和Unix的设计哲学有关。
> Do one thing, and do it well.


请看下面这个例子：

    # Look for any lines mentioning 'user' in the current git diff
    # and display them one page at a time using less(1).
    $ git diff | grep user | less
   
## 一个简单Ruby脚本

我写了一个叫做hilong的脚本。

它的用途也很简单。接受一个文件当参数，然后再把这个文件打印出来，在每一行的前面显示这一行的长度并且，对于那些长度超过80个字符的，把它输出成红色。下面就是这个脚本最初的版本。

    #!/usr/bin/env ruby

    # A file is passed in as an argument
    input = File.open(ARGV[0])

    # escaped bash color codes
    red = "\e[31m"
    reset_color = "\e[0m"

    maximum_line_length = 80

    # For each line of the input...
    input.each_line do |line|
      # Construct a string that begins with the length of this line
      # and ends with the content. The trailing newline is #chop'ped 
      # off of the content so we can control where the newline occurs.
      # The strings are joined with a tab character so that indentation
      # is preserved.
      output_line = [line.length, line.chop].join("\t")

      if line.length > maximum_line_length
        # Turn the output to red starting at the first character.
        output_line.insert(0, red)
        # Reset the text color back to what it was at the end of the
        # line.
        output_line.insert(-1, reset_color)
      end

      $stdout.puts output_line
    end
    
然后我们运行下看看。

    $ hilong Gemfile
    17    source :rubygems
    1
    13    gem 'jekyll'
    82    gem 'liquid', '2.2.2'         # The comment on this line is long-winded, not sure why... 
    16    gem 'RedCloth'  
    
这个脚本运行得挺好的，输入一个文件，然后处理每一行，最后输出到`$stdout`。接下来，让我们来看看，如何将它和其他的linux命令结合在一起使用吧。

## Pipe

首先，让我们试着把这个脚本的输出pipe到另一个命令。

    # Only show 'gem' lines.
    $ hilong Gemfile | grep gem

运行的很好！我们把这个脚本的结果输出到`$stdout`， grep(1)可以从此读取字符串然后过滤。连我们原先输出的红色也保留了。

好的，再试试别的看下。

    # View the output one page at a time.
    $ hilong Gemfile | more

额...那些用来设定颜色（红色）的字符串也输出出来了。好像`more(1)`并不会对bash的颜色的字符串最特殊处理，所以它把这些字符串也打印出来了。如果你把这个hilong这个命令的输出写入文件的，也会出现同样的情况。

所以，当你把输出pipe到另外一个程序的时候，你应该总是输出没有特殊颜色处理的字符串。

## TTY

如果有pipe的话，我们不能把颜色字符串输出给下一个程序。但是再没有pipe的情况下，我们又想直接打印出有颜色的字符。怎么办呢？

Ruby的`IO#isatty`方法（也叫做`IO#tty?`) 可以帮我们检测正在使用的IO有没有被绑定到terminal上。如果是你的`$stdout`被pipe到下一个命令的话，那么刚才的方法就会返回`false`。

我们现在重现刚才hilong那个脚本，让它可以正确的输出颜色字符。修改的部分如下：

    # If the line is long and our $stdout is not being piped then we'll
    # colorize this line.
    if $stdout.tty? && line.size > maximum_line_length
      # Turn the output to red starting at the first character.
      output_line.insert(0, red)
      # Reset the text color back to what it was at the end of the
      # line.
      output_line.insert(-1, reset_color)
    end 

    $stdout.puts output_line
    
好的，如果现在我们把hilong这个命令的输出pipe到`more`或者一个文件里面，刚才出现的那些颜色字符串就消失了！

## 这还没完！

大部分的unix命令都可以利用pipe把上一个命令的输出作为输入。试一下我们的这个脚本吧。

    $ cat Gemfile Gemfile.lock | hilong 
    /Users/jessestorimer/projects/hilong/hilong:4:in `initialize': can't convert nil into String (TypeError)
    from /Users/jessestorimer/projects/hilong/hilong:4:in `open'
    from /Users/jessestorimer/projects/hilong/hilong:4:in `<main>'
:/

到目前为止，我们的这脚本这能接受从`ARGV`过来的文件名作为输入。怎么让它接受pipe过来的数据呢？

Ruby有一个很的参数叫做`ARGF`。`ARGF`可以用来读取pipe过来的原始数据，同时也可以读取`ARGV`里面的文件。那我们来修改下我们的脚本吧。

    # Read input from files or pipes.
    -input = File.open(ARGV[0])
    +input = ARGF.read

OK！如果`ARGV`里面又值的话，`ARGF`会认为它是一个文件名，然后调用`IO#read`来一个一个读取文件的内容。如果`ARGV`里面没有参数的话，它就会去`$stdin`里面读取从pipe里传过来的原始数据。

在有文件名作为参数的情况下，Unix的命令不会从标准输入里面读取数据。

我们的这个脚本现在可以用多种方式调用的。

    $ hilong Gemfile
    $ hilong Gemfile | more
    $ hilong Gemfile > output.txt
    $ hilong Gemfile Gemfile.lock
    $ cat Gemfile* | hilong
    $ cat Gemfile | hilong - Gemfile.lock
    $ hilong < Gemfile
   
## 还有一种情况没有考虑到！

经过几次小的改动，我们的脚本现在可以很好的和其他unix命令结合起来使用了。但是还有一种情况没有考虑到。设想，如果我们的hilong从一个pipe里面源源不断的读取数据，比如`tail -f`，会发生什么呢？

    $ tail -f log/test.log | hilong

执行上面的命令，我们可以看到什么都没有被打印出来。

这是因为我们使用了`ARGF#read`。`#read`会一直读知道发现`EOF`，这个表示文件结束的字符才停止。但是另一方面，`tail -f`又不会输出`EOF`。所以我们的hilong也就不会有输出了。我们应该改变下，从`ARGF`里读取数据的方式。

我们现在用`#each_line`这个方法，每次从ARGF里面读取一行字符串。修改如下。

    # Keep reading lines of input as long as they're coming.
    ARGF.each_line do |line|
      # Construct a string that begins with the length of this line
      # and ends with the content. The trailing newline is #chop'ped 
      # off of the content so we can control where the newline occurs.
      # The string are joined with a tab character so that indentation
      # is preserved.
      output_line = [line.size, line.chop].join("\t")

好的，现在我们的这个命令不但可以接受上面的调用方式，对于连续的输入也是可以处理的。

## 更新
下面有一个评论说，还有一种我们没有考虑到的情况。他给下面这个例子。

    $ cat /dev/urandom | base64 -b 80 | hilong | head

这个pipeline有什么特殊的地方吗？我们的hilong的输出被pipe到了`head(1)`这个命令里，`head(1)`会读去前十行，同时关闭掉这个pipe。

如果你在shell里面跑上面的命令的话，你会看到hilong抛出了一个异常`Broken pipe - <STDOUT> (Errno::EPIPE)`。这是因为hilong认为这个pipe在它把输出打印完之前是没有被关闭的，所以它接着向这个关闭的pipe里面写数据的时候，就会报错！

解决的办法是，把直接写到STDOUT里面的那段代码放到一个`begin block`里面，然后抓住抛出的异常就可以了。

    begin
      $stdout.puts output_line
    rescue Errno::EPIPE
      exit(74)
    end

`exit(74)`用错误码74退出程序。根据`sysexits(3)`，这个错误码表示一个和IO相关的错误。用在这里感觉还算合适。

现在你把hilong的输出pipe到`head`就不会报错了。

完整的代码请看这个[gist](https://gist.github.com/jstorimer/1465437)
