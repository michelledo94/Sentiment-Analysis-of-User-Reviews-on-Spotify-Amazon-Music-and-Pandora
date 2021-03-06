# Sentiment-Analysis-of-User-Reviews-on-Spotify-Amazon-Music-and-Pandora

Full code and visualization on RPubs: http://rpubs.com/michelledo202/sa

The datasets include almost 40,000 user reviews on Spotify, Amazon Music and Pandora collected from Trustpilot. Sentiment analysis is conducted to get an overview of users’ opinion on the three products and compare sentiment for each app in terms of subscription fee, UX/UI, music catalog and app integration with smart devices.


```{r message=FALSE, warning=FALSE, result="hide"}
library(tidytext)
library(dplyr)
library(ggplot2)
library(stringr)
library(tidyr)
library(igraph)
library(ggraph)
library(tm)
```

Load the data

```{r include=FALSE}
spotify <- read.csv("spotify.csv", stringsAsFactors = FALSE)
pandora <- read.csv("pandora.csv", stringsAsFactors = FALSE)
amazon <- read.csv("amazon.csv", stringsAsFactors = FALSE)
```


```{r}
print("Spotify")
str(spotify)
print("Amazon")
str(amazon)
print("Pandora")
str(pandora)
```


```{r}
colnames(pandora)[1] <- "ID"
```


## Data prep

```{r}
#Tokenize sentences
tidy_spotify <- spotify %>% unnest_tokens(word, Review)
tidy_amazon <- amazon %>% unnest_tokens(word, Review)
tidy_pandora <- pandora %>% unnest_tokens(word, Review)
stop_words <- stopwords("en")
cleaner <- function(x){
  filter(x, !word %in% stop_words)
}
tidy_spotify <- cleaner(tidy_spotify)
tidy_amazon <- cleaner(tidy_amazon)
tidy_pandora <- cleaner(tidy_pandora)
```


## Sentiment analysis

Use 3 popular lexicons nrc, bing and afinn to assign numerical values to each word. The sum of scores given to all words (excluding stop words) in a review is used to classify whether a review is positive, neutral or negative.

```{r}
#Get lexicons
get_sentiments(lexicon="nrc")
get_sentiments(lexicon="bing")
get_sentiments(lexicon="afinn")
```



```{r}
#Join the lexicons
nrc_join <- function(x){
  x %>% inner_join(get_sentiments("nrc"), by = "word")
}
bing_join <- function(x){
  x %>% inner_join(get_sentiments("bing"), by = "word")
}
afinn_join <- function(x){
  x %>% inner_join(get_sentiments("afinn"), by = "word")
}
nrc_sentiment_spotify <- nrc_join(tidy_spotify)
nrc_sentiment_amazon <- nrc_join(tidy_amazon)
nrc_sentiment_pandora <- nrc_join(tidy_pandora)
bing_sentiment_spotify <- bing_join(tidy_spotify)
bing_sentiment_amazon <- bing_join(tidy_amazon)
bing_sentiment_pandora <- bing_join(tidy_pandora)
afinn_sentiment_spotify <- afinn_join(tidy_spotify)
afinn_sentiment_amazon <- afinn_join(tidy_amazon)
afinn_sentiment_pandora <- afinn_join(tidy_pandora)
  
```


Select only words with positive and negative sentiment from the nrc_sentiment dataframes and remove those with anger, anticipation, joy, etc. sentiment by subsetting: 

```{r}
#Aggregate review-level popularity information
#Subsetting function
subset_pn <- function(x){
  x[x$sentiment %in% c("positive", "negative"),]
}
nrc_sentiment_spotify <- subset_pn(nrc_sentiment_spotify)
nrc_sentiment_amazon <- subset_pn(nrc_sentiment_amazon)
nrc_sentiment_pandora <- subset_pn(nrc_sentiment_pandora)
```


```{r}
#Aggregate review-level popularity information
nrc_aggregate <- function(x){
  x %>% select(ID, score) %>% 
    group_by(ID) %>% 
    summarise(nrc_score=sum(score))
}
bing_aggregate <- function(x){
  x %>% select(ID, score) %>% 
    group_by(ID) %>% 
    summarise(bing_score=sum(score))
}
afinn_aggregate <- function(x){
  x %>% select(ID, score) %>% 
    group_by(ID) %>% 
    summarise(afinn_score=sum(score))
}
```

Apply:

