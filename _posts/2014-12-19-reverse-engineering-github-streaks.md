---
layout: post
title: Reverse engineering GitHub streaks
---

To say I'm addicted to GitHub is an understatement. But I've attempted to focus my addiction towards productive goals, and so I decided that I wanted to process GitHub streak data programmatically. To my dismay, streak data isn't exposed as part of their API, and my request that they add it was met with polite neutrality. So I set out to see how their site built the streak chart on [the user page](https://github.com/akerl).

When I began this adventure, I knew enough JavaScript to shoot myself in the foot and I'd never dealt with large existing JavaScript codebases, but I nonetheless dove in to Chrome's Developer Tools to dissect how the page created that chart. I got my first win from [Jon Chen](https://github.com/fly), who identified the source of the data: a JSON array of dates and scores served at https://github.com/users/{username}/contributions_calendar_data (a URL that no longer works, which we'll get to). This gave me the raw score data, and I got to work building a module around that.

<!--more-->

githubstats
===========

I'm a firm believer that small, focused libraries are the best foundation for solid code. In keeping with that, I first set to work building a module that consumed the raw streak data and exposed it with some helper methods. The result is [githubstats](https://github.com/akerl/githubstats). It was one of my first Ruby projects, and it turned out to be a pretty solid intro to Ruby programming.

I quickly realized that I was dealing with 2 different "objects", GitHub users and the underlying stats bundles, and architected my module accordingly. Unfortunately, this made using the module quite obnoxious: you'd have to create a GithubStats::User object, and accessing the data would require user.data.{method} for every call. The first thing I did to simplify this was adding a helper method to the module itself:

{% highlight ruby %}
module GithubStats
  ##
  # Helper method for creating new user objects

  def self.new(*args)
    self::User.new(*args)
  end
end
{% endhighlight %}

This ended up being a pattern I copied for most of my future modules: the module has a class method .new() that is proxied to the most commonly desired class.

To solve the user.data.{method} problem, I decided to do some more complicated proxying. Thankfully, I'd recently completed the [Ruby Koans](http://rubykoans.com/) (which I highly recommend), and one of the sections is about method_missing and how Ruby handles method calls. I decided to override method_missing and respond_to? on the User object, so that any method called on User that didn't exist but *did* exist on .data would be proxied through. Initially, I had it proxy the calls directly from method_missing, but I read that the process of checking method_missing was fairly intensive, compared to making direct method calls, so I settled on a solution:

{% highlight ruby %}
def respond_to?(method, include_private = false)
    load_data if @last_updated.nil?
    super || @data.respond_to?(method, include_private)
end

def method_missing(sym, *args, &block)
  load_data if @last_updated.nil?
  return super unless @data.respond_to? sym
  instance_eval "def #{sym}(*args, &block) @data.#{sym}(*args, &block) end"
  send(sym, *args, &block)
end
{% endhighlight %}

This is, in theory, better for performance at the cost of some code cleanliness. For example, the first time user.mean is called, method_missing is invoked, and it confirms that user.data will respond to .mean. It then dynamically adds a method to the user object for user.mean, which calls @data.mean, and then calls that method. Future calls to user.mean will just use the newly created method directly, rather than hitting method_missing. At the advice of the koans, I also overrode respond_to?(), so that it accurately reflects what calls will succeed.

Having completed an initial version of the data library, I set out to find an actual use for it.

githubchart
===========

In an interesting turn of events, I decided that it would be awesome to have my own SVG copy of the GitHub contribution streak chart. At the time, I figured it would be cool to display on my blog or elsewhere. With this in mind, I started work on [githubchart](https://github.com/akerl/githubchart).

Going into this project, I expected to spend some time learning how to draw SVGs in Ruby, a decent amount of time figuring out how to handle command line utilities, and about 3 seconds checking GitHub to figure out where the breakpoints between the various colors fell. My estimates were shockingly terrible.

Drawing SVGs in Ruby is pretty easy. I ended up using [rasem](https://github.com/aseldawy/rasem), which is a pretty solid library, despite missin support for some fun things like title attributes that I wish it had. The command line tool was similarly easy, I dropped OptionParser in, created a GithubChart::SVG object, and dropped the result into a file. The massive timesink turned out to be deciphering GitHub's color picking logic.

It turns out that they don't use static break-points between color levels. That part was immediately apparent. After realizing that, I grabbed a bunch of peoples' data and tried brute-forcing it, expecting to find some linear algorithm or other obvious pattern. That was a total failure. From there, I decided to dig in to their actual Javascript. It was minified to the point of insanity, but over the course of several weeks I broke it apart and commented it out:

{% highlight javascript %}
var t, e, n, r, a, s, i, o, c, l;
c = 721,                                                # Width of the graph
e = 110,                                                # Height of the graph
l = [20, 0, 0, 20],                                     # Only used for s/a/n/r below
s = l[0],                                               # Y offset of top of graph
a = l[1],                                               # Not used
n = l[2],                                               # Not used
r = l[3],                                               # X offset of top of graph
t = 13,                                                 # Cell size
i = 2,                                                  # Cell padding
o = function(t) {                                       # Calculates standard deviation(?) for outlier math
    var e, n, r, a, s;                                  
    if (r = t.length, 1 > r)
        return 0 / 0;
    if (1 === r)
        return 0;
    for (n = d3.mean(t), e = -1, a = 0; ++e < r; )
        s = t[e] - n, a += s * s;
    return a / (r - 1)
},
$(document).on("graph:load", ".js-calendar-graph", function(n, a) {
    var l, u, d, h, f, m, p, g, v, b, y, j, w, x, C, k, S, _, T, D, P, A, L, E, M, B, I, O, q, F;
    for (
            l = $(this),
            f = l.attr("data-from"),                    # Set if we asked to hilight a specific date
            f && (f = C = moment(f).toDate()),          # Turn that into a date if it exists
            A = l.attr("data-to"),                      # This never appears to be used
            A && (A = moment(A).toDate()),              # Neither does this
            a || (a = []),                              # If we got no contrib data, give us an empty array
            a = a.map(function(t) {                     # Convert the points to [Date, Score] pairs, sorted by date
                    return [new Date(t[0]), t[1]]
                }).sort(function(t, e) {
                    return d3.ascending(t[0], e[0])
                }),
            u = 3.77972616981,                          # Magic number for finding outliers
            w = a.map(function(t) {                     # Array of just the scores
                    return t[1]
                }),
            T = Math.sqrt(o(w)),                        # Square root of o() above, used for finding outliers
            b = d3.mean(w),                             # Mean of the scores
            _ = 3,                                      # This is our loop max
            p = d3.max(w),                              # Max score
            L = p - b,                                  # Max minus the mean
            (6 > L || 15 > p) && (_ = 1),               # If (max-mean) is greater than 5 or the max is greater than 15, drop the loop max to 1
            x = 0                                       # Start the loop counter at 0
        ;
            _ > x;                                      # Loop check
        )
            E = w.filter(function(t) {                  # Grab any outliers
                    var e;
                    return e = Math.abs((b - t) / T), e > u
                }),
            E.length > 0 ?                              # Did we find any outliers?
                (
                    E = E[0],                           # If so, pop the first one
                    w = w.filter(function(t) {          # Now remove any instances of *that* number from the scores
                        return t !== E          
                    }),
                    0 === x && (g = w)                  # If this is the first oulier, set g to the new list
                )
            :
                E = null,                               # If we have no outliers, null E
            x += 1;                                     # Increment the loop counter and spin again
        return
{% endhighlight %}

The first realization was that they're doing some outlier math. I later learned that this is two-pass variance, but at the time I just knew it looked weird and set to work replicating it:

{% highlight ruby %}
    ##
    # The standard variance (two pass)

    def std_var
      first_pass = @raw.reduce(0) { |a, e| (e.score.to_f - mean)**2 + a }
      Math.sqrt(first_pass / (@raw.size - 1))
    end

    ##
    # Outliers of the set

    def outliers
      return [] if scores.uniq.size < 5
      scores.select { |x| ((mean - x) / std_var).abs > GITHUB_MAGIC }.uniq
    end

    ##
    # Outliers as calculated by GitHub
    # They only consider the first 3 or 1, based on the mean and max of the set

    def gh_outliers
      outliers.take((6 > max.score - mean || 15 > max.score) ? 1 : 3)
    end

    ##
    # The boundaries of the quartiles
    # The index represents the quartile number
    # The value is the upper bound of the quartile (inclusive)

    def quartile_boundaries # rubocop:disable Metrics/AbcSize
      top = scores.reject { |x| gh_outliers.include? x }.max
      range = (1..top).to_a
      range = [0] * 3 if range.empty?
      mids = (1..3).map do |q|
        index = q * range.size / 4 - 1
        range[index]
      end
      bounds = (mids + [max.score]).uniq.sort
      [0] * (5 - bounds.size) + bounds
    end
{% endhighlight %}

You'll note that they do some weird things with outlier counts. They bump a set number of outliers, and any numbers that match those outliers. So if your outliers are, in order, "20, 30, 40, 50, 100, 20", then any instances of 20, 30, and 40 are rejected from the set, leaving 50 and 100 behind. That tripped my up for quite a while until I really dug in to what the JS was doing.

The next component was getting the layout of the SVG right. Doing the grid squares was pretty easy, but it turns out there's some cute math involved in the month labels to decide when they show up / where they show up. This all happens in the return statement, which happens right at the pottom of the last gist:

{% highlight javascript %}
        return
            v = d3.max(w),                              # The max of the non-outliers
            k = ["#d6e685", "#8cc665", "#44a340", "#1e6823"], # The array of color codes for the chart
            m = d3.scale.quantile().domain([0, v]).range(k), # Map function for converting scores to colors
            h = d3.time.format("%w"),                   # Time formatter for weekday as a decimal (0-6)
            F = d3.time.format("%Y%U"),                 # Time formatter for "${year}${week_number}", with year as a 4 digit number
            q = d3.time.format("%m-%y"),                # Time formatter for month-year, each as a 2 digit integer
            y = d3.time.format("%b"),                   # Time formatter for abbreviated month name
            I = {},                                     # Keys are "$year$week_number" from F above, values are [Date, Score] pairs
            j = {},                                     # Keys are "$month-$year" from q above, values are a count of how many $week_numbers begin in that month
            a.forEach(function(t) {                     # Function to set I above
                var e;
                return
                    e = F(t[0]),                        # Get "$year$week_number" for this day
                    I[e] || (I[e] = []),                # Default I[e] to an empty list, if it's not already been initialized
                    I[e].push(t)                        # Push this [Date, Score] pair onto the appropriate key's array
            }),
            I = d3.entries(I),                          # Convert associative array to array of objects with {key, value} attributes
            I.forEach(function(t) {                     # Function to set j above
                var e;
                return
                    e = q(t.value[0][0]),               # Get "$month-$year" for first day of this week (noteworthy: January 1st is a different week from December 31st, even if they fall on the same calendar line)
                    j[e] || (j[e] = [t.value[0][0], 0]),# If we've not already initialized this key, create it with [First_Date, 0]
                    j[e][1] += 1                        # Increment this month's counter by one
            }),
            j = d3.entries(j).sort(function(t, e) {     # Convert hash of j[$month-$year] = [first_day_a_week_starts_in_this_month, counter_of_weeks_that_begin_in_this_month] to array of {key, value} objects
                    return d3.ascending(t.value[0], e.value[0])
                }),
            P = d3.tip().attr("class", "svg-tip").offset([-10, 0]).html(function(t) { # Used to make the tooltips for the grid
                    var e;
                    return
                        e = 0 === t[1] ? "No" : t[1],   
                        "<strong>" + e + " " + $.pluralize(t[1], "contribution") + "</strong> on " + moment(t[0]).format("MMMM Do YYYY")
                }),
            M = d3.select(this)                         # Time to build the graph
                .append("svg")                          # Add the SVG
                .attr("width", c)                       # With the right width
                .attr("height", e)                      # And the right height
                .attr("id", "calendar-graph")           # Add the CSS ID
                .append("g")                            # GitHub uses SVG groups to line up the cells
                .attr("transform", "translate(" + r + ", " + s + ")") # This does the offset, shifting the start of the cells 20 units right and 20 down
                .call(P),                               # Adds P above as a callback for mouseover
            B = 0,                                      # Counter for O below
            D = (new Date).getFullYear(),               # Get the current year
            O = M.selectAll("g.week")                   # Lets build out the groups
                .data(I)                                # Using our array of {key: ${year}${week_number}, value: [[Date, score], ...]} objects
                .enter().append("g")                    # Add a group for each week
                .attr("transform", function(e, n) {     # Add the transform to position it
                    var r;
                    return
                        r = e.value[0][0],              # Set r to the date of the first day of the week
                        r.getFullYear() === D && 0 !== r.getDay() && 0 === B && (B = -1), # If this week started in this year and this day isn't Sunday and B is 0, set B to -1
                        "translate(" + (n + B) * t + ", 0)" # Shift this group ($index + $offset) * $cell_size to the right
                }),
            S = O.selectAll("rect.day").data(function(t) { # Now lets build out each day's cell
                    return t.value                      # Using the value for that day
                }).enter().append("rect")               # Make a new rectangle
                .attr("class", "day")                   # Add the CSS class
                .attr("width", t - i)                   # Set the width to cell size minus padding
                .attr("height", t - i)                  # Likewise for the height
                .attr("y", function(e) {                # Set the y coord to $weekday * $cell_size
                    return h(e[0]) * t
                })
                .style("fill", function(t) {            # Set the color based on the quartile map function
                    return 0 === t[1] ? "#eee" : m(t[1])
                })
                .on("click", function(t) {              # Set the on-click action to hilight the cell
                    return $(document).trigger("contributions:range", [t[0], d3.event.shiftKey])
                })
                .on("mouseover", P.show)                # Set the mouseover action to show the tooltip
                .on("mouseout", P.hide),                # And the mouseout to hide it
            d = 0,                                      # Offset for use in making month labels
            M.selectAll("text.month")                   # Now we're working on months
            .data(j)                                    # Using the $month-$year mapping
            .enter().append("text")                     # Make the text labels
            .attr("x", function(e) {                    # Set the x coordinate to $cell_size * $offset
                var n;
                return
                    n = t * d,                          # Calculate $cell_size * $offset
                    d += e.value[1],                    # Then increment the offset by the size of this month
                    n                                   # Then return the previously calculated value
            })
            .attr("y", -5)                              # Set the y coord
            .attr("class", "month")                     # Class it up
            .style("display", function(t) {             # If it would be on the very left edge, hide it
                return t.value[1] <= 2 ? "none" : void 0
            })
            .text(function(t) {                         # The text is the abbreviated month name ("Jan")
                return y(t.value[0])
            }),
            M.selectAll("text.day")                     # Now build the text for weekdays
            .data(["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"]) # Define our weekdays
            .enter().append("text")                     # They're text objects
            .style("display", function(t, e) {          # Only show the odd-index days (Monday, Wednesday, Friday)
                return 0 === e % 2 ? "none" : void 0
            })
            .attr("text-anchor", "middle")              # Set some CSS properties for display
            .attr("class", "wday")                      # And a class for formatting
            .attr("dx", -10)                            # Set the x coord
            .attr("dy", function(e, n) {                # The y coord is $graph_y_offset + ($index - 1) * $cell_size + $padding + $cell_offset
                return s + ((n - 1) * t + i)
            })
            .text(function(t) {                         # And the text is the first letter of the weekday (why did we start with 3 letters anyways?)
                return t[0]
            }),
            f || A ? $(document).trigger("contributions:range", [f, A]) : void 0 # if we're hilighting a specific day, call that
{% endhighlight %}


The very important thing to note, which threw me off for a long time, is that they use $week_number. This skews results for January, because January 1st is always the start of a new week_number, even if it falls mid-week. They count how many week_numbers start in each month and give each month label that many columns of space. They also check the front/end and hide the first/last label if they're going to overlap other labels.

I spent a lot of time just tweaking pixel counts, and the end result was pixel-perfect matching to GitHub's graphs. For now. I seem to keep finding things that I need to tweak.

Next steps
==========

I've already made some major tweaks to the code, some based on feedback and some based on whimsy.

Better pipeline support
-------------------

I've made some changes and merged some PRs in response to [Stan's](http://www.schwertly.com/2014/09/creating-github-style-contribution-graphs-for-anything/) usage, which involves pumping self-generated data in to GithubChart. To support this, he submitted [a pull request](https://github.com/akerl/githubchart/pull/31) to support providing score data on STDIN.

He also is a big fan of piping together commands, so I [added support](https://github.com/akerl/githubchart/commit/d99e6a465e445786a232607e4ae87b250ee81012) for printing the SVG to STDOUT rather than to a file.

Alternate colors
----------------

I baked in support for changing the colors for the chart from the start, but didn't expose that to the script. Then on Halloween, GitHub switched up their color scheme to more festive colors, and I of course needed to follow suit. The [resulting change](https://github.com/akerl/githubchart/commit/7c90bcc9b07965cb978a11a29981a03433aa7ab2) turned out to be pretty easy: I grabbed the new colors, added an option to the script, and tweaked the defaults to support named color schemes.

Actual usage?
-------------

The one thing I've not yet done with the charts? Actually use them. I have used the data in a couple fun projects, like [this twilio endpoint](https://github.com/akerl/committed) that will tell me if I've committed today.

