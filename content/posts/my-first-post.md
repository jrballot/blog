---
title: "My First Post"
date: 2020-08-16T13:17:52-03:00
draft: false
---


# This is a simple test of deployment using Hugo blog/site platform deployed to Github pages


```shellscript
echo "some code goes here"
```

> Nodes are also importante

**The Jenkinsfile**

```groovy
pipeline {
	agent any

	options{
		timeout(time: 10, unit: "MINUTES")
	}

	stages {
		stage('BUILD'){
			steps{
				sh 'sleep 15'
			}
		}
	}
}
```