```{r warning=FALSE}
#Spotify
nrc_sentiment_spotify <- mutate(nrc_sentiment_spotify, score = ifelse(sentiment == "negative", -1, 1))
nrc_aggregate_spotify <- nrc_aggregate(nrc_sentiment_spotify)
bing_sentiment_spotify <- mutate(bing_sentiment_spotify, score = ifelse(sentiment == "negative", -1, 1))
bing_aggregate_spotify <- bing_aggregate(bing_sentiment_spotify)
afinn_aggregate_spotify <- afinn_aggregate(afinn_sentiment_spotify)
#Amazon
nrc_sentiment_amazon <- mutate(nrc_sentiment_amazon, score = ifelse(sentiment == "negative", -1, 1))
nrc_aggregate_amazon <- nrc_aggregate(nrc_sentiment_amazon)
bing_sentiment_amazon <- mutate(bing_sentiment_amazon, score = ifelse(sentiment == "negative", -1, 1))
bing_aggregate_amazon <- bing_aggregate(bing_sentiment_amazon)
afinn_aggregate_amazon <- afinn_aggregate(afinn_sentiment_amazon)
#Pandora
nrc_sentiment_pandora <- mutate(nrc_sentiment_pandora, score = ifelse(sentiment == "negative", -1, 1))
nrc_aggregate_pandora <- nrc_aggregate(nrc_sentiment_pandora)
bing_sentiment_pandora <- mutate(bing_sentiment_pandora, score = ifelse(sentiment == "negative", -1, 1))
bing_aggregate_pandora <- bing_aggregate(bing_sentiment_pandora)
afinn_aggregate_pandora <- afinn_aggregate(afinn_sentiment_pandora)
```



```{r warning=FALSE}
#Aggregate sentiment info to the original dataset
spotify_sent <- merge(x = spotify, y = nrc_aggregate_spotify, all.x = TRUE, by = "ID")
spotify_sent <- merge(x = spotify_sent, y = bing_aggregate_spotify, all.x = TRUE, by = "ID")
spotify_sent <- merge(x = spotify_sent, y = afinn_aggregate_spotify, all.x = TRUE, by = "ID")
spotify_sent[is.na(spotify_sent)] <- 0
amazon_sent <- merge(x = amazon, y = nrc_aggregate_amazon, all.x = TRUE, by = "ID")
amazon_sent <- merge(x = amazon_sent, y = bing_aggregate_amazon, all.x = TRUE, by = "ID")
amazon_sent <- merge(x = amazon_sent, y = afinn_aggregate_amazon, all.x = TRUE, by = "ID")
amazon_sent[is.na(amazon_sent)] <- 0
pandora_sent <- merge(x = pandora, y = nrc_aggregate_pandora, all.x = TRUE, by = "ID")
pandora_sent <- merge(x = pandora_sent, y = bing_aggregate_pandora, all.x = TRUE, by = "ID")
pandora_sent <- merge(x = pandora_sent, y = afinn_aggregate_pandora, all.x = TRUE, by = "ID")
pandora_sent[is.na(pandora_sent)] <- 0
```

Denote review sentiment:
 
```{r}
#Denote review sentiment
##Spotify
  spotify_sent$afinn_judgement <- ifelse(spotify_sent$afinn_score < 0, "negative",
                                      ifelse(spotify_sent$afinn_score > 0, "positive", "neutral"))
  spotify_sent$bing_judgement <- ifelse(spotify_sent$bing_score < 0, "negative",
                                     ifelse(spotify_sent$bing_score > 0, "positive", "neutral"))
  spotify_sent$nrc_judgement <- ifelse(spotify_sent$nrc_score < 0, "negative",
                                    ifelse(spotify_sent$nrc_score > 0, "positive", "neutral"))
  rowSums(table(spotify_sent$bing_judgement, spotify_sent$bing_score))
```


```{r}
##Amazon
  amazon_sent$afinn_judgement <- ifelse(amazon_sent$afinn_score < 0, "negative",
                                         ifelse(amazon_sent$afinn_score > 0, "positive", "neutral"))
  amazon_sent$bing_judgement <- ifelse(amazon_sent$bing_score < 0, "negative",
                                        ifelse(amazon_sent$bing_score > 0, "positive", "neutral"))
  amazon_sent$nrc_judgement <- ifelse(amazon_sent$nrc_score < 0, "negative",
                                       ifelse(amazon_sent$nrc_score > 0, "positive", "neutral"))
  rowSums(table(amazon_sent$bing_judgement, amazon_sent$bing_score))
```


