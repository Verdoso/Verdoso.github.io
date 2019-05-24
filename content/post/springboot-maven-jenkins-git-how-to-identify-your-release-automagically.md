---
date: 2018-04-16T18:05:09+02:00
tags:
- Jenkins
- Spring Boot
- Maven
- Deployment Pipelines
featured_image: ''
title: 'SpringBoot, Maven, Jenkins & Git: How to identify your release automagically'

---
One of the issues that usually comes up when you have an application up & running is finding out what’s deployed in the different servers so you can know if what you committed in the source code repository is deployed in those releases or not.

That thing that sounds so trivial becomes not that easy when you add automatic deployment, branches and everything that allows you to be “agile”, so you need to add some automatic mechanisms to your release and deployment process in order to make it easy again.

As a brief summary, these are the things we want to facilitate:

* Given a release, we want to know what is included in that release.
* Given a deployed application, we want to know which release is deployed.
* Having a look at the source code repository, we want to know if a given commit is included in a release

### **First step, identifying what’s in a deployed** [**Spring Boot**](https://projects.spring.io/spring-boot/) **application**

One of the easiest ways is to include the [maven git commit id plugin](https://github.com/ktoso/maven-git-commit-id-plugin) , something you can do easily, just by adding this, for example, to your pom.xml file

{{< gist Verdoso fef3719708380e7cd86286a958016e10}}

After that, enable the [Spring Boot info actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html#production-ready-application-info) by adding the actuator dependency

{{< gist Verdoso a6fcee347ad297e6ab5c9fad61ee317e}}

After that and after creating a release, you should be able to identify what’s included in the deployed release by looking at your.app.com/info (or whatever custom path you configured your actuator to answer at).

You can get more information about the whole process at the following Baeldung tutorials: [Injecting Git Information Into Spring](http://www.baeldung.com/spring-git-information) and [Spring Boot Actuator](http://www.baeldung.com/spring-boot-actuators).

### **Second step, using Maven plugins in Jenkins to set a specific version and tag it in the source code repository**

Now that we know we can get the release information from the deployed application, what we need now is to add the information that we want in the release. In order to do that, we will use various Maven plugins: [Build Helper](https://www.mojohaus.org/build-helper-maven-plugin/), [Versions](https://www.mojohaus.org/versions-maven-plugin/) and the [Maven SCM Plugin](https://maven.apache.org/scm/maven-scm-plugin/). With those plugins, we can read the current version of the project using “_build-helper:parse-version_”, use the properties configured by the plugin to set a new version with “_versions:set -DnewVersion=\\${parsedVersion.majorVersion}.\\${parsedVersion.minorVersion}.\\${parsedVersion.incrementalVersion}-BUILD-${BUILD_NUMBER} versions:commit_”.

After setting the new version (note that if the project is a –SNAPSHOT version, it removes the suffix), we will also use “_mvn –Dusername=Jenkins scm:tag_” to tag automatically the release in the source repository.

Remember that In order to be able to perform the scm commands, you have to set up the scm section of your pom correctly, include the Maven scm plugin, and define a user with the proper permissions in your settings.xml file.

For example, you could have something like that in your pom.xml

{{< gist Verdoso 7ebfcd323c9c47eb6b3d45bee59de44c}}

And something like this in your settings.xml

{{< gist Verdoso 109b90654ff8fe21ea570b3431770b35}}

Check other options at the instructions about [Bootstrapping a Project Using a POM](https://maven.apache.org/scm/maven-scm-plugin/examples/bootstrapping-with-pom.html) of the [Maven SCM Plugin](https://maven.apache.org/scm/maven-scm-plugin/).

All in all, one can add something like the following snippets to your project [Jenkinsfile](https://jenkins.io/doc/book/pipeline/jenkinsfile/):

{{< gist Verdoso 04f83b21808174455faf08dab16c389b}}

In them, we define a parameter (see [Handling parameters](https://jenkins.io/doc/book/pipeline/jenkinsfile/#handling-parameters) so we can toggle the feature on/off and then a [Stage](https://jenkins.io/doc/pipeline/steps/pipeline-stage-step/) with some steps that that set the proper version, deploy the release to the artifacts repository and then tags the release with the build number at the source code repository.

Et voilà, when we execute the Pipeline and set the UPLOAD_TO_REPOSITORY variable set to true:

* The artifact uploaded to the repository will include the Jenkins build number.
* When the artifact is deployed, it will show at the info endpoint the last commit information and the build number from the Jenkins execution that generated it.
* The source code repository will have a tag with the version number included in the artifact at the point where it was created.

With that information, it is very easy to find out what is deployed and given that the info endpoint response type is JSON, it would be also very easy to integrate that information in other tools, like a screen where the versions deployed in the different servers are shown, a checker that verifies that all the servers in a balanced cluster are running the same version, etc.

And that’s all for today, happy coding.

D.