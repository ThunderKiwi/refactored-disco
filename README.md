# refactored-disco


First lets install and load the required packages
 * tidyverse for data wrangling
 * haven, labelled, likert, and sjmisc for labelled data transformation

```{r}

install.packages(c("tidyverse", "likert", "haven", "labelled", "sjmisc"))

library(tidyverse)
library(likert)
library(haven)
library(labelled)
library(sjmisc)

```

# naming conventions
This approach works best when use with naming conventions that group like variables. In survey research this could be a validated scale or group of questions with similar response options. Your personalized naming conventions will allow speedy data cleaning over time and your system will become personalized. Here are a few examples:
 * _r (recoded) 
 * _daX (X point disagree - agree scale)
 * _catX (X categories)s


# Create test data

First, we will specify the number of cases for our test datset.  In this case, we've defined the object `n_sample` the number 100 cases, which we use in the creation of our data set.  We create the data set using `tibble()` to work with tidyverse package. First, we use `seq()` to create a unique, sequential id variable, the length of the sample `n_sample`. The next variables are created using `sample.int()` that samples integers that will represent questionnaire responses to a Likert scale.  We are not dealing with missing data in this tutorial. Remember to specify `replace = TRUE` in order to sample with replacement. Our grouping variable, prepost, is created as a character variable using `sample()` to demonstrate how to create text variables and to show the differences in cleaning various types of data.  

```{r}

n_sample <- 100

dat <- tibble(
  ## set a unique, sequential id using seq(), the length of n_sample
  id = seq(1, n_sample, 1),
  ## use sample.int(5) to create test variables the length of n_sample that consist of whole numbers from 1 to 5
  ## must specify replace = TRUE to ensure that data is sampled with replacement
  q1 = sample.int(5, n_sample, replace = TRUE),
  q2 = sample.int(5, n_sample, replace = TRUE),
  q3 = sample.int(5, n_sample, replace = TRUE),
  q4 = sample.int(5, n_sample, replace = TRUE),
  q5 = sample.int(5, n_sample, replace = TRUE),
  ## create prepost variable sampling from character options c("Pre", "Post") to demonstrate cleaning approach
  prepost = sample(c("Pre","Post"), n_sample, replace = TRUE)
)

```

# Define your Likert scales

Now we define our Likert scales for transformation. Below are some examples of common Likert scales you can use in your transformations along with a template for creating your own. By saving these as objectswe only need to define them once.

```{r}

# specify the labels for your Likert scales
da5_labels <- c("Strongly disagree" = 1,
                "Somewhat disagree" = 2,
                "Neither disagree nor agree" = 3,
                "Somewhat agree" = 4,
                "Strongly agree" = 5)



# labeltemplate <- c("" = 1,
#                    "" = 2,
#                    "" = 3,
#                    "" = 4,
#                    "" = 5,
#                    "Don't know" = 7,
#                    "Refused" = 9)

```

# Applying Labels

Now that we have our test data and have defined our Likert scales, lets apply those labels and save the transformed data into the new data set `dat_labelled`.  We use `mutate()` to transform our variables, in this case we're overwriting the existing variables, so be sure to double check your transformations. By using `across(variables, ~labelled())` within `mutate()` we can apply a function, in this case `labelled()`, to multiple variables. `contains()` makes selecting by naming conventions easy, here we're selecting the variables whose names contains "q".  Finally, `~labelled(.x,)` applies the labels we defined above (the ~ and .x are necessary programming nonsense - just make sure they're there for now).

```{r}

dat_labelled <- dat %>%
  mutate(across(contains("q"), ~labelled(.x, labels = da5_labels)))
  
# always double check any transformation
apply(dat %>% select(contains("q")), 2, table)
apply(dat_labelled %>% select(contains("q")), 2, table)
lapply(dat %>% select(contains("q")), str)
lapply(dat_labelled %>% select(contains("q")), str)

```

# Transforming our character variable: 

Here we use `factor()` within `mutate()` to overwrite our prepost with a factorized version of itself. We must remember to explicitly define the levels within `factor()` since the default numeric order is alphabetically which poses a problem for Pres and Posts. Now, rather than using pre-defined labels, we can define the labels directly in `labelled()` through the `labels = c()` option.  Again, be sure to check any transformation.

```{r}

dat_labelled.1 <- dat_labelled %>% 
  mutate(prepost = factor(prepost, levels = c("Pre", "Post"))) %>%
  ## creating labelled variables, an example of defining labels directly rather than using predefined labels
  mutate(across("prepost", ~labelled(.x, labels = c(Pre = 1, Post = 2)))) 

# double check your transformation
table(dat_labelled$prepost)
table(dat_labelled.1$prepost)
str(dat_labelled$prepost)
str(dat_labelled.1$prepost)

```

# Collapsing response options / recoding variables:

Let's create new variables with the agree and disagree categories collapsed and provide these new variables with an appended name. We will again use mutate(across(contains())) to apply `rec()` across the desired variables. Specifying the recode within `rec()` use the pattern `rec = "old=new;old,old=new"`, chaining grouped recodes together with `;`.  The `.names` option within `across()` appends a naming convention to the new transformed variables. In this case we are adding "_rda3" to indicate that these variables have been recoded to a three category disagree agree scale. `{col}` in this case keeps the original column name at the front.

```{r}

dat_labelled_r <- dat_labelled.1 %>%
  mutate(across(contains("q"), ~ sjmisc::rec(.x, rec = "1,2=1;3=2;4,5=3"), .names = "{col}_rda3")) %>%
  ## remember you may have to adjust across() to get the correct variables [i.e. starts_with() and ends_with()]
  mutate(across(c(starts_with("q"), ends_with("rda3")), ~ labelled(.x, labels = c("Disagree" = 1,
                                                                                   "Neutral" = 2, 
                                                                                   "Agree" = 3))))
                                                                                   
```

# Writing the data

We can  use haven's `write_sav()` to write our labelled data as an spss data file `.sav`, or use `write_rds()` to write the data as an r data file `.rds`.

```{r}

haven::write_sav(dat_labelled_r, "path/to/data/filename.sav")
write_rds(dat_labelled, "path/to/data/filename.rds")

```