```{r}
##Pandora
  pandora_sent$afinn_judgement <- ifelse(pandora_sent$afinn_score < 0, "negative",
                                         ifelse(pandora_sent$afinn_score > 0, "positive", "neutral"))
  pandora_sent$bing_judgement <- ifelse(pandora_sent$bing_score < 0, "negative",
                                        ifelse(pandora_sent$bing_score > 0, "positive", "neutral"))
  pandora_sent$nrc_judgement <- ifelse(pandora_sent$nrc_score < 0, "negative",
                                       ifelse(pandora_sent$nrc_score > 0, "positive", "neutral"))
  rowSums(table(pandora_sent$bing_judgement, pandora_sent$bing_score))
```


## Sentiment contribution

Which words contribute the most positive/negative sentiment towards the three apps?

Spotify:

```{r}
bing_wc <- function(x){
  x %>%
    inner_join(get_sentiments("bing"), by = "word") %>%
    count(word, sentiment, sort = TRUE) %>%
    ungroup() %>%
    group_by(sentiment) %>%
    top_n(10) %>%
    ungroup() %>%
    mutate(word = reorder(word, n)) %>%
    ggplot(aes(word, n, fill = sentiment)) +
     geom_col(show.legend = FALSE) +
     facet_wrap(~sentiment, scales = "free_y") +
     labs(y = "Contribution to sentiment",
     x = NULL) +
     coord_flip()
}
```

See which words contribute the most to positive/negative sentiment on Spotify: 

```{r fig.width=10}
bing_wc(tidy_spotify)
```

Amazon:

```{r fig.width=10}
bing_wc(tidy_amazon)
```

And Pandora: 

```{r fig.width=10}
bing_wc(tidy_pandora)
```

## Word cloud

Word cloud of the most common words in each app's review, with word size indicating frequency level:

```{r warning=FALSE}
library(wordcloud)
c_wordcloud <- function(i){
  i %>%
  count(word) %>%
  with(wordcloud(word, n, max.words = 150, rot.per=0.1, scale = c(6,1), colors=brewer.pal(8, "Dark2")))
}
```

Spotify's word cloud:

```{r fig.width=15, message=FALSE, warning=FALSE}
c_wordcloud(tidy_spotify)
```

Amazon's word cloud:

```{r fig.width=15, message=FALSE, warning=FALSE}
c_wordcloud(tidy_amazon)
```

Pandora's word cloud: 

```{r fig.width=15, message=FALSE, warning=FALSE}
c_wordcloud(tidy_pandora)
```

## Check review sentiment on specific topics

#### Price 
How many percent of the reviews on price/subscription fee of each app are positive?

```{r}
#Check price sentiment 
spotifyPrice <- grepl("cost| subscription| price| fee| premium| worth| money| pay| paid| pricing", spotify_sent$Review, ignore.case = TRUE)
sdf <- data.frame(sentiment = spotify_sent$bing_judgement, related_words = spotifyPrice)
sp = sum(sdf$related_words == "TRUE" & sdf$sentiment == "positive")/sum(sdf$related_words == "TRUE")*100
cat("Spotify price =", sp,"%")
amazonPrice <- grepl("cost| subscription| price| fee| premium| worth| money| pay| paid| pricing", amazon_sent$Review, ignore.case = TRUE)
adf <- data.frame(sentiment = amazon_sent$bing_judgement, related_words = amazonPrice)
ap = sum(adf$related_words == "TRUE" & adf$sentiment == "positive")/sum(adf$related_words == "TRUE")*100
cat("Amazon price =", ap,"%")
pandoraPrice <- grepl("cost| subscription| price| fee| premium| worth| money| pay| paid| pricing", pandora_sent$Review, ignore.case = TRUE)
pdf <- data.frame(sentiment = pandora_sent$bing_judgement, related_words = pandoraPrice)
pp = sum(pdf$related_words == "TRUE" & pdf$sentiment == "positive")/sum(pdf$related_words == "TRUE")*100
cat("Pandora price =", pp,"%")
```

#### UX/UI

