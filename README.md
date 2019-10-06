---
title: "Calculating (Twitter) Vocabulary Breadth of U.S. Presidential Candidates Using TTR"
output: html_notebook
---

I was listening to a podcast that referenced Donald Trump's limited vocabulary in his tweets. It was implied that this was a fact -- that, relatively speaking, he used a limited number of unique words. I wondered if this was something that could be quantified and confirmed. That led to a passing discussion with [Joe Sutherland](https://twitter.com/J0eSutherland), the head of the data science department where I work, and he immediately noted, "That sounds like a TTR question." And he was right (I'd never heard of TTR, but, then again, I'm not a data scientist nor an NLP expert).

Calculating the TTR just for Trump's tweets would just be a number that was greater than 0 and less than or equal to 1, so I'd need some context. So, I also calculated the TTR for the 12 Democratic candidates who have made the cut to be on the debate stage on October 15, 2019.

## What is TTR?

The Type-Token Ratio (TTR) is a pretty simple calculation. It is a measure of how many _unique_ words are used in a corpus relative to the _total_ words in the corpus. There is a detailed write-up of the process [here](https://www.sltinfo.com/type-token-ratio/), but the formula is really simple:

$$\frac{Number\ of\ Unique\ Words}{Total\ Words}\times100$$

As a silly, simple example, let's use: `The quick brown fox jumps over the lazy dog`. The word "the" is the only word used more than once in this sentence.

$${Number\ of\ Unique\ Words}=8$$
$${Total\ Words}=9$$
$${TTR}=\frac{8}{9}\times100=88.9\%$$
A **low TTR** means that _there are more words that are used again and again_ in the data, while a **high TTR** (maximum of 100%) means that _few words are repeated in the dataset_.

## Setup to Get the Data

There's a little bit of setup required to use `rtweet`, in that a (free) app has to be created in the [Twitter Developer Console](https://developer.twitter.com/en/apps). The credentials for that app can then be stored in a `.Renviron` file and then read in with `Sys.getenv()`.

And, to make the code repurposable, we specify a single Twitter account that is the primary account of interest, and then a vector of accounts to use for context/comparison.

```{r setup}

# Load the necessary libraries. 
if (!require("pacman")) install.packages("pacman")
pacman::p_load(rtweet,        # How we actually get the Twitter data
               tidyverse, 
               scales,        # % formatting in visualizations
               knitr,         # Eye-friendly tables
               ggforce,       # Simpler labels on a scatterplot
               tidytext)      # Text analysis 

# Set users to assess
user_highlight <- "realdonaldtrump"   # The primary user of interest for comparison
users_compare <- c("joebiden", "corybooker", "petebuttigieg", "juliancastro", 
                   "tulsigabbard", "kamalaharris", "amyklobuchar", "betoorourke",
                   "sensanders", "tomsteyer", "ewarren", "andrewyang")

# Make a vector of all users to query
users <- c(user_highlight, users_compare)

# Add the "@" to the user_highlight for ease of use later
user_highlight <- paste0("@", user_highlight)

# Create the token. 
tw_token <- create_token(
  app = Sys.getenv("TWITTER_APPNAME"), 
  consumer_key = Sys.getenv("TWITTER_KEY"),
  consumer_secret = Sys.getenv("TWITTER_SECRET"),
  access_token = Sys.getenv("TWITTER_ACCESS_TOKEN"),
  access_secret = Sys.getenv("TWITTER_ACCESS_SECRET"))

# Go ahead and set up the theme we'll use for the visualizations
theme_main <- theme_light() +
  theme(plot.title = element_text(hjust = 0.5, face = "bold", size = 24),
        plot.subtitle = element_text(hjust = 0.5, face = "italic", size = 20),
        panel.border = element_blank(),
        axis.title = element_blank(),
        axis.text.y = element_text(face = "bold", size = 20),
        axis.text.x = element_blank(),
        axis.ticks = element_blank(),
        panel.grid = element_blank())
```

## Function to Calculate the TTR

Ultimately, we're looking for 200 tweets of original text (there's no magic to this -- it just seemed like a reasonable count to work with), so we pull ~500, remove the retweets, and see what we've got left. Because we're going to do this for multiple accounts, we set it up as a function that takes a username as an input and returns a single-row data frame with the username, the total words used in the 200 tweets, the number of _unique_ words used in those tweets, and the TTR.

