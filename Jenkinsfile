node{

	stage('Pull'){
		git url: 'https://github.com/cabral85/udemy-handson-deploy.git'
	}
	stage('Build'){
		sh "/u01/maven/bin/mvn clean package -DskipTests"
	}
	stage('Move'){
		sh "cp -rf target/testeweb.war /data"
	}
	stage('Deploy'){
		sh "curl --header '\"Content-Type: application/json\"' --request POST --data ''{\"file\":\"testeweb.war\",\"app\":\"testeweb\",\"server\":\"ADMIN_SERVER\"}'' http://172.17.0.2:8080/deploy"
	}
}