```{r}
#Check UX/UI sentiment
spotifyUX <- grepl("functionality|UI|UX|experience|user experience|user interface|interface|usability|utility|app|glitch|device", spotify_sent$Review, ignore.case = TRUE)
sux <- data.frame(sentiment = spotify_sent$bing_judgement, related_words = spotifyUX)
su <- sum(sdf$related_words == "TRUE" & sdf$sentiment == "positive")/sum(sdf$related_words == "TRUE")*100
cat("Spotify UX =", su,"%")
amazonUX <- grepl("functionaliyu|UI|UX|experience|user experience|user interface|interface|usability|utility|app|glitch|device", amazon_sent$Review, ignore.case = TRUE)
aux <- data.frame(sentiment = amazon_sent$bing_judgement, related_words = amazonUX)
au <- sum(aux$related_words == "TRUE" & aux$sentiment == "positive")/sum(aux$related_words == "TRUE")*100
cat("Amazon UX =", au,"%")
pandoraUX <- grepl("functionality|UI|UX|experience|user experience|user interface|interface|usability|utility|app|glitch|device", pandora_sent$Review, ignore.case = TRUE)
pux <- data.frame(sentiment = pandora_sent$bing_judgement, related_words = pandoraUX)
pu <- sum(pux$related_words == "TRUE" & pux$sentiment == "positive")/sum(pux$related_words == "TRUE")*100
cat("Pandora UX", pu,"%")
```

#### Music catalog

```{r}
#Check music catalog 
##Spotify
spotifyMusic <- grepl("music selection| selection| music| customized| playlist| recommendation| AI| discovery| variety| song| songs| library| broad| wide| range| catalogue| list| singers", spotify_sent$Review, ignore.case = TRUE)
sm <- data.frame(sentiment = spotify_sent$bing_judgement, related_words = spotifyMusic)
smc <- sum(sm$related_words == "TRUE" & sm$sentiment == "positive")/sum(sm$related_words == "TRUE")*100
cat("Spotify catalog =", smc,"%")
##Amzon
amazonMusic <- grepl("music selection| selection| music| customized| playlist| recommendation| AI| discovery| variety| song| songs| library| broad| wide| range| catalogue| list| singers", amazon_sent$Review, ignore.case = TRUE)
amusic <- data.frame(sentiment = amazon_sent$bing_judgement, related_words = amazonMusic)
amc <- sum(amusic$related_words == "TRUE" & amusic$sentiment == "positive")/sum(amusic$related_words == "TRUE")*100
cat("Amazon catalog =", amc,"%")
##Pandora
pandoraMusic <- pandoraMusic <- grepl("music selection| selection| music| customized| playlist| recommendation| AI| discovery| variety| song| songs| library| broad| wide| range| catalogue| list| singers", pandora_sent$Review, ignore.case = TRUE)
pmusic <- data.frame(sentiment = pandora_sent$bing_judgement, related_words = pandoraMusic)
pmc <- sum(pmusic$related_words == "TRUE" & pmusic$sentiment == "positive")/sum(pmusic$related_words == "TRUE")*100
cat("Pandora catalog =", pmc,"%")
```

#### App's integration with smart devices

```{r}
#Check integration 
spotifyIn <- grepl("device| connection| connect| speaker| smart speaker| alexa| google home| siri| echo| integrate| integration| seamless| across| sync| synchronize| synchronization| bluetooth| strong| car| headphone| earphone| phone| airpod", spotify_sent$Review, ignore.case = TRUE)
si <- data.frame(sentiment = spotify_sent$bing_judgement, related_words = spotifyIn)
s <- sum(si$related_words == "TRUE" & si$sentiment == "positive")/sum(si$related_words == "TRUE")*100
cat("Spotify integration =", s,"%")
amazonIn <- grepl("device| connection| connect| speaker| smart speaker| alexa| google home| siri| echo| integrate| integration| seamless| across| sync| synchronize| synchronization| bluetooth| strong| car| headphone| earphone| phone| airpod", amazon_sent$Review, ignore.case = TRUE)
ai <- data.frame(sentiment = amazon_sent$bing_judgement, related_words = amazonIn)
a <- sum(ai$related_words == "TRUE" & ai$sentiment == "positive")/sum(ai$related_words == "TRUE")*100
cat("Amazon integration =", a,"%")
pandoraIn <- grepl("device| connection| connect| speaker| smart speaker| alexa| google home| siri| echo| integrate| integration| seamless| across| sync| synchronize| synchronization| bluetooth| strong| car| headphone| earphone| phone| airpod", pandora_sent$Review, ignore.case = TRUE)
pi <- data.frame(sentiment = pandora_sent$bing_judgement, related_words = pandoraIn)
p <- sum(pi$related_words == "TRUE" & pi$sentiment == "positive")/sum(pi$related_words == "TRUE")*100
cat("Pandora integration =", p, "%")
```

## N-gram visualization 

N-gram is a useful technique to see which words often go together in the document, so that we can make more sense of the text's content. Here we will start with bigrams (pairs of words). 

