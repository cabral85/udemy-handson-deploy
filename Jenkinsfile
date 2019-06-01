node{

	stage('Pull'){
		git url: 'https://github.com/cabral85/udemy-handson-deploy.git'
	}
		sh "/u01/maven/bin/mvn clean package -DskipTests"
	stage('Build'){
	}
	stage('Move'){
		sh "rm -f /data/testeweb.war"
		sh "mv target/testeweb.war /data/testeweb.war"
	}
	stage('Deploy'){
		sh "curl --header '\"Content-Type: application/json\"' --request POST --data \'{\"file\":\"testeweb.war\",\"app\":\"testeweb\",\"server\":\"ADMIN_SERVER\"}\' http://172.17.0.2:8080/deploy"
	}
}