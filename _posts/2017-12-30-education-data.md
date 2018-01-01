---
layout: post
title: "R in the World of Education"
tags: data science education
---

I recently gave a [r-ladies presentation](({{ site.baseurl }}/Presentations/rladies-nyc/data-education.html)) about my work cleaning and working with really messy education data. This blog post is an attempt at summarizing the main points of the [talk]({{ site.baseurl }}/Presentations/rladies-nyc/data-education.html).


<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Excited for <a href="https://twitter.com/astroeringrand?ref_src=twsrc%5Etfw">@astroeringrand</a>’s <a href="https://twitter.com/RLadiesNYC?ref_src=twsrc%5Etfw">@RLadiesNYC</a> talk on <a href="https://twitter.com/hashtag/rstats?src=hash&amp;ref_src=twsrc%5Etfw">#rstats</a> in education! <a href="https://t.co/wX3PT7F2KI">pic.twitter.com/wX3PT7F2KI</a></p>&mdash; Emily Robinson (@robinson_es) <a href="https://twitter.com/robinson_es/status/940730589077999617?ref_src=twsrc%5Etfw">December 12, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
_Look how cool I am. Go me! The slides are [here]({{ site.baseurl }}/Presentations/rladies-nyc/data-education.html) in case you missed them._


# Uncommon Schools
I've been an Associate Director of Data Analytics at Uncommon Schools for almost a year and a half now. Part of my job at Uncommon has been working with and teaching R to my fellow data analysts.  As such, I've developed a sense of what works best for the type of messy data we're constantly analyzing.