```{r calc_ttr}
# Function to calculate the TTR
get_ttr <- function(username){
  
  # Get the tweets. For some reason, when you specify 500 it returns
  # 599. That's fine. We just want extras so we can discard retweets and
  # still have at least 200 left.
  tweets_df <- get_timeline(username, n=500)
  
  # Remove the retweets
  tweets_df <- tweets_df %>% 
    filter(is_retweet == FALSE)
  
  # Just work with the last 200
  tweets_df <- tweets_df[1:200,]
  
  # Work with just the text field. Remove Twitter usernames referenced
  # and links.
  tweets_text <- tweets_df %>% 
    select(text) %>% 
    mutate(text = gsub("@\\S+","", text)) %>% 
    mutate(text = gsub("https://t.co/.{10}", "", text))
  
  # Convert the tweet text into individual words. This drops punctuation
  # marks and, by default, converts all words to lowercase (which is what we want).
  all_words <- tweets_text %>% 
    unnest_tokens(output=word, input=text)
  
  # De-duplicate to get a list with the unique words
  unique_words <- all_words %>% 
    group_by(word) %>% 
    summarise()
  
  # Calculate the TTR
  ttr <- nrow(unique_words) / nrow(all_words)
  
  # Return a data frame with the details
  ttr_df <- data.frame(username = paste0("@",username),
                       unique_words = nrow(unique_words),
                       all_words = nrow(all_words),
                       ttr = round(ttr,3),
                       stringsAsFactors = FALSE)
}
```

# Get the Results
This is now pretty easy -- just call our function for all of the users.

```{r get_results}

# Calculate the TTR for all users
ttr_df <- map_dfr(users, get_ttr)

# Reorder descending by TTR and add columns that are JUST @realdonaldtrump 
# values for highlighting those values in plots
ttr_df <- ttr_df %>% arrange(ttr) %>%
  mutate(username = factor(username, levels = username),
         highlight_ttr = ifelse(username == user_highlight, ttr, NA),
         highlight_unique_words = ifelse(username == user_highlight, unique_words, NA),
         highlight_all_words = ifelse(username == user_highlight, all_words, NA))

# Take a quick look at the data
ttr_df %>% select(username, unique_words, all_words, ttr) %>% kable()
```

## Analysis of the Results

We can start with a simple bar chart showing the TTR for each user. 

```{r bar_chart_ttr, fig.height = 6, fig.width = 8, warning = FALSE}
gg <- ggplot(ttr_df, aes(x = username, y = ttr, label = scales::percent(ttr))) +
  geom_bar(stat = "identity", fill = "gray80") +
  # Highlight the user that is of interest
  geom_bar(stat = "identity", mapping = aes(y = highlight_ttr), fill = "#0060af") +
  geom_text(nudge_y = 0.005, size = 8, fontface = "bold", hjust = 0) +
  geom_hline(yintercept = 0) +
  coord_flip() +
  scale_y_continuous(expand = c(0,0), limits = c(0, max(ttr_df$ttr) + 0.05)) +
  labs(title = "Type-Text Ratio (TTR) by Username", 
       subtitle = paste("Most Recent 200 Tweets as of", Sys.Date(), 
                        "(Excluding Retweets)")) +
  theme_main
gg
```

Alas! Nothing dramatic. It appears that anyone who wants to claim Trump has a troglodytian vocabulary in his tweets...well...will have to dig deeper and use either a different methodology or a different comparison set. Actually, by this measure, arguably the _most_ erudite and wonky of the Democratic frontrunners, Elizabeth Warren (@ewarren), has one of the _lowest_ TTRs in her tweets.

At the same time, Andrew Yang, who is arguably one of the _most_ single-issue (UBI) candidates remaining in race, has the highest TTR, when one might expect that he would be _more_ prone to repeating the same words more often in his tweets.

If we dig a little deeper, we can look at the raw word volume used in the tweets. Since we worked with "200 tweets," and a tweet is constrained to 280 characters, there are some natural constraints within the data, but the different candidates actually bump up against those constraints in differents ways.

So, let's look at the _total words_ across the 200 tweets from each user (keeping the users in the same order for comparison purposes):

```{r bar_chart_all_words, fig.height = 6, fig.width = 8, warning = FALSE}
gg <- ggplot(ttr_df, aes(x = username, y = all_words, label = scales::comma(all_words))) +
  geom_bar(stat = "identity", fill = "gray80") +
  # Highlight the user that is of interest
  geom_bar(stat = "identity", mapping = aes(y = highlight_all_words), fill = "#0060af") +
  geom_text(nudge_y = 100, size = 8, fontface = "bold", hjust = 0) +
  geom_hline(yintercept = 0) +
  coord_flip() +
  scale_y_continuous(expand = c(0,0), limits = c(0, max(ttr_df$all_words) + 700)) +
  labs(title = "Total Words by Username", 
       subtitle = paste("Most Recent 200 Tweets as of", Sys.Date(), 
                        "(Excluding Retweets)")) +
  theme_main
gg
```

