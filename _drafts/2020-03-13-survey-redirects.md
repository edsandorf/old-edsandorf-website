---
layout: post
title: "The use of URL variables and redirects to handle survey respondents"
header-image: /assets/img/blogs/2020-03-06-persistent-storage-01.jpg
author: Erlend
image: /assets/img/blogs/2020-03-06-persistent-storage-01.jpg
category: blog
excerpt: A post about how I set up persistent storage and transfer of data from a Shiny survey to MariaDB using SSL certificates.
---

When we self-host and run a survey, we often need to work with a survey panel company to get our samples. In this post, I will go through how we handled this for [the INSPiRE Project](https://inspire-project.info) survey. In a previous post, I talked about how we secured our survey responses and deposited them in our database.



# Setting up the website
The survey website itself was hosted on Github Pages and the Shiny application embedded in an `<iframe>`. Embedding the application in an `<iframe>` allows you to access the survey on a website you control (with a custom domain and HTTPS). We created the website in

Because the application exists inside an `<iframe>`, it does not have access to the URL in the browser window. To solve this, we use a small JavaScript that effectively grabs the URL parameters, creates a custom URL for that respondent and feeds it to the `src` parameter. Now, we can capture the URL parameters from inside our Shiny App.

```html
<html>
  <head>
    <style type = "text/css">
      #survey {
        border: none;
        width: 100%;
        height: 100%;
      }
    </style>
  </head>
  <body>
    <script type = "text/javascript">
      window.onload = initialize;

      function initialize() {
        const url_parameters = new URLSearchParams(window.location.search);
        const panel_id = url_parameters.get("ID");
        var new_url = "https://this-is-my-survey.shinyapps.io/this-is-my-survey/?ID=" + panel_id;
        document.getElementById("survey").src = new_url;
      }
    </script>

    <iframe id = "survey" src = "about:blank" frameborder = "0"></iframe>

  </body>
</html>
```

# Capturing URL variables



# Creating custom URL redirects
