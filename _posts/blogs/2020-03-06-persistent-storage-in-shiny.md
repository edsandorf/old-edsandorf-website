---
layout: post
title: "Persistent storage on Shinyapps.io: Storing survey responses responsibly"
header-image: /assets/img/blogs/2020-03-06-persistent-storage-01.jpg
author: Erlend
image: /assets/img/blogs/2020-03-06-persistent-storage-01.jpg
category: blog
excerpt: A post about how I set up persistent storage and transfer of data from a Shiny survey to MariaDB using SSL certificates.
---

In this post, I will outline how I set up persistent storage and transfer of data from a Shiny survey to a MariaDB using SSL certificates. How I set up the survey and how I handled URL redirects in and out of the survey are topics for future posts. Coming up with a satisfactory solution to this problem took a lot of trial and error (and Googleing...). It turned out that all of the challenges related to persistent storage, sending data to a database, preventing SQL injection attacks and securing the data traffic with SSL certificates had existing solutions. The problem was that none of them seemed to be in the same place. This post is that same place. At least for me and this particular solution. I do hope that this will benefit more people than just future me looking at his own "notes" on what he did way back when.

Throughout this post, I will make use of the following packages: `shiny`, `RMariaDB`, `config`, `pool`, `jsonlite` and `DBI`. The solution provided here is not a stand-alone solution and cannot be run as a functioning minimal working example, but it should contain enough detail to be easily applied to any Shinyapp trying to do the same. Towards the end of the post, I show an outline of what your `app.R` may look like.

