node {
   stage('Preparation') { // for display purposes
      // Get some code from a GitHub repository
      def scmData = checkout([
        $class: 'GitSCM',
        branches: [[name: 'master']],
        doGenerateSubmoduleConfigurations: false,
        extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'tempDir']],
        submoduleCfg: [],
        userRemoteConfigs: [[url: 'https://github.com/meekrosoft/gildedrose.git']]
      ])
      script {
          def pomFile = readMavenPom(file: 'pom.xml')
          if (pomFile.version.endsWith('-SNAPSHOT')) {
              def currentDateTime = new Date()
              String timeStamp = currentDateTime.format("yyyy.MM.dd'T'HH.mm.ss.S")
              String commitSha1 = scmData.GIT_COMMIT.trim()
              pomFile.version = "${pomFile.version.replace('-SNAPSHOT', '')}-${timeStamp}-${commitSha1}"
              writeMavenPom(file: 'pom.xml', model: pomFile)
          }
          stash (name: 'metadataFile', includes: 'pom.xml')
      }
   }
   stage('Build') {
     sh 'cat $PWD/tempDir/pom.xml'
     sh 'docker run -i --rm --name my-maven-project -v ~/.m2:/root/.m2  -v "$PWD/tempDir":/usr/src/mymaven -w /usr/src/mymaven maven:3-jdk-8 mvn install'
   }
   stage('Results') {
      junit '**/target/surefire-reports/TEST-*.xml'
      archive 'target/*.jar'
   }
   stage('Javadoc') {
      sh 'docker run -i --rm --name my-maven-project -v ~/.m2:/root/.m2  -v "$PWD/tempDir":/usr/src/mymaven -w /usr/src/mymaven maven:3-jdk-8 mvn site'
      archive 'target/site/**/*'
   }
   stage('Nexus Upload') {
      sh 'docker run -i --rm --name my-maven-project -v ~/.m2:/root/.m2  -v "$PWD/tempDir":/usr/src/mymaven -w /usr/src/mymaven maven:3-jdk-8 mvn deploy'
      archive 'target/site/**/*'
   }
   stage('Promote to TEST') {
      input "Deploy to STAGING?"
      //deleteDir() // Clean-up working directory
      def scmData = checkout([
          $class: 'GitSCM',
          branches: [[name: 'master']],
          doGenerateSubmoduleConfigurations: false,
          extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'tempDir2']],
          submoduleCfg: [],
          userRemoteConfigs: [[url: 'https://github.com/meekrosoft/MavenRepositoryPromotion.git']]
        ])
      dir ('tempDir2') {
        unstash 'metadataFile'
      }
      sh 'docker run -i --rm --name my-maven-project -v ~/.gradle:/root/.gradle -v ~/.m2:/root/.m2  -v "$PWD/tempDir2":/usr/src/mymaven -w /usr/src/mymaven maven:3-jdk-8 ./gradlew publish -P targetEnvironment=test -P pomFile=pom.xml'
   }
}
