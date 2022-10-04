# Comando para entrar no container
docker compose exec -it consul01 sh

## Comando para subir o consul em modo dev
consul agent -dev

## Visualizar membros do consul
consul members

### Pela api
curl localhost:8500/v1/catalog/nodes

## Acessando servidor DNS do consul

### Instalar
apk -U add bind-tools

### Comando
dig @localhost -p 8600 consul01.node.consul

### Criando um consul agent
consul agent -server -bootstrap-expect=3 -node=consulserver01 -bind=172.22.0.3 -data-dir=/var/lib/consul -config-dir=/etc/consul.d

**-bootstrap-expect**: Quantidade de servidores que o consul deve esperar para iniciar o cluster
**-node**: Nome do servidor que o agent vai usar
**-bind**: Endereço IP do servidor
**-data-dir**: Diretório onde o consul vai armazenar os dados (é necessário criar antes)
**-config-dir**: Diretório onde o consul vai buscar as configurações (é necessário criar antes)

### Join entre agentes
Entrar em um dos servers consul (docker exec -it consulserver01 sh) e executar o comando abaixo:
consul join <ip_servidor_consul>

### Criando um client
consul agent -bind=172.22.0.5 -data-dir=/var/lib/consul -config-dir=/etc/consul.d

**retry-join**: Comando para o consul tentar se conectar a um servidor caso ele não consiga se conectar no momento da inicialização
```
consul agent -bind=172.22.0.6 -data-dir=/var/lib/consul -config-dir=/etc/consul.d -retry-join=172.22.0.3 -retry-join=172.22.0.4
```

## Registrando serviços
- Criar um json com as configurações do serviço
- Rodar `consul reload` (no container do client pode ser) para recarregar as configurações

### Buscando um serviço
dig @localhost -p 8600 nginx.service.consul (precisa do bind-tools)

consul catalog nodes -service nginx
```
Node            ID        Address     DC
consulclient01  bb65a81c  172.22.0.5  dc1
```

## Health checks

- Instalar o nginx em um dos containers dos clients: `apk -U add nginx`
- Criar o health check no services.json do client:
```JSON
{
	"service": {
		"id": "nginx",
		"name": "nginx",
		"tags": [
			"web"
		],
		"port": 80,
		"check": {
			"id": "nginx",
			"name": "HTTP 80",
			"http": "http://localhost",
			"interval": "10s", // A cada 10 segundos o consul vai verificar se o serviço está ok
			"timeout": "1s"
		}
	}
}
```

- Rodar `consul reload` no container do client
- Logo que subir, irá dar erro de check do agent, isso é normal, pq o nginx por padrão retorna um html com 404
- Alterar o default.conf do nginx para retornar o html correto:
```nginx
# This is a default site configuration which will simply return 404, preventing
# chance access to any other virtualhost.

server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /usr/share/nginx/html;

        # You may need this to prevent return 404 recursion.
        location = /404.html {
                internal;
        }
}
```

- Recarregar o nginx: `nginx -s reload`
- O health check deve parar de reclamar agora
