---
title: "Creating An Efficient Data"
output: html_document
---

```{r}
library(tidyverse)
library(lubridate)

sales <- read_csv("sales2019.csv")
```

# Data Exploration

```{r}
# How big is the dataset?
dim(sales)
```

```{r}
# What are the column names?
colnames(sales)
```

The `date` column shows the date that the order of books was made. This will help us distinguish between orders that were made before and after the new program was implemented. `quantity` describes how many books were made, and `user_submitted_review` looks like it's a hand typed review of the books themselves. `customer_type` indicates whether or not the customer was an individual or a business. It seems that the company has started selling in bulk to other business too.

```{r}
# What are the types of all the columns?
for (col in colnames(sales)) {
  paste0(col, " : ", typeof(sales[[col]])) %>% print
}
```


```{r}
# Is there missing data anywhere?
for (col in colnames(sales)) {
  paste0(col, 
         ", number of missing data rows: ", 
         is.na(sales[[col]]) %>% sum) %>% print
}
```

The `user_submitted_review` column has some missing data in it. We'll have to handle this later in the data cleaning, but at least we know about it ahead of time. The `total_purchased` column also has missing data, which we'll handle with imputation.

# Handling Missing Data

```{r}
# Remove the rows with no user_submitted_review
complete_sales <- sales %>% 
  filter(
    !is.na(user_submitted_review)
  )

# Calculate the mean of the total_purchased column, without the missing values
purchase_mean <- complete_sales %>% 
  filter(!is.na(total_purchased)) %>% 
  pull(total_purchased) %>% 
  mean

# Assign this mean to all of the rows where total_purchased was NA
complete_sales <- complete_sales %>% 
  mutate(
    imputed_purchases = if_else(is.na(total_purchased), 
                                purchase_mean,
                                total_purchased)
  )
```

# Processing Review Data

```{r}
complete_sales %>% pull(user_submitted_review) %>% unique
```

The reviews range from outright hate ("Hated it") to positive ("Awesome!"). We'll create a function that uses a `case_when()` function to produce the output. `case_when()` functions can be incredibly bulky in cases where there's many options, but housing it in a function to `map` can make our code cleaner.

```{r}
is_positive <- function(review) {
  review_positive = case_when(
  str_detect(review, "Awesome") ~ TRUE,
  str_detect(review, "OK") ~ TRUE,
  str_detect(review, "Never") ~ TRUE,
  str_detect(review, "a lot") ~ TRUE,
  TRUE ~ FALSE # The review did not contain any of the above phrases
  )
}

complete_sales <- complete_sales %>% 
  mutate(
    is_positive = unlist(map(user_submitted_review, is_positive))
  )
```

# Comparing Book Sales Between Pre- and Post-Program Sales

```{r}
complete_sales <- complete_sales %>% 
  mutate(
    date_status = if_else(mdy(date) < ymd("2019/07/01"), "Pre", "Post")
  )

complete_sales %>% 
  group_by(date_status) %>% 
  summarize(
    books_purchased = sum(imputed_purchases)
  )
```

It doesn't seem that the program has increased sales. Maybe there were certain books that increased in sales?

```{r}
complete_sales %>% 
  group_by(date_status, title) %>% 
  summarize(
    books_purchased = sum(imputed_purchases)
  ) %>% 
  arrange(title, date_status)
```

It turns out that certain books actually got more popular after the program started! R For Dummies and Secrets of R For Advanced Students got more popular.

# Comparing Book Sales Within Customer Type

```{r}
complete_sales %>% 
  group_by(date_status, customer_type) %>% 
  summarize(
    books_purchased = sum(imputed_purchases)
  ) %>% 
  arrange(customer_type, date_status)
```

Baserd on the table, it looks like businesses started purchasing more books after the program! There was actually a drop in individual sales.

# Comparing Review Sentiment Between Pre- and Post-Program Sales

```{r}
complete_sales %>% 
  group_by(date_status) %>% 
  summarize(
    num_positive_reviews = sum(is_positive)
  )
```

There's slightly more reviews before the program, but this difference seems negigible.
