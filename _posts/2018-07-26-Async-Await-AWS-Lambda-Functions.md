---
layout:             post
title:              "Async Await in AWS Lambda functions"
menutitle:          "Async Await in AWS Lambda functions"
date:               2017-07-26 07:23:00
category:           AWS
author:             adimoraret
language:           EN
comments:           false
published:          true
---
## What I will implement ##
Using [Serverless framework](https://serverless.com/framework/docs/), ES6, async and await, I will implement an AWS lambda function that gets the latest Jobs from HackerNews. I will use [HackerNews API](https://github.com/HackerNews/API).

I'll clone a sample lambda function project created in this [article](/ES6-Code-AWS-Lambda) and I'll install ```node-fetch``` which is a library to create http requests and it supports promises. This npm package is equivalent to window.fetch which can be used in regular ES6 code inside the browser. 

```bash
git clone https://github.com/adimoraret/serverless-aws-es6-demo.git serverless-aws-async-await
cd serverless-aws-async-await
npm install node-fetch --save
```
```node-fetch``` npm module is, of course, external module and it needs to be included inside the package in order to be uploaded into the cloud.  

## Include external packages ##
We'll need to change ```serverless.yml``` to include, into the package, all third party modules which our function depends on, at the runtime. These packages should be defined into "dependencies" section from package.json. In our case only one package ```node-fetch```

```bash
custom:
    webpack:
      webpackConfig: 'webpack.config.js'
      includeModules: true
```
If we run ```serverless package``` we'll see the external module will be included into the package.
![serverless package external module](/assets/posts/2018-07-26/serverless-package-external-module.png)

Trying to run the function in this state, we'll get the following errors:

```bash
ERROR in ./node_modules/node-fetch/lib/index.mjs 496:11-22
Can't import the named export 'PassThrough' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 497:11-22
Can't import the named export 'PassThrough' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 1413:27-38
Can't import the named export 'PassThrough' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 1460:29-40
Can't import the named export 'PassThrough' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 1049:34-46
Can't import the named export 'STATUS_CODES' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 1194:9-15
Can't import the named export 'format' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 1142:16-21
Can't import the named export 'parse' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 1145:16-21
Can't import the named export 'parse' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 1149:15-20
Can't import the named export 'parse' from non EcmaScript module (only default export is available)
 @ ./handler.js

ERROR in ./node_modules/node-fetch/lib/index.mjs 1352:51-58
Can't import the named export 'resolve' from non EcmaScript module (only default export is available)
 @ ./handler.js
```

That is because webpack will try to include ```node-fetch``` module into the final bundle. We don't want that since this module will be packaged separately. In order to exclude these files from the bundle we'll need to install the following module: ```webpack-node-externals```

```bash
npm install webpack-node-externals --save-dev
```

and use it into our webpack.config.js file

```javascript
var nodeExternals = require('webpack-node-externals')

module.exports = {
  entry: { handler: './handler.js' },
  target: 'node',
  mode: 'development',
  externals: [nodeExternals()]
}
```

## Writing lambda function ##

Here is the lambda function written fully in ES6 code and using ```async``` and ```await``` instead of promises.

```javascript
import fetch from 'node-fetch'

const getHackerNewsLatestJobs = async () => {
  const response = await fetch('https://hacker-news.firebaseio.com/v0/jobstories.json')
  const jobIds = await response.json()

  const jobs = jobIds.map(async (jobId) => {
    const jobResponse = await fetch(`https://hacker-news.firebaseio.com/v0/item/${jobId}.json`)
    const job = await jobResponse.json()
    return {
      title: job.title,
      url: job.url
    }
  })

  return await Promise.all(jobs)
}

export async function jobs(event, context, callback) {
  const latestJobs = await getHackerNewsLatestJobs()
  callback(null, {
    statusCode: 200,
    body: JSON.stringify(latestJobs),
  })
}
```

## Run lambda function ##
Locally
![run lambda function locally](/assets/posts/2018-07-26/serverless-run-async-lambda-function-locally.png)

Or in the cloud after deploy (``` serverless deploy ```)
![run aws es6 function in aws](/assets/posts/2018-07-26/serverless-run-async-lambda-function-in-aws.png)

Full example can be found on [GitHub](https://github.com/adimoraret/serverless-es6-async-await-demo)