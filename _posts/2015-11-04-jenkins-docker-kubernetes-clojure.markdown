---
layout: post
title:  "Building Clojure projects with the Jenkins Kubernetes Plugin"
excerpt: "Configure Jenkins to build Clojure projects with Leiningen using the Kubernetes plugin"
date:   2015-11-05 10:30
categories: clojure
tags: clojure jenkins kubernetes docker
---

The [Jenkins Kubernetes plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) is a very cool bit of tech that allows Jenkins to dynamically provision workers from a pool of Docker hosts managed by Kubernetes.

Setting up a worker with `lein` is straightforward. With the Kubernetes plugin installed, visit your Jenkins configuration page and add a pod template along the lines of

![Kubernetes pod template](/images/kubernetes-pod-template.png)

The Docker image url refers to [this Quay.io repository](https://quay.io/repository/stuarth/docker-jnlp-clojure-slave), based on [a Dockerfile on Github](https://github.com/stuarth/docker-jnlp-clojure-slave/blob/master/Dockerfile) that adds lein to the base `jenkinsci/jnlp-slave` image.

Configure your project to use the pod and you're set.

![Jenkins Project config](/images/jenkins-project-config.png)

Note that only `lein` is supported currently -- please submit a PR or open an issue if you'd like to see additional build tools supported!
