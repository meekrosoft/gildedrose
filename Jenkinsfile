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
      dir ('tempDir') {
        sh 'cat pom.xml'
        script {
            def pomFile = readMavenPom(file: 'pom.xml')
            if (pomFile.version.endsWith('-SNAPSHOT')) {
                echo "\n\nThis IS a SNAPSHOT version!\n\n"
                def currentDateTime = new Date()
                String timeStamp = currentDateTime.format("yyyy.MM.dd'T'HH.mm.ss.S")
                String commitSha1 = scmData.GIT_COMMIT.trim()
                pomFile.version = "${pomFile.version.replace('-SNAPSHOT', '')}-${timeStamp}-${commitSha1}"
                writeMavenPom(file: 'pom.xml', model: pomFile)
            }
            else{
                echo "\n\nNot a SNAPSHOT version\n\n"
            }
            stash (name: 'metadataFile', includes: 'pom.xml')
            echo "\nJust finished stashing this pom.xml file:"
            sh 'cat pom.xml'
        }
      }
   }
   stage('Build') {
      dir ('tempDir') {
        unstash 'metadataFile'
      }
     sh 'docker run -i --rm --name my-maven-project -v ~/.m2:/root/.m2  -v "$PWD/tempDir":/usr/src/mymaven -w /usr/src/mymaven maven:3-jdk-8 mvn install'
   }
   stage('Results') {
      junit '**/target/surefire-reports/TEST-*.xml'
      archive 'target/*.jar'
   }
   stage('Javadoc') {
      dir ('tempDir') {
        unstash 'metadataFile'
      }
      sh 'docker run -i --rm -v ~/.m2:/root/.m2  -v "$PWD/tempDir":/usr/src/mymaven -w /usr/src/mymaven maven:3-jdk-8 mvn site'
      archive 'target/site/**/*'
   }
   stage('Nexus Upload') {
      dir ('tempDir') {
        unstash 'metadataFile'
      }
      sh 'docker run -i --rm -v ~/.m2:/root/.m2  -v "$PWD/tempDir":/usr/src/mymaven -w /usr/src/mymaven maven:3-jdk-8 mvn deploy'
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
