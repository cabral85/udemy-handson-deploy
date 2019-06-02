# udemy-handson-deploy
# Root no servidor do weblogic
docker exec --interactive --tty --user root --workdir / 8013340aab31 bash

# RepositÃ³rio Python -- https://copr.fedorainfracloud.org/coprs/pypa/pypa/repo/epel-7/pypa-pypa-epel-7.repo
cat > /etc/yum.repos.d/pip.repo

[pypa-pypa]
name=Copr repo for pypa owned by pypa
baseurl=https://copr-be.cloud.fedoraproject.org/results/pypa/pypa/epel-7-$basearch/
type=rpm-md
skip_if_unavailable=True
gpgcheck=1
gpgkey=https://copr-be.cloud.fedoraproject.org/results/pypa/pypa/pubkey.gpg
repo_gpgcheck=0
enabled=1
enabled_metadata=1

#Baixa o Maven no servidor do Jenkins e descompacta na pasta /u01


# Instalar Python e PIP no servidor weblogic
yum update
yum install python-pip python-wheel
yum upgrade python-setuptools

--> Criar webservice python com flask para receber dados do deploy
#!/usr/bin/env python
# -*- coding: utf-8 -*-
from flask import Flask, render_template
from flask import request
from flask import Response
from flask import json
import os
import sys
import time

app = Flask(__name__)

@app.route('/deploy', methods=['POST'])
def deploy():
	# Pega o conteudo do POST e divide em variaveis
	content = request.get_json()
	app_name = content['app']
	file = content['file']
	server = content['server']
	# Monta o nome de um arquivo de log para execucao do script de deploy
	logfile = "/logs/%s" % (app_name + time.strftime("%Y%m%d-%H%M%S") + ".txt")
	# Esse script de deploy vai rodar um shell do weblogic que contem informacoes como variaveis de ambiente e bibliotecas
	# E em seguida vai rodar nosso script python para fazer o deploy no weblogic
	os.system("sh /u01/oracle/weblogic/oracle_common/common/bin/wlst.sh /u01/scripts/deploy.py %s %s %s > %s" % (app_name, file, server, logfile))
	# Abre o arquivo de log gerado no deploy e devolve no webservice para logar no Jenkins
	text = open(logfile, 'r+')
	content = text.read()
	text.close()
	return Response(content, mimetype='text/plain')
app.run(host='0.0.0.0', port=8080)


# Python acionado pelo webservice, esse script se conecta ao weblogic que esta no mesmo servidor e fazer o deploy
import sys
import re

# Pega os dados do deploy que vieram por argumentos na chamada
app=sys.argv[1]
file=sys.argv[2]
server=sys.argv[3]

print 'Conectando ao servidor'
connect('weblogic', 'welcome1', 'localhost:7001', adminServerName=server)
# Valida se no diretorio de deploys se ja existe nossa aplicacao
appList = re.findall(app, ls('/AppDeployments'))
# Caso exista, realiza o undeploy (Remove a aplicacao do weblogic)
if len(appList) >= 1:
	print 'Parando e removendo aplicacao'
	stopApplication(app)
	undeploy(app)

#Faz o deploy / Instalacao da aplicacao no weblogic
print 'Instalando aplicacao'
deploy(app, '/data/' + file)
# Inicia a aplicacao
startApplication(app)

print 'Desconectando do servidor'
disconnect()
exit()


# Criar JenkinsFile com passos do deploy no projeto
# Primeiro passo faz copia / clone do codigo que esta no github
# Segundo passo, faz build da aplicacao com os recursos do maven
# Terceiro passo, move a aplicacao para a pasta compartilhada entre os servidores
# Quarto passo, aciona um script shell que vai rodar um comando CURL fazendo POST com os dados de deploy para o servidor do weblogic
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
		sh "/u01/send_curl_deploy.sh"
	}
}


# Script Curl
curl --header "Content-Type: application/json" --request POST --data '{"file":"testeweb.war","app":"testeweb","server":"ADMIN_SERVER"}' http://172.17.0.2:8080/deploy
