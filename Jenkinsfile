def readProperties()
{
	def properties_file_path = "${workspace}" + "@script/properties.yml"
	def property = readYaml file: properties_file_path

    env.APP_NAME = property.APP_NAME
    env.MS_NAME = property.MS_NAME
    env.BRANCH = property.BRANCH
    env.GIT_SOURCE_URL = property.GIT_SOURCE_URL
    env.SONAR_HOST_URL = property.SONAR_HOST_URL
    env.envor = property.envor
    env.size = property.envor.size()
    env.CODE_QUALITY = property.CODE_QUALITY
    env.UNIT_TESTING = property.UNIT_TESTING
    env.CODE_COVERAGE = property.CODE_COVERAGE
    env.FUNCTIONAL_TESTING = property.FUNCTIONAL_TESTING
    env.SECURITY_TESTING = property.SECURITY_TESTING	
    
}

def firstTimeDevDeployment(projectName,msName){
    openshift.withCluster() {
        openshift.withProject(projectName) {
            def bcSelector = openshift.selector( "bc", msName)
            def bcExists = bcSelector.exists()
            if (!bcExists) {
                openshift.newApp("redhat-openjdk18-openshift:1.1~${GIT_SOURCE_URL}","--strategy=source")
                sh 'sleep 120'
                openshiftTag(namespace: projectName, srcStream: msName, srcTag: 'latest', destStream: msName, destTag: 'test')
                openshiftTag(namespace: projectName, srcStream: msName, srcTag: 'latest', destStream: msName, destTag: 'prod')
            } else {
                sh 'echo build config already exists in development environment'  
            } 
        }
    }
}

def firstTimeTestDeployment(sourceProjectName,destinationProjectName,msName){
    openshift.withCluster() {
        openshift.withProject(destinationProjectName){
	    def dcSelector = openshift.selector( "dc", msName)
            def dcExists = dcSelector.exists()
	    if(!dcExists){
	    	openshift.newApp(sourceProjectName+"/"+msName+":"+"test")   
	    }
            else {
                sh 'echo deployment config already exists in testing environment'  
            } 
        }
    }
}

def firstTimeProdDeployment(sourceProjectName,destinationProjectName,msName){
    openshift.withCluster() {
        openshift.withProject(destinationProjectName){
	    def dcSelector = openshift.selector( "dc", msName)
            def dcExists = dcSelector.exists()
	    if(!dcExists){
	    	openshift.newApp(sourceProjectName+"/"+msName+":"+"prod")   
	    }
            else {
                sh 'echo deployment config already exists in production environment'  
            } 
        }
    }
}
def DatabaseDeployment(projectName,msName){
    openshift.withCluster() {
        openshift.withProject(projectName) {
            def bcSelector = openshift.selector( "bc", msName)
            def bcExists = bcSelector.exists()
            if (!bcExists) {
                openshift.newApp("-e MYSQL_USER=admin","-e MYSQL_PASSWORD=admin","-e MYSQL_DATABASE=admin","registry.access.redhat.com/rhscl/mysql-56-rhel7")
                sh 'sleep 120'
                openshiftTag(namespace: projectName, srcStream: msName, srcTag: 'latest', destStream: msName, destTag: 'test')
                openshiftTag(namespace: projectName, srcStream: msName, srcTag: 'latest', destStream: msName, destTag: 'prod')
            } else {
                sh 'mvn flyway:migrate'  
            } 
        }
    }
}

def buildApp(projectName,msName){
    openshift.withCluster() {
        openshift.withProject(projectName){
            openshift.startBuild(msName,"--wait")   
        }
    }
}

def deployApp(projectName,msName){
    openshift.withCluster() {
        openshift.withProject(projectName){
            openshiftDeploy(namespace: projectName,deploymentConfig: msName)
        }
    }
}

podTemplate(cloud:'openshift',label: 'selenium', 
  containers: [
    containerTemplate(
      name: 'jnlp',
      image: 'cloudbees/jnlp-slave-with-java-build-tools',
      alwaysPullImage: true,
      args: '${computer.jnlpmac} ${computer.name}'
    )])
{
node 
{
   def MAVEN_HOME = tool "MAVEN_HOME"
   def JAVA_HOME = tool "JAVA_HOME"
   env.PATH="${env.PATH}:${MAVEN_HOME}/bin:${JAVA_HOME}/bin"
   
   stage('First Time Deployment'){
	   readProperties()
        
   }
   
   stage('Checkout')
   {
       checkout([$class: 'GitSCM', branches: [[name: "*/${BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: "${GIT_SOURCE_URL}"]]])
   }

   stage('Initial Setup')
   {
       sh 'mvn clean compile'
   }
   if(env.UNIT_TESTING == 'True')
   {
   	stage('Unit Testing')
   	{
        	sh 'mvn test'
   	}
   }
   if(env.CODE_COVERAGE == 'True')
   {
   	stage('Code Coverage')
   	{
		sh 'mvn package'
   	}
   }
   if(env.CODE_QUALITY == 'True')
   {
   	stage('Code Quality Analysis')
   	{
       		sh 'mvn sonar:sonar -Dsonar.host.url="${SONAR_HOST_URL}"'
   	}
   }
   
   

   stage('Dev - Build Application')
   {
	   firstTimeDevDeployment(env.envor[0], "${MS_NAME}")
       buildApp("${APP_NAME}-dev", "${MS_NAME}")
   }

   stage('Dev - Deploy Application')
   {

       deployApp("${APP_NAME}-dev", "${MS_NAME}")
   }

   stage('Tagging Image for Testing')
   {
       openshiftTag(namespace: '$APP_NAME-dev', srcStream: '$MS_NAME', srcTag: 'latest', destStream: '$MS_NAME', destTag: 'test')
   }

   stage('Test - Deploy Application')
   {
	   firstTimeDevDeployment(env.envor[1], "${MS_NAME}")
	   deployApp("${APP_NAME}-test", "${MS_NAME}")
   }
   stage('Jmeter')
   {
	sh 'mvn verify'
   }
   node('selenium')
   {
      if(env.FUNCTIONAL_TESTING == 'True')
      {
	stage('Integration Testing')
	{
	    container('jnlp')
	    {
	         checkout([$class: 'GitSCM', branches: [[name: "*/${BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: "${GIT_SOURCE_URL}"]]])
		 sh 'mvn integration-test'
	    }
	 }
      }
    }
	
	if(env.SECURITY_TESTING = 'True')
	{
   	 	stage('Security Testing')
    		{
        		sh 'mvn findbugs:findbugs'
    		}	
	}

    stage('Tagging Image for Testing')
    {
        openshiftTag(namespace: '$APP_NAME-dev', srcStream: '$MS_NAME', srcTag: 'latest', destStream: '$MS_NAME', destTag: 'prod')
    }	
    
    stage('Deploy to Production approval')
    {
	    firstTimeDevDeployment(env.envor[2], "${MS_NAME}")
       input "Deploy to Production Environment?"
    }
	
    stage('Prod - Deploy Application')
    {
       deployApp("${prod}", "${MS_NAME}")
    }	
 
}
}	