As a bit of background, Uncommon Schools is a Charter Management Organization (CMO), or network of 52 public [charter schools](http://www.uncommonschools.org/our-approach/faq-what-is-charter-school) across Massachusetts, New Jersey, and New York. The oldest school (North Star Academy in Newark) was established in 1997 and the CMO was formed in 2005. For a more in depth history of Uncommon Schools, check out our [website](http://www.uncommonschools.org/our-approach/our-history).

![uncommon-schools](http://www.uncommonschools.org/sites/default/files/datagraphics_063017.jpg)


## What kind of data do we work with? 
The Data Analytics team focuses at the overall picture, so while we don't work on reporting data directly back to the state (our regional teams have excellent people working on this) we do get to work with most of the data the organization collects. 

The data we work with generally fits into one of these buckets...

- **Assessments**: Interim assessments are taken as practice tests through out the school year.
- **Exams**: Common Core aligned state exams, SAT, PSAT, APs, ...etc
- **Classroom**: Assignment grades, attendance, suspensions, ...etc
- **Teacher**: student - course - teacher linkage information
- **Staff Data**: HR and Recruitment

Unfortunately, given the amount of data we have and the number of sources it may be coming from, we have a ton of data challenges to overcome in every analysis.

Of course, every piece of data has its own challenges and "messy nature,"  but there are patterns.

- Missing/Incomplete data
- Different data sources without matching IDs (i.e HR to Teacher to Student)
- Movement between schools and courses of students and teachers
- Alignment of data and data processes across all schools and regions
- Student IDs that change
- Human data reporting error
- Historical data quality

Same of these challenges are easy to fix and some are harder. For example, a messy excel sheet can be cleaned (by hand or by code). We've developed (or are developing in some cases) systems to work with most of these types of challenges.

- Messy excel sheets (historical or human entered)
- Column names that don't apply anymore
- Lack of historical documentation
- Finding duplicates tests
- Students that take half of one test and the other half of another
- Vanishing leading zeros
- Tracking of student IDs that change
- Lack of common definitions (i.e "cohort")
- How to refer to school years or school abbreviations
- Data audits


# The `janitor` Package

I really like this explanation on the package by package author, Sam Firke. _[Janitor](https://github.com/sfirke/janitor) was built with beginning-to-intermediate R users in mind and is optimized for user-friendliness. Advanced users can already do everything covered here, but they can do it faster with janitor and save their thinking for more fun tasks._

Meaning, if you're experienced with the Tidyverse in general, you should be able to do everything inside `janitor` on your own. However, we don't always have the time to always clean up data without some help.

<div align="center">
<img src = "http://media3.giphy.com/media/3oKIPCSX4UHmuS41TG/giphy-downsized.gif" width="100px"/>
</div>

I like using `janitor` over writing my own code because, (1) functions are well tested, (2) I can turn multiple lines of code into one or two, and (3) the `janitor` functions are written to be pipe-able to work in the tidyverse space. It's a pretty cool bonus that Sam works in the education space, so the functions were created to handle the education data problems I constantly face.


## An example, using `janitor` to clean a messy excel file.

I'm often tasked with cleaning roster files, which contain entry and exit data for students. These files can be very messy due to students who moved between schools or were not exited properly from the system, causing duplicates. 

To clean this data, first I read it in with `read_excel` and use `janitor's` `clean_names` to convert all the column names to something I can use. `remove_empty_cols` and `remove_empty_rows` remove entire columns or rows that are NA as excel sometimes can't tell where there is data and where there isn't. 

I choose to use `col_type = "text"` in my `read_excel` statement, because I sometimes have to deal with leading zeros, NAs that are not written as NAs, or other text fields in my numerical columns. Reading in as text and converting later allows me to find and correct problems before they become NAs. I use `mutate_at` to convert these columns back to numbers after examining that everything looks good.

```r
students <- readxl::read_excel(filepath, sheet="Sheet1", col_types = "text") %>%
  janitor::clean_names() %>%
  janitor::remove_empty_cols() %>%
  janitor::remove_empty_rows() %>%
  dplyr::mutate_at(vars(entrydate, exitdate, student_id, yearsinuncommon), as.numeric) %>%
  dplyr::mutate_at(vars(entrydate, exitdate), excel_numeric_to_date) 
```

The next step in data cleaning is to look for duplicates. Luckily, `janitor` has a super helpful `get_dupes()` function which does just that!

```r
students %>% 
  get_dupes(student_id)
```

```
# A tibble: 2 x 6
  student_id dupe_count grade yearsinuncommon  entrydate   exitdate
       <dbl>      <int> <dbl>           <dbl>     <date>     <date>
1    2342675          2    10               1 2017-11-11 2017-12-11
2    2342675          2    11               1 2017-11-11 2017-12-11
```

In this example data, I have one student with duplicate information. They're in two different grades, ugh! Now, I have to choose a method to correct this student in my data. 

### There are three main ways I use to correct dupes.


#### 1. Correct the dupes individually with `if_else` or `case_when`.
If there are only a few duplicates, or they're all in one grade or class room, a quick set of _if_ statements will do the trick to make the rows perfectly duplicated. From there, use `distinct` to get only distinct rows and remove the duplicates.

```r
mutate(students, grade = if_else(student_id == 2342675, 10, grade))
```

####  2. Summarize by taking minimum date / grade to choose one row to keep. 
This is helpful if you just need one of the rows, and don't really care which row is the one you keep. For example, our exit and enter date information is not usually great, so I'm okay with the 'just pick one' version as long as the student's grade and teacher information is the same in both rows.


```r
group_by(students, student_id) %>% summarize(grade = min(grade))
```

#### 3. Output the duplicates and manually choose which version to keep. 
This involves the most manual work, so I usually grab help when I need to do this, yay teammates!

```r
dupes_correct <- read_csv("dupes_correct.csv")
left_join(students, dupes_correct) %>%
  replace_na(list(keep = 0)) %>%
  assert(not_na, keep) %>%
  filter(keep = 0)
```

### Managing Data Changes

As data is updated, there might be more duplicates to worry about. A great way to check for duplicate updates is to use the 1:2 punch of **janitor's** `get_dupes` and **assertr's** `verify()`. This allows you to put checks in place in case the data changes.

```
check <- students %>% 
  get_dupes(student_id) %>% 
  verify(nrow(.) == 0)
```

If new duplicates occur the code will HALT at this step alerting that something is wrong.

#  An Example Project: State Test Analysis

The largest impact project the data analytics team is in charge of all year is our annual state test analysis. We gather and clean all the raw results data for each of our schools, clean it, and combine the information into one cohesive story.

The old process for this used a lot of excel workbooks, manual edits, and a very confusing naming system to port the data into tableau dashboards. With the many steps, and points or error, the process took a really long time. A big goal of ours was to go from raw data to published tableau dashboard in a *few hours* without any big hiccups.

This year we completed an overhaul of the process with R scripts, that clean and QC the data, add variables we need for analysis, combine with historical state test results, and output tableau ready inputs. The entire state test analyses from raw data to dashboard can now be completed with the press of a few buttons.

<img src = "{{ site.baseurl }}/Presentations/rladies-nyc/process.png" height = 100px>

Using `PURRR` code to read and combined multiple files into one data frame has been the saving grace of this analysis. We have each grade/subject/school combination in a separate file, and with just a few lines of code R can bring them together for further cleaning.

```r
files <- list.files("../Input/", pattern = ".xlsx", full.names =  TRUE)
nys <- map_dfr(files, prep_nys_files)
```

If you're curious to other parts of the code that really changed this process (for example `assertr` or `tidyr`) please feel free to ask! 


# Wrap Up
These changes didn't come easy for my team. I work with a group of extremely smart people, but before I started with Uncommon most people on my team didn't code at all let alone use R. It's been part of my job to teach everyone to write in R, and make sure we're all using best practices. One day I will write a longer blog post about some of the learnings I've had throughout this process, so stay tuned for that! In the mean time, I offer you some closing remarks on what I've found to work best.

- Choose the packages to teach that are needed every day. (For me, this was `janitor` and `dplyr`)
- Have someone that is active in R community, so that you can be on the cutting edge of best practices and new packages.
- The more practice someone has, the faster they'll learn. Pair professional development sessions with coding projects. 

Introducing my team to the `tidyverse` and `janitor` have been a really big help for getting my team members on board and excited about learning and using R. A quick demonstration of `group_by()` and `get_dupes()` was really all I needed to motivate big changes in our analysis process.
