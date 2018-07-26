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

## Configure ES6 ##

### Changes in serverless.yml ###
1. Change ```runtime: nodejs6.10``` to ```runtime: nodejs8.10```
1. Add plugins section and include [serverless-webpack](https://github.com/serverless-heaven/serverless-webpack) plugin

```yml
plugins:
  - serverless-webpack
```
1. Create a function which gets triggered by a http event

```yml
functions:
  news:
    handler: handler.news
    events:
      - http:
          path: /news
          method: get
```
### Changes in handler.js ###


### Install npm packages ###
Inside of the terminal, navigate to project folder and run the following
```bash
1. npm init
1. npm install webpack serverless-webpack webpack-node-externals --save-dev
1. npm install babel babel-cli babel-loader babel-core babel-preset-env --save-dev
1. npm install aws-sdk --save-dev
```