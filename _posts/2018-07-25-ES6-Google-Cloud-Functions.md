---
layout:             post
title:              "Runing ES6 code in AWS lambda"
menutitle:          "Runing ES6 code in AWS lambda"
date:               2017-07-25 07:23:00
category:           AWS
author:             adimoraret
language:           EN
comments:           false
published:          true
---
## Create aws serveless project ##
I use [serverless](https://serverless.com/framework/docs/) to manage my lambda aws projects. However, by default, when creating the project, javascript code is written in ES5. For serverless installation please refer to [Installing the Serverless Framework](https://serverless.com/framework/docs/providers/aws/guide/installation/) section.

```bash
#creating serverless project
serverless create --template aws-nodejs --path serverless-es6
```
Running this command will result creating "serverless-es6" project.
![serverless create aws project](/assets/posts/2018-07-25/serverless-create-aws-project.png)

Project contains 3 files:
* .gitignore
* handler.js
* serverless.yml

## Configure ES6 function to run as a Lambda Function ##

### Changes in serverless.yml ###

```yaml
service: serverless-es6

provider:
  name: aws
  runtime: nodejs8.10 # Change from nodejs6.10 to nodejs8.10

plugins:
  - serverless-webpack

functions:
  news: # Create a news function which gets triggered by a http event
    handler: handler.news
    events:
      - http:
          path: /news
          method: get

custom: # Add custom section and include webpack configuration
    webpack:
      webpackConfig: 'webpack.config.js'
      includeModules: false
```

### Changes in handler.js ###
Add a javascript function and make sure you have some ES6 features
```javascript
export function news(event, context, callback) {

  const happyDay = () => 'Friday'
  const buildColor = 'green'
  const goodNews = [`Today is ${happyDay()}`, `Build is ${buildColor}`]

  callback(null, {
    statusCode: 200,
    body: JSON.stringify(goodNews),
  })
}
```

### Create webpack.config.js ###
webpack.config.js file is usually at the same level with package.config file. If you want to change the location, you need to update serverless.yml file with the new location.
```javascript
module.exports = {
  entry: { handler: './handler.js' },
  target: 'node',
  mode: 'development'
}
```

### Install npm packages ###
Inside of the terminal, navigate to project folder and run the following
```bash
1. npm init
1. npm install webpack serverless-webpack --save-dev
```

## Run your function ##
Locally
![run lambda function locally](/assets/posts/2018-07-25/serverless-run-lambda-function-locally.png)

Or in the cloud after deploy (``` serverless deploy ```)
![run aws es6 function in aws](/assets/posts/2018-07-25/serverless-run-lambda-function-in-aws.png)