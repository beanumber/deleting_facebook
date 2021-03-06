Parsing friend’s birthdays
================

``` r
knitr::opts_chunk$set(echo = TRUE, eval = FALSE)
library(calendar)
library(here)
library(dplyr)
library(readr)
```

There is a package called `calendar`, which is supposed to parse .ics
files. However, it doesn’t work right out of the box on a
facebook-generated .ics file.

``` r
birthdays_calendar_pkg <- ic_read(here("../Dropbox/birthdays.ics"))
```

All of the birthdays get set to the same date (October 3), which was the
first birthday in my file.

So, now we need to debug to figure out what’s going wrong. I am not a
master debugger, so what I do is look at the source code for a function,
and then run it step-by-step on my data to see where the error is.

## ic\_read

The top-level function is `ic_read`.

``` r
## taken from the source code of ic_read
birthdays <- readLines("birthdays.ics")
y <- c()
    for (line in birthdays) {
        if (!grepl(pattern = "[A-Z]-?[A-Z]:|[A-Z];", x = line)) {
            y[length(y)] <- paste0(y[length(y)], "\n", line)
        }
        else {
            y[length(y) + 1] <- line
        }
    } ## to this point, the data is still fine
ical(y) ## this is where everything becomes October 3
```

## ical

``` r
## taken from the source code of ical
x <- y
is_df <- is.data.frame(x)
    if (methods::is(x, "character")) {
        ical_df <- ic_dataframe(x) ## here's where the next problem happens
        ical_tibble <- tibble::as_tibble(ical_df)
        if (is.null(ic_attributes)) {
            attr(ical_tibble, "ical") <- ic_attributes_vec(x)
        }
        else {
            attr(ical_tibble, "ical") <- ic_attributes
        }
    }
    else {
        if (!is_df) 
            stop("x must be a data frame or charcter strings")
        n <- names(x)
        is_core = calendar::properties_core %in% n
        if (!all(is_core)) {
            stop(paste0("x must contain column names: ", paste0(calendar::properties_core, 
                collapse = ", ")))
        }
        ical_tibble <- tibble::as_tibble(x)
        if (is.null(ic_attributes)) {
            attr(ical_tibble, "ical") <- ic_attributes_vec()
        }
        else {
            attr(ical_tibble, "ical") <- ic_attributes
        }
    }
    class(ical_tibble) <- c("ical", class(ical_tibble))
    ical_tibble
```

## ic\_dataframe

``` r
## taken from the source code of ic_dataframe
x <- y
if (methods::is(object = x, class2 = "data.frame")) {
        return(x)
    }
    stopifnot(methods::is(object = x, class2 = "character") | 
        methods::is(object = x, class2 = "list"))
    if (methods::is(object = x, class2 = "character")) {
        x_list <- ic_list(x) ## seems fine here
    }
    else if (methods::is(object = x, class2 = "list")) {
        x_list <- x
    }
    x_list_named <- lapply(x_list, function(x) {
        ic_vector(x) ## fine here
    })
    x_df <- ic_bind_list(x_list_named) ## good here
    date_cols <- grepl(pattern = "VALUE=DATE", x = names(x_df)) ## this is the right columns
    if (any(date_cols)) {
        tmp <- lapply(x_df[, date_cols], ic_date) ## here's the issue. I renamed the variable to tmp to debug
        x_df <- x_df %>%
          mutate(`DTSTART;VALUE=DATE` = ymd(`DTSTART;VALUE=DATE`)) # this works on my particular data
    }
    datetime_cols <- names(x_df) %in% c("DTSTART", "DTEND")
    if (any(datetime_cols)) {
        x_df[datetime_cols] <- lapply(x_df[, datetime_cols], 
            ic_datetime)
    }
    x_df
```

## Working code

Since I can’t see what the issue is (so terrible at `tapply`) I didn’t
think I could fix the function itself. Instead, I just copied the
working code into a messy long chunk. Here’s what works for me:

``` r
birthdays <- readLines("birthdays.ics")
y <- c()
    for (line in birthdays) {
        if (!grepl(pattern = "[A-Z]-?[A-Z]:|[A-Z];", x = line)) {
            y[length(y)] <- paste0(y[length(y)], "\n", line)
        }
        else {
            y[length(y) + 1] <- line
        }
    } ## to this point, the data is still fine
x_list <- ic_list(y)
x_list_named <- lapply(x_list, function(x) {
        ic_vector(x) ## fine here
})
x_df <- ic_bind_list(x_list_named) 

x_df <- x_df %>%
          mutate(`DTSTART;VALUE=DATE` = ymd(`DTSTART;VALUE=DATE`)) 
ical_tibble <- tibble::as_tibble(x_df)
attr(ical_tibble, "ical") <- ic_attributes_vec()
class(ical_tibble) <- c("ical", class(ical_tibble))
```

## Parsing close friends

``` r
birthday_names <- ical_tibble %>%
  select(SUMMARY)
birthday_names <- birthday_names %>%
  separate(SUMMARY, into = "name", sep="'s Birthday", remove=FALSE)
```

``` r
facebookfriends_augmented <- read_csv("facebookfriends_augmented.csv") %>%
  filter(Rank %in% c(1,2))
which(birthday_names$name %in% facebookfriends_augmented$name)
birthdays <- ical_tibble[which(birthday_names$name %in% facebookfriends_augmented$name),]
```

``` r
ic_write(birthdays, "close_friends_birthdays.ics")
```

I was able to read the .ics file into my Calendar app and it all seems
to have worked\! Spot-checked birthdays are correct.

Of course, not every close friend had their birthday listed on facebook.
So, there is also the question of friends I should learn the birthday
of:

``` r
find_birthdays <- facebookfriends_augmented[which(!facebookfriends_augmented$name %in% birthday_names$name),]
```

``` r
write_csv(find_birthdays, "learn_birthdays.csv")
```
