One html page send a POST json to logstash and the kibana plot a dashboard



projeto:

Uma Pagina HTML envia um arquivo json para o logstash e o kibana cria um dashboard com essas informacoes, tambem sao usadas informacoes sobre a localizacao aproximida do usuario atraves da geolocation do proprio navegador e no final, e criado um mapa com a localizacao aproximada de cada usuario detalhando operadora, cidade, regiao, estado, pais, tipo de navegador web, sistema operacional, etc.

apos instalado, links para acesso:

obs: alterar para o IP/dns desejado nos arquivos

pagina html:

http://finalproject.eastus.cloudapp.azure.com/

Elasticsearch:

http://finalproject.eastus.cloudapp.azure.com:8080



##kibana dashboard

![kibana Dashboard](../main/images/dasboard1.png)


![kibana Dashboard](../main/images/dashboard2.png)


![kibana Dashboard](../main/images/dashboard4.png)


![kibana Dashboard](../main/images/dashboard5.png)



pagina html

![Formulario](../main/images/formulario.png)



Steps to install elasticsearch, kibana,logstash and nginx:

Se quiser, É possivel usar o terraform para fazer o deploy de uma vm ubuntu no azure com o arquivo main.tf

referencias:

https://www.codeproject.com/Articles/5260755/Create-an-Azure-Virtual-Machine-with-Terraform

https://learn.hashicorp.com/tutorials/terraform/azure-build?in=terraform/azure-get-started

atencao, necessario instalar o az cli antes:

apt install azure-cli
sudo apt-get install ca-certificates curl apt-transport-https lsb-release gnupg

az login


No azure, abrir as seguintes portas:

![Formulario](../main/images/ports_azure.png)



apt-get install ssh net-tools

Instalando elastic search:

wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

apt-get install apt-transport-https -y

echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

apt-get update && sudo apt-get install elasticsearch -y

configurando systemd

/bin/systemctl daemon-reload && /bin/systemctl enable elasticsearch.service

log information on systemd journal

journalctl -f

configuração do elasticsearch

sed -i '$adiscovery.type: single-node' /etc/elasticsearch/elasticsearch.yml

sed -i '$ahttp.port: 9200' /etc/elasticsearch/elasticsearch.yml

sed -i '$anetwork.host: 0.0.0.0' /etc/elasticsearch/elasticsearch.yml

sed -i '$anetwork.bind_host: 0.0.0.0' /etc/elasticsearch/elasticsearch.yml

sed -i '$anetwork.publish_host: 0.0.0.0' /etc/elasticsearch/elasticsearch.yml

sed -i '$axpack.security.enabled: false' /etc/elasticsearch/elasticsearch.yml

mkdir /etc/systemd/system/elasticsearch.service.d

echo -e "[Service]\nTimeoutStartSec=180" | sudo tee

systemctl daemon-reload

systemctl start elasticsearch

instalando kibana

apt-get install kibana -y

###instalar java

apt install default-jre default-jdk software-properties-common -y

##Se quiser instalar manualmente o java:

apt-key adv --keyserver keyserver.ubuntu.com --recv-keys EA8CACC073C3DB2A

add-apt-repository ppa:linuxuprising/java

#baixar java

https://www.oracle.com/java/technologies/javase/jdk11-archive-downloads.html

#passo a passo

https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-on-ubuntu-20-04#installing-the-default-jrejdk%20in%20our%20guide

mkdir -p /var/cache/oracle-jdk11-installer-local/

cp /home/user/jdk-11.0.12_linux-x64_bin.tar.gz /var/cache/oracle-jdk11-installer-local

apt install oracle-java11-installer-local -y

update-alternatives --config java

update-alternatives --config javac

setar o 2 ( baixado )

sed -i '$aJAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"' /etc/environment

source /etc/environment

echo $JAVA_HOME

/bin/systemctl daemon-reload

/bin/systemctl enable kibana.service

systemctl start kibana.service

instalar nginx

apt-get install nginx -y

#adicionar usuario/senha para acessar

echo "kibanaadmin:openssl passwd -apr1" | sudo tee -a /etc/nginx/htpasswd.users

senha: admin

o servidor escuta no finalproject.eastus.cloudapp.azure.com e redirecionará para o um serviço local ouvindo na 5601 quando acessado externamente na 8080
root@user-VirtualBox:/# cat /etc/nginx/sites-available/finalproject

server {

listen 8080;

server_name finalproject.eastus.cloudapp.azure.com;
auth_basic "Restricted Access";

auth_basic_user_file /etc/nginx/htpasswd.users;

location / {
    proxy_pass http://localhost:5601;
	
    proxy_http_version 1.1;
	
    proxy_set_header Upgrade $http_upgrade;
	
    proxy_set_header Connection 'upgrade';
	
    proxy_set_header Host $host;
	
    proxy_cache_bypass $http_upgrade;
}
}

ln -s /etc/nginx/sites-available/finalproject /etc/nginx/sites-enabled/finalproject

instalando logstash

apt-get install logstash -y

criar o arquivo de configuração do logstash que receberá os "JSONS" via http e enviará para o elasticsearch

atencao:

necessario baixar o arquivo GeoLite2-City.mmdb de www.maxmind.com/en/accounts/XXXXXX/downloads . o arquivo e o GeoLite2-City:

format: GeoIP2 Binary (.mmdb)

extrair em etc/logstash/conf.d/

ajustar o path na linha database

root@projetofinal:/etc/logstash/conf.d/# cat http.conf

input {

http {

host => "0.0.0.0"

port => 8085

codec => json

response_headers => {

"Access-Control-Allow-Origin" => "*"

"Content-Type" => "text/plain"

"Access-Control-Allow-Headers" => "Origin, X-Requested-With, Content-Type, Accept"

}

}

}

