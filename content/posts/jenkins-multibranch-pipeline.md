---
title: "Jenkins Multibranch Pipeline"
description: "How to properly integra git flow concepts with pipelines on Jenkins"
date: 2020-08-16T13:17:52-03:00
draft: true
tags: [
  "Jenkins",
  "Pipeline",
  "CI/CD",
]
---

This tutorial is going to be a simple and straightforward explanation about using Multibranch Pipeline project on Jenkins

First let take for example a Jenkinsfile as bellow:

```groovy
pipeline {
  agent any

  options{
    timeout(time: 10, unit: "MINUTES")
  }

  stages {

    /* This stage will run for every branch. */
    stage('BUILD'){
      steps{
        echo 'do something do build'
      }
    }

    /* This stage will only run when the branch running the build is the develop branch.*/
    stage('Deploy to Development'){
      when {
        branch develop
      }
      steps {
        echo "do something to deploy"
      }
    }
  }


}
```