Now we see that Andrew Yang, who had the highest TTR, also had far and away the _lowest_ number of individual words across our sample of 200 tweets. Trump came in a fairly distant -- but solid -- second. For Trump, this lines up somewhat with my impression of many of his tweets being fairly pithy proclamations (although these are mixed in with multi-tweet rants, too). 

A visual inspection of Yang's Twitter feed turns up that he is actually brevity-biased, with quick notes and thoughts that cross his mind (e.g., ["Why do I feel like I have to see Joker."](https://twitter.com/AndrewYang/status/1180600460098424832) -- brief and punctuation-challenged to boot.)

On the other extreme, Bernie Sanders had far and away the _most_ words across his most recent 200 tweets. A manual of his Twitter feed shows that, unlike Yang, his tweets are fully formed thoughts/commentary about the various issues that are at the core of his platform (often commenting on current events and making the link to his proposed policies...which requires more words than an idle musing).

Is there an obvious relationship between the volume of overall words and the TTR? Scatterplot, here we come:

```{r scatterplot, fig.height = 6, fig.width = 8, warning = FALSE}
gg <- ggplot(ttr_df, aes(x = all_words, y = ttr, label = username)) +
  # A handy ggforce function to get annotations on a static scatterplot
  geom_mark_circle(mapping = aes(fill = username), alpha = 0, color = NA,
                   label.fontsize = 18, expand = 0.01,
                   con.cap = 0, con.colour = "gray50", 
                    show.legend = FALSE) +
  geom_point(stat = "identity", color = "gray60", size = 6) +
  # Highlight the user that is of interest
  geom_point(stat = "identity", mapping = aes(x = highlight_all_words,
                                              y = highlight_ttr), 
             color = "#0060af", size = 8) +
  scale_x_continuous(expand = c(0,0), limits = c(min(ttr_df$all_words) - 700,
                                                 max(ttr_df$all_words) + 500),
                     labels = scales::comma) +
  scale_y_continuous(expand = c(0,0), limits = c(min(ttr_df$ttr) - 0.03, 
                                                 max(ttr_df$ttr) + 0.03),
                     labels = scales::percent_format(accuracy=1)) +
  labs(title = "Total Words vs. TTR by Username", 
       subtitle = paste("Most Recent 200 Tweets as of", Sys.Date(), 
                        "(Excluding Retweets)")) +
  xlab("Total Words") +
  ylab("TTR") +
  theme_main +
  theme(plot.subtitle = element_text(margin = margin(0,0,20,0)),
        panel.border = element_rect(color = "black", fill = NA),
        panel.grid.major = element_line(color = "gray80"),
        axis.title = element_text(size = 24, face = "bold"),
        axis.text.x = element_text(size = 22, margin = margin(10,0,10,0)),
        axis.text.y = element_text(face="plain", margin = margin(0,10,0,10)))
gg
```

Interesting. It's starting to look like there _might_ be an inverse correlation. Andrew Yang may be an outlier on that front (as might be Bernie Sanders). But, let's do a simple check of the correlation coefficient with and without the outliers.

The correlation coefficient between `Total Words` and `TTR` for all users is <strong>`r cor(ttr_df$all_words, ttr_df$ttr)`</strong>.

But, if we check the correlation coefficient between `Total Words` and `TTR` with Andrew Yang _excluded_, it drops to a mere <strong>`r cor(filter(ttr_df, username != "@andrewyang") %>% select(all_words), filter(ttr_df, username != "@andrewyang") %>% select(ttr))`</strong>.

If we remove both Andrew Yang _and_ Bernie Sanders, then the correlation coefficient jumps back up to <strong>`r cor(filter(ttr_df, username != "@andrewyang" & username != "@sensanders") %>% select(all_words), filter(ttr_df, username != "@andrewyang" & username != "@sensanders") %>% select(ttr))`</strong>.

We're not working with very many data points here, so we're veering into dangerous data cherrypicking territory at this point, and really should not do that!

## Conclusions

In general, Donald Trump uses fewer words per tweet than any of the Democratic presidential hopefuls, with Andrew Yang being a notable exception.

When it comes to the number of _unique words_ Trump uses in Tweets as normalized by the total words he uses, he's pretty much middle of the road with those candidates.

And, of course, I had to check where I, personally, netted out, so I ran the analysis on [@tgwilson](https://twitter.com/tgwilson), too. I came out with a **TTR of 35.7%** (which is above even Yang) with **4,597 total words**, which is a dead heat with Trump. If I'd thrown myself into the correlation assessment, it would have simply muddied the waters further.

I like TTR, though. It's a simple idea, simple to calculate (it's even a function in the `koRpus` package, but it's so easy to calculate that it didn't seem warranted to add another package to the mix).