```{r}
#Word clustering using bigrams
bi_graph <- subset(spotify_sent, bing_judgement == "positive")
app_bigram <- bi_graph %>% unnest_tokens(bigram, Review, token = "ngrams", n = 2)
##Separate 
bigrams_separated <- app_bigram %>% 
    separate(bigram, c("word1", "word2"), sep = " ")
##Filter 
bigrams_filtered <- bigrams_separated %>%
    filter(!word1 %in% stop_words) %>%
    filter(!word2 %in% stop_words)
##New bigram counts:
bigram_counts <- bigrams_filtered %>%
    count(word1, word2, sort = TRUE)
  
##Unite
bigrams_united <- bigrams_filtered %>%
    unite(bigram, word1, word2, sep = " ")
```



```{r}
##Filter for only relatively common combinations
bigram_graph <- bigram_counts %>%
    filter(n > 10) %>%
    graph_from_data_frame() 
bigram_graph
```


```{r fig.width=15, fig.height=15}
##void-themed graph 
  a <- grid::arrow(type = "closed", length = unit(0.15, "inches"))
  ggraph(bigram_graph, layout = "fr") +
  geom_edge_link(show.legend = FALSE,
                   end_cap = circle(.07, 'inches')) +
  geom_node_point(color = "seagreen3", size = 3) +
  geom_node_text(aes(label = name), vjust = 1, hjust = 1) +
  theme_void() + 
  labs(title="Spotify bigrams")
```

With Amazon:


```{r message=FALSE, warning=FALSE}
#Word clustering using bigrams
library(tidyr)
bi_graph <- subset(amazon_sent, bing_judgement == "positive")
app_bigram <- bi_graph %>% unnest_tokens(bigram, Review, token = "ngrams", n = 2) 
app_bigram %>%
  count(bigram, sort = TRUE)
##Separate 
bigrams_separated <- app_bigram %>% 
    separate(bigram, c("word1", "word2"), sep = " ")
##Filter 
bigrams_filtered <- bigrams_separated %>%
    filter(!word1 %in% stop_words) %>%
    filter(!word2 %in% stop_words)
##New bigram counts:
bigram_counts <- bigrams_filtered %>% 
    count(word1, word2, sort = TRUE)
  
##Unite
bigrams_united <- bigrams_filtered %>%
    unite(bigram, word1, word2, sep = " ")
##Filter the most common bigrams
bigram_graph <- bigram_counts %>%
    filter(n > 20) %>%
    graph_from_data_frame()
```


```{r fig.width=15, fig.height=15}
##void-themed graph 
a <- grid::arrow(type = "closed", length = unit(0.15, "inches"))
ggraph(bigram_graph, layout = "fr") +
  geom_edge_link(show.legend = FALSE,
                   end_cap = circle(.07, 'inches')) +
  geom_node_point(color = "orange", size = 3) +
  geom_node_text(aes(label = name), vjust = 1, hjust = 1) +
  theme_void() + 
  labs(title="Amazon bigrams")
```

And Pandora:

```{r message=FALSE, warning=FALSE}
#Word clustering using bigrams
library(tidyr)
bi_graph <- subset(pandora_sent, bing_judgement == "positive")
app_bigram <- bi_graph %>% unnest_tokens(bigram, Review, token = "ngrams", n = 2) 
app_bigram %>%
  count(bigram, sort = TRUE)
##Separate 
bigrams_separated <- app_bigram %>% 
    separate(bigram, c("word1", "word2"), sep = " ")
##Filter 
bigrams_filtered <- bigrams_separated %>%
    filter(!word1 %in% stop_words) %>%
    filter(!word2 %in% stop_words)
##New bigram counts:
bigram_counts <- bigrams_filtered %>% 
    count(word1, word2, sort = TRUE)
  
##Unite
bigrams_united <- bigrams_filtered %>%
    unite(bigram, word1, word2, sep = " ")
##Filter the most common bigrams
bigram_graph <- bigram_counts %>%
    filter(n > 25) %>%
    graph_from_data_frame()
```


```{r fig.width=15, fig.height=15}
##void-themed graph 
a <- grid::arrow(type = "closed", length = unit(0.15, "inches"))
ggraph(bigram_graph, layout = "fr") +
  geom_edge_link(show.legend = FALSE,
                   end_cap = circle(.07, 'inches')) +
  geom_node_point(color = "skyblue2", size = 3) +
  geom_node_text(aes(label = name), vjust = 1, hjust = 1) +
  theme_void() + 
  labs(title="Pandora bigrams")
```

The bigram plots above could help us see the links between the most common words, better understand review contexts and explore notable topics for further analysis. 


