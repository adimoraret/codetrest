---
layout:             post
title:              "Run ES6 & async/await code in Goolge Cloud Functions"
menutitle:          "Run ES6 & async/await code in Goolge Cloud Functions"
date:               2018-07-27 07:23:00
category:           GoogleCloud
author:             adimoraret
language:           EN
comments:           false
published:          true
---
## What I will implement ##
Using [Serverless framework](https://serverless.com/framework/docs/), ES6, async and await, I will implement a Google Cloud Function that gets the latest news from New York Times.

I will start with the code from this AWS [article](/Async-Await-AWS-Lambda-Functions) as a boilerplate for my Google Cloud Functions.

## Add Google Cloud Platform settings ##
Serverless.yml file will need to contain google cloud platform settings like:
* provider name will be google instead of aws
* we'll need to associate the function to a project which it should be defined in Google Cloud Platform
* we'll need to add the path to keyfile used for GCP authentication
* we'll need to include and run npm install for serverless-google-cloudfunctions plugin
* Google Cloud Functions are defined a little different than in AWS. It will be needed a main javascript file that contains all GCP functions definitions. The implementation of the function can be done in different files. There is no restriction regarding that.

```bash
service: serverless-es6-gcp-demo

provider:
  name: google
  runtime: nodejs   runtime: nodejs8
  region: us-central1
  project: your-google-project
  credentials: ~/.gcloud/keyfile.json

plugins:
  - serverless-webpack
  - serverless-google-cloudfunctions

functions:
  newYorkTimesNews:
    handler: newYorkTimesNews
    events:
      - http: newyorktimesnews

custom:
    webpack:
      webpackConfig: 'webpack.config.js'
      includeModules: true
```

## Add JavaScript files ##

```index.js``` file is the entry file that Google Cloud is looking for when running functions.
```javascript
import NewYorkTimesNews from './NewYorkTimesNews'

export async function newYorkTimesNews(request, response) {
  const newYorkTimesNews = new NewYorkTimesNews()
  const news = await newYorkTimesNews.getLatestTechnologyNews()
  response.status(200).send(news)
}
```

```NewYorkTimesNews.js``` file which contains the business logic to retrieve New York Times news. It also contains a reference to an environment variable. Value of the environment variable will be set into the cloud.
```javascript
import fetch from 'node-fetch'
const NewYorkApiKey = process.env.NewYorkApiKey

export default class NewYorkTimesNews {
  async getLatestTechnologyNews() {
    const url = `https://api.nytimes.com/svc/topstories/v2/technology.json?api-key=${NewYorkApiKey}`
    const response = await fetch(url)
    return await response.json()
  }
}
```

## Running the function ##
With all the code ready let's run the deployment of the function.

```bash
npm install
npm install serverless-google-cloudfunctions --save-dev
serverless deploy
```

In Google Cloud Console we'll add NewYorkApiKey environment variable like in the following image
![add environment NewYorkApiKey variable](/assets/posts/2018-07-27/update-google-cloud-function.png)

## Run lambda function ##
Running our lambda function by HTTP url
![successfully google cloud function run](/assets/posts/2018-07-27/success-function-run.png)

Full example can be found on [GitHub](https://github.com/adimoraret/serverless-es6-gcp-demo)