##  Securing the connection
I used the `RMariaDB` package to connect to my database since it has built in functionality to use SSL certificates, which allows data to be transferred securely between my survey hosted on Shinyapps.io and the database. The IT department at the University were really efficient and helpful, and provided me with the necessary usernames, passwords and SSL certificates to secure the information and access the database. To keep my security details separate from the rest of the code I used `config` package in R. See for example [Section 3.8.3](https://docs.rstudio.com/shinyapps.io/applications.html#configuring-your-application) in the Shinyapps.io User Manual or the [package documentation](https://cran.r-project.org/web/packages/config/vignettes/introduction.html) for more advanced use cases.

```yaml
default:
    dataconnection:
        dbname: 'MY_DB_NAME'
        host: 'MY_HOSTNAME'
        username: 'MY_USERNAME'
        password: 'MY_PASSWORD'
        ssl.key: 'client-key.pem'
        ssl.cert: 'client-cert.pem'
        ssl.ca: 'ca-cert.pem'
```

Here I have used placeholders for `dbname`, `host`, `username` and `password`. I placed the `config.yml` file in the root of my app folder along with my SSL certificates. To avoid accidentally publishing my passwords or security certificates, I added `config.yml`, `client-key.pem`, `client-cert.pem` and `ca-cert.pem` to my `.gitignore` file.

To send data through the University firewall, I contacted the IT department to let them know which servers were trying to send information to the database. A detailed approach, including the IP addresses can be found in [Section 3.8.4](https://docs.rstudio.com/shinyapps.io/applications.html#configuring-your-application) in the Shinyapps.io User Manual.

To make the content of my `config.yml` file available, at the top of my `app.R` file (I could also use my `global.R` file), I included the following line of code to assign all the information in my `config.yml` file to the list `db_config`. Since your `config.yml` file is not tracked by version control, the information contained in `db_config` is not visible to people reading the code, which helps protect your sensitive information.

```r
db_config <- config::get("dataconnection")
```

## Set up a pool to handle database connections

Using Shiny to run a survey implies that you will have multiple simultaneous connections to your database. To handle this in an efficient manner, I used the `pool` package. All you have to do is set up your pool, and connections to the database is handled in the background. No need to worry. For a quick introduction to `pool` and its benefits, please check out this RStudio [article](https://shiny.rstudio.com/articles/pool-basics.html).

```r
db_pool <- dbPool(
  drv = MariaDB(),
  dbname = db_config$dbname,
  host = db_config$host,
  username = db_config$username,
  password = db_config$password,
  ssl.key = db_config$ssl.key,
  ssl.cert = db_config$ssl.cert,
  ssl.ca = db_config$ssl.ca
)
```

## Writing a function to store responses

The function for saving information to the database, `save_db()` is a function of the database pool object, `db_pool`; the named vector of responses `x`, where the names correspond to the columns in the database; the name of the database, `db_name`; the configuration object, `db_config`; and the boolean `replace_val`.

The call to the databse is sent as a literal string, which does expose you to SQL injection attacks. Worst case, you can loose all your data. To eliminate this risk, I used the `sqlInterpolate()` function from the `DBI` package. A great guide on how to use this function can be found [here](https://shiny.rstudio.com/articles/sql-injections.html). In my case, all of the vecor elements in `x` are JSON strings (created using the `jsonlite` package), which necessitated a slightly different approach to interpolation. I wrote more about how (and why) this approach works in [this Stackoverflow post](https://stackoverflow.com/questions/59321618/sanitizing-multiple-json-strings-sent-to-mariadb-using-rmariadb-and-pool-in-r/59326719#59326719). The next part of the function creates one of two database queries depending on whether we are creating a new entry (`replace_val == FALSE`) or updating an existing one (`replace_val == TRUE`). Finally, we send the query to the database using our database pool object. Using the pool package, I do not have to worry about closing the connection to the database.

```r
save_db <- function (db_pool, x, db_name, db_config, replace_val) {
  # Interpolate the elements of x
  x <- do.call(c, lapply(x, function(y) {
    sql <- "?value"
    sqlInterpolate(db_pool, sql, value = y)
  }))

  # Construct the DB query to be sent to the database
  if (!replace_val) {
    query <- sprintf(
      "INSERT INTO %s (%s) VALUES (%s)",
      db_name,
      paste(names(x), collapse = ", "),
      paste(x, collapse = ", ")
    )

  } else {
    query <- sprintf(
      "UPDATE %s SET %s WHERE %s;",
      db_name,
      paste(paste0(names(x)[-1], " = ", x[-1]), collapse = ", "),
      paste0(names(x)[1], " = ", x[1])
    )
  }

  # Submit the insert query to the database via the opened connection
  dbExecute(db_pool, query)
}
```

##  Storing the result
In my first attempt at sending the data to the database, I put my `save_db()` inside `session$onSessionEnded()` to send all the data for a respondent to the database when a respondent left the application, either because they were redirected back to the survey company when completing the survey or disconnected for whatever reason. Problematically, `onSessionEnded` only triggered once. If a respondent was briefly disconnected, but reconnected before the session timed out, none of the data collected after the reconnection would be sent to the database because the `onSessionEnded` did not trigger again. It is possible that I simply was unable to find a solution to this particular problem. However, I came up with a workaround. To solve the problem, I set up the function to send updated information to the database every time a respondent clicked the "next" button. Specifically, I placed the `save_db()` inside the observer for the "next" button.

##  An outline of the solution
Below I have created a very simple static one-page survey with a single text input box. When you click the "next" button the value in the text input field will be submitted to the database. In a future post, I will show how we can extend this simple example to a dynamic survey with multiple pages.

```r
# Define globals ----
pkgs <- c("shiny", "RMariaDB", "config", "pool", "jsonlite", "DBI")
invisible(lapply(pkgs, require, character.only = TRUE))

# Get configuration
db_config <- config::get("dataconnection")

# Set up the database pool
db_pool <- dbPool(
  drv = MariaDB(),
  dbname = db_config$dbname,
  host = db_config$host,
  username = db_config$username,
  password = db_config$password,
  ssl.key = db_config$ssl.key,
  ssl.cert = db_config$ssl.cert,
  ssl.ca = db_config$ssl.ca
)

# Define the UI ----
ui <- fluidPage(
  # Open form question box
  textInput(inputId = "answer", label = "Please write your answer", value = "")

  # Define your next alternative button
  actionButton(
    inputId = "next",
    label = "Next question"
  )
)

# Set up the server side ----
server <- function(input, output, session) {

  # Observe whether the "next" button has been clicked
  observeEvent(
    {
      input[["next"]]
    },
    {
      # Capture the response to the question in 'survey_output'
      survey_output <- input$answer

      # Make sure that the names of survey_output matches the columns in your
      # database
      names(survey_output) <- "question_1"

      # Send the data to the database. Note replace_val = FALSE. A single page survey
      save_db(db_pool, survey_output, "MY_DB", db_config, FALSE)
    }
  )
}

# Combine into an app ----
shinyApp(ui = ui, server = server)

```

##  Wrapping up
While this post was not a minimal working example, I do hope that it adds value. The goal was to outline some of the issues I encountered when trying to securely store survey data from a Shiny application. I will continue to write short blog posts outlining some of the technical challenges I faced implementing a survey in Shiny, and eventually the full code for the survey will be publically available.
