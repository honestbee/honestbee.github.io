---
layout: post
title:  "Building a web app using Google Sheets"
excerpt: "Build a web app using Google Sheets backend and host it on Google cloud infrastructure for free"
date:   2017-02-09 19:04:33 +0800
categories: [integrations]
tags: [google,google-sheet]
author: Taher Dhilawala
---

I am sure we all use and love Google Sheets. But I think very few know that we can actually build a small web application on top of Google Sheets (leveraging Google cloud infrastructure) using [Google App Scripts](https://developers.google.com/apps-script/)

## Background

At [honestbee](https://honestbee.sg), we recently wanted to collect data from our partners, who are not tech savvy. So, we created a simple spreadsheet for them to be able to add their data. There can be no argument about spreadsheets being the most simplest way of collecting data without any hassle. But, the data we were collecting also had images. A usual way of collecting images would be to ask the partners to zip everything and name the image files same as the key column in the excel spreadsheet. But as I said, they are not tech savvy and this process is prone to errors. So, we needed a way for them to upload images so that it gets stored directly on our servers. All this, without them leaving the comfort of Google spreadsheet

## App Scripts

The solution we came up with was to build a small web application with an image uploader. The interesting thing was - the web application runs on top of Google Sheets. So no need to deploy it anywhere. You can even reference the Google Sheets values inside the html application using [Google Sheets API](https://developers.google.com/sheets/)

## Script Editor

Start by opening the script editor from Google Sheets as shown below

![img](/img/posts/web_app_google_sheet/script_editor.png)

You start with a `Code.gs` file. `.gs` stands for google script.

When you make your web application live using Google cloud, it will call `doGet()` method in your `Code.gs` script. That would be your entry point.

`doGet()` for GET request and `doPost()` for POST request.

So, lets write some code,

```
function doGet(e) {
  var queryString = e.queryString;
}
```
You also get the query string passed in the URL. Do note that the queryString you get is a string. You will need to parse it in order to retrieve the `key=value` pairs.

## HTML file

The output of the `doGet()` can be a string, json etc. But it would be great if we can show a real html page. For that, create an `.html` file using the `File > New` in the menu

Lets say you name your file `imageUploader.html`. Then, to render this html file, in your `doGet()` you will do,

```
var htmlTemplate = HtmlService.createTemplateFromFile('imageUploader');
var htmlOutput = htmlTemplate.evaluate().setSandboxMode(HtmlService.SandboxMode.IFRAME);
return htmlOutput;
```

The above snippets tells Google to render the given html template in an IFRAME.

## The Web application

This is what I have in my `Code.gs`

```
function doGet(e) {
  var queryString = e.queryString;

  var name = getQueryStringValue(queryString, "name")

  var htmlTemplate = HtmlService.createTemplateFromFile('imageUploader');

  htmlTemplate.qsName = name; //setting a variable in html template using the query string value

  var htmlOutput = htmlTemplate.evaluate().setSandboxMode(HtmlService.SandboxMode.IFRAME);
  return htmlOutput;
}

// Utility function to fetch key values from query string
function getQueryStringValue(query, key){
  var queryParts = query.split("&");
  if(queryParts && queryParts.length > 0){
    for(var i=0; i<queryParts.length; i++){
      var k = queryParts[i].split("=")[0];
      if(k == key) return queryParts[i].split("=")[1];
    }
  }
}
```

And my HTML code

```
<!DOCTYPE html>
<html>
  <head>
    <base target="_top">
    <script type="text/javascript">
      var name = '<?= qsName ?>';
    </script>
  </head>
  <body onload=>
    <h1>Hello World, from Google cloud</h1>
    <div>
      <label>Value from Query String = </label>
      <span><?= qsName ?></span>
      <!-- we use the value from the variable which was set in the template in google script -->
    </div>
  </body>
</html>
```

The cool part to highlight above is the fact that we can pass the variables from the google script to html template.

Lets publish it!

![img](/img/posts/web_app_google_sheet/publish.png)

![img](/img/posts/web_app_google_sheet/deploy_web.png)

You need to set the following

* **Project version**: For every new deploy you need to increment the project version and give a comment (just like git, it's for versioning)

* **Execute the app as**: Since this app will be running inside the Google Sheets context, you need to specify under what user this app runs. It can be either you (the developer) or the user who is accessing the sheet.

* **Access**: Finally, you can also restrict who can access this web app. Options are `Only myself` and `Anyone`. If you choose to execute the app as `Me` then you also get the third option here, which is `Anyone, even anonymous`. Since its your user, people can use it without having to authenticate with Google.

Hit deploy!

![img](/img/posts/web_app_google_sheet/deployed.png)

Now lets navigate to the URL and pass name as query string. Also, you can test the Web app in sandbox mode which will generate logs (click on `latest code` link)

![img](/img/posts/web_app_google_sheet/web_app.png)

Voila!!

## Google Sheets integration

So far so good, right? Well, what would be more awesome is that we can store web app values in Google Sheets.

Doing that is quite easy using the Google Sheets API. In your `Code.gs` you can access the active sheet like this,

```
function setSheetValue(value){
  var doc = SpreadsheetApp.getActive();
  var sheet = doc.getSheetByName("Sheet 1");
  var range = sheet.getRange("Sheet 1!B2");
  range.setValue(value);
}
```

You can call the above function from your HTML file javascript, like

```
if(google && google.script){
    google.script.run.setSheetValue("some value");
}
```

## Summary

Since you can do HTML and Javascript, you have endless possibilities here. With powerful Google Sheets API and its integration with Google cloud hosting, you can build a simple HTML web app with Google Sheets as your database.

Go try it out :)
