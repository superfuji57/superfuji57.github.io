---
layout: post
title: "Scraping NBA Player Bios with R and the RVest Package"
categories: [projects]
excerpt: Using R to scrape, clean, and visualize data from NBA.com's shot log API
tags: [a, NBA, data visualization, data science]
data: 2015-03-25
---

The NBA.com/stats site is arguably the best sports statistics interface across all sports and sports leagues. Thanks to a couple of great posts by [Greg Reda](gregreda.com/2015/02/15/web-scraping-finding-the-api/) and [Daniel Forsyth](danielforsyth.me/exploring_nba_data_in_python/), I decided to see how I can use R to explore the NBA's player shot logs, which provides details for every field goal attempt taken over the current and last season.

As Greg and Daniel show in their posts, a numeric player ID is needed to query the NBA's API. It became clear early on that a list or table of IDs for all active players wasn't available. To build this, we'll have to scrape the NBA's stats site.

The site's [Player Index page](http://stats.nba.com/players/) should have what we need. Does it? Well, kind of. Since the page uses a client-side framework (which Greg explains wonderfully in his post), we don't see a lot of what we get in the browser when pulling data from `R`. At the time of writing this post, I am not aware of a method to mimic the actions that trigger the site to be fully rendered in the browser by -- in this case -- angular.js. If anyone knows of such wizardry, please do tell!

Instead, I went with the hard but mostly long way: brute force search. This means looping over every possible ID and building the table one player at a time.

After loading the necessary packages...


{% highlight r %}
library(rvest)
library(rjson) # to parse the results from the API
library(beepr) # cool package that triggers an alert when the script is finished
#library(httr)
library(dplyr) # easy data manipulation
library(data.table)
options(stringsAsFactors = FALSE)
{% endhighlight %}

Using the example player from Greg's post and the method he used to find shots API, we can the player bio API endpoint for John Wall. It looks like this:

> http://stats.nba.com/stats/commonplayerinfo?LeagueID=00&PlayerID=**202322**&SeasonType=Regular+Season

With the `html` function from `rvest`, we some information about John Wall mucked up in some html and JSON.


{% highlight r %}
player_info <- html(paste0(
        "http://stats.nba.com/stats/commonplayerinfo?LeagueID=00&PlayerID=",
        202322, #John Wall's Player ID
        "&SeasonType=Regular+Season"))

player_info
{% endhighlight %}



{% highlight text %}
## <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.0 Transitional//EN" "http://www.w3.org/TR/REC-html40/loose.dtd">
## <html><body><p>{"resource":"commonplayerinfo","parameters":[{"PlayerID":202322},{"LeagueID":"00"}],"resultSets":[{"name":"CommonPlayerInfo","headers":["PERSON_ID","FIRST_NAME","LAST_NAME","DISPLAY_FIRST_LAST","DISPLAY_LAST_COMMA_FIRST","DISPLAY_FI_LAST","BIRTHDATE","SCHOOL","COUNTRY","LAST_AFFILIATION","HEIGHT","WEIGHT","SEASON_EXP","JERSEY","POSITION","ROSTERSTATUS","TEAM_ID","TEAM_NAME","TEAM_ABBREVIATION","TEAM_CODE","TEAM_CITY","PLAYERCODE","FROM_YEAR","TO_YEAR","DLEAGUE_FLAG"],"rowSet":[[202322,"John","Wall","John Wall","Wall, John","J. Wall","1990-09-06T00:00:00","Kentucky","USA","Kentucky/USA","6-4","195",4,"2","Guard","Active",1610612764,"Wizards","WAS","wizards","Washington","john_wall","2010","2014","N"]]},{"name":"PlayerHeadlineStats","headers":["PLAYER_ID","PLAYER_NAME","TimeFrame","PTS","AST","REB","PIE"],"rowSet":[[202322,"John Wall","2014-15",17.7,9.8,4.7,0.152]]}]}</p></body></html>
## 
{% endhighlight %}

The `rjson` package comes in handy here to turn that into something useful. Nested inside is `html_text` from `rvest`, which will clean out the html tags. And, here we just want an outline, so we take the headers vector from John Wall's bio and store them in an object, `cols`.



{% highlight r %}
player_json <- fromJSON(html_text(player_info))
cols <- player_json$resultSets[[1]]$headers # taking the column names from the initial API test call
cols
{% endhighlight %}



{% highlight text %}
##  [1] "PERSON_ID"                "FIRST_NAME"              
##  [3] "LAST_NAME"                "DISPLAY_FIRST_LAST"      
##  [5] "DISPLAY_LAST_COMMA_FIRST" "DISPLAY_FI_LAST"         
##  [7] "BIRTHDATE"                "SCHOOL"                  
##  [9] "COUNTRY"                  "LAST_AFFILIATION"        
## [11] "HEIGHT"                   "WEIGHT"                  
## [13] "SEASON_EXP"               "JERSEY"                  
## [15] "POSITION"                 "ROSTERSTATUS"            
## [17] "TEAM_ID"                  "TEAM_NAME"               
## [19] "TEAM_ABBREVIATION"        "TEAM_CODE"               
## [21] "TEAM_CITY"                "PLAYERCODE"              
## [23] "FROM_YEAR"                "TO_YEAR"                 
## [25] "DLEAGUE_FLAG"
{% endhighlight %}

We can see that the API returns 25 fields. Next, we'll create an emtpy dataframe with 25 columns. For simplicity and consistency with `R` style, we'll use `tolower` to set all the column names into lower case.


{% highlight r %}
player_df <- data.frame(matrix(NA, nrow=1, ncol=25)) # empty dataframe
names(player_df) <- tolower(cols)
{% endhighlight %}

Now, on to the nasty part. To find out what the possible range for player IDs is I looked at a rookie (Andrew Wiggins, 203952) and a legend (Bill Russell, 78049). Pretty big gap. We want to get everybody -- and this is going to take a while -- so let's just go the whole distance to make sure we don't miss anyone. Something we hope to never do again, a scraping for loop from 1 to 300,000:



{% highlight r %}
for (i in 1:300000) {
        url <- paste0(
                "http://stats.nba.com/stats/commonplayerinfo?LeagueID=00&PlayerID=",
                i,
                "&SeasonType=Regular+Season")
        
        player_info <- try(html(url), silent=TRUE)
        if (!("try-error" %in% class(player_info))) {
                player_json <- fromJSON(html_text(player_info))
                # the API returns 25 different columns
                for (x in 1:25) {
                        if (!is.null(player_json$resultSets[[1]][[3]][[1]][[x]])) {
                                player_df[i,x] <- player_json$resultSets[[1]][[3]][[1]][[x]] 
                        }
                }
        }
        if (i %% 7 == 0) handle_reset(url)
}
beep(2) # Mario sound to let us know we're done
{% endhighlight %}

We're done! Still there? Let's clean up the dataframe of empty rows then convert it into a faster data.table.


{% highlight r %}
player_df <- filter(player_df, !(is.na(first_name)))
player.table <- data.table(player_df)
{% endhighlight %}



And for a final count...

{% highlight r %}
nrow(player.table)
{% endhighlight %}



{% highlight text %}
## [1] 5486
{% endhighlight %}

Over 5,000 player bios! A quick search for "how many players have played in the nba?" on Google tells us that there were likely 3,017 players in 50 years of the NBA as of January 2014.

Now we can really have fun with NBA.com player data. Next post coming soon!

[Github repo for this post](https://github.com/superfuji57/nba-playerIDs)