filter {

grok {

match => { "message" => "%[headers][http_user_agent]" }
} date {

match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
}

geoip {

source => "host"

target => "geoip"

database => "/etc/logstash/conf.d/GeoLite2-City_20211019/GeoLite2-City.mmdb"

add_field => [ "[geoip][coordinates]", "%{[info][longitude]}" ]

add_field => [ "[geoip][coordinates]", "%{[info][latitude]}"  ]
}

mutate {

convert => [ "[geoip][coordinates]", "float"]

rename => { "[info][país]" => "[info][country]" }

rename => { "[info][operadora]" => "[info][operator]" }

rename => { "[info][operador]" => "[info][operator]" }

rename => { "[info][região]" => "[info][region]" }

rename => { "[info][cidade]" => "[info][city]" }

add_field => { "dnsname" => "%{host}" }
}

dns {

reverse => [ "dnsname" ]

action => "replace"
} useragent {

source => "[headers][http_user_agent]"
}

}

output {

elasticsearch {

             hosts => "http://localhost:9200"
			 
             index => "logstash-formulario"
			 
             template_name => "meumapa"
			 
            }
			
   stdout { codec => "rubydebug" }
}

Adicionar o bin do logstash ao $PATH

root@projetofinal:/# cat /etc/environment

PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/usr/share/logstash/bin"

JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"

source /etc/environment

criar serviço carregando as configurações do arquivo criado

cat /etc/systemd/system/logstash.service

[Unit]

Description=logstash

[Service]

Type=simple

User=logstash

Group=logstash

EnvironmentFile=-/etc/default/logstash

EnvironmentFile=-/etc/sysconfig/logstash

ExecStart=/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/http.conf

Restart=always

WorkingDirectory=/

Nice=19

LimitNOFILE=16384

TimeoutStopSec=infinity

[Install]

WantedBy=multi-user.target

/bin/systemctl daemon-reload

/bin/systemctl enable logstash.service

systemctl start logstash.service

verificando os jsons no banco do logstash

curl -XGET "http://127.0.0.1:9200/formulario/_search?pretty=true" -H 'Content-Type: application/json'

curl -XGET "http://127.0.0.1:9200/_mapping"

###copiando o index.html e os modulos do elastic

precisa do elasticsearch-js e index.html

depois dar um restart no ngnix

systemctl restart nginx


#importar o dashboard para o kibana

importar o arquivo export.ndjson

![Formulario](../main/images/kibana_import.png)


referencias:

https://support.logz.io/hc/en-us/articles/210207225-How-can-I-export-import-Dashboards-Searches-and-Visualizations-from-my-own-Kibana-


testando e vendo o input no kibana ( o comando abaixo sobe o servico, necessario para do systemd se quiser testar )

/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/http.conf




exemplo de entrada (apos clicar em submit na pagina html):

{

 "pergunta8" => [],
 
"@timestamp" => 2021-11-30T05:34:50.702Z,

   "os_name" => "Windows",
   
 "pergunta6" => "don't share wifi",
 
 "pergunta5" => "consigo me virar bem",
 
  "@version" => "1",
  
   "version" => "96.0.4664.45",
   
     "minor" => "0",
	 
        "os" => "Windows",
		
     "geoip" => {
	 
     "country_code2" => "BR",
	 
     "country_code3" => "BR",
	 
       "coordinates" => [
	   
        [0] -46.6361,
		
        [1] -23.5475
		
    ],
	
         "longitude" => -46.6359,
		 
          "timezone" => "America/Sao_Paulo",
		  
       "region_code" => "SP",
	   
       "region_name" => "Sao Paulo",
	   
          "latitude" => -23.5335,
		  
          "location" => {
		  
        "lon" => -46.6359,
		
        "lat" => -23.5335
		
    },
	
         "city_name" => "São Paulo",
		 
                "ip" => "185.54.230.160",
				
       "postal_code" => "01323",
	   
      "country_name" => "Brazil",
	  
    "continent_code" => "SA"
	
},

 "pergunta4" => "room of my home",
 
      "name" => "Chrome",
	  
    "device" => "Other",
	
   "dnsname" => "185.54.230.160",
   
      "info" => {
	  
     "operator" => "AVAST Software",
	 
     "latitude" => "-23.5475",
	 
         "org2" => "Software",
		 
         "org1" => "AVAST",
		 
    "longitude" => "-46.6361",
	
         "city" => "São Paulo",
		 
       "region" => "São Paulo",
	   
      "country" => "BR"
	  
},

      "host" => "185.54.230.160",
	  
      "tags" => [
	  
    [0] "_grokparsefailure"
	
],

 "pergunta2" => "mobile devices",
 
     "major" => "96",
	 
   "os_full" => "Windows 10",
   
  "os_major" => "10",
  
"os_version" => "10",

 "pergunta1" => "100mb",
 
 "pergunta3" => "Always",
 
     "patch" => "4664",
	 
   "headers" => {
   
     "request_method" => "POST",
	 
             "origin" => "http://finalproject.eastus.cloudapp.azure.com",
			 
       "request_path" => "/formulario/_doc?timeout=120s",
	   
    "accept_encoding" => "gzip, deflate",
	
          "http_host" => "finalproject.eastus.cloudapp.azure.com:8085",
		  
       "content_type" => "application/json",
	   
    "accept_language" => "en-US,en;q=0.9,pt-BR;q=0.8,pt;q=0.7",
	
       "http_version" => "HTTP/1.1",
	   
     "content_length" => "354",
	 
         "connection" => "keep-alive",
		 
        "http_accept" => "*/*",
		
            "referer" => "http://finalproject.eastus.cloudapp.azure.com/",
			
    "http_user_agent" => "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.45 Safari/537.36"
	
}
}
