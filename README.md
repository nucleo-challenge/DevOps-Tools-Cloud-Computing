# Núcleo API - DevOps Tools & Cloud Computing (Sprint 1)

API Java (Spring Boot) do projeto **Núcleo**, app educativo sobre tecnologia nuclear (jogo *Mito ou Verdade*). Nesta sprint, a API roda em um container Docker dentro de uma VM Linux na Microsoft Azure, criada com Azure CLI, e persiste as perguntas no banco de dados da Oracle da FIAP.

## Integrantes

Turma: **2TDSPR**

| Nome | RM |
|------|----|
| Pedro Gonçalves Rodrigues | 567651 |
| Davi Isac Colin Pereira | 567265 |
| Hugo Souza de Jesus | 568542 |
| Matheus Peres Moraes de Jesus | 567300 |
| Cassio Ribeiro | 567295 |

Vídeo da demonstração: [Assistir no YouTube](https://youtu.be/0HzkH-y1eTs)

## Arquitetura

![Arquitetura](docs/arquitetura.png)

Fluxo: o usuário (navegador, cURL ou app) acessa o **IP público da VM** na porta **8080**. O NSG da Azure libera essa porta, o Docker encaminha a requisição para o container da API, e a API lê e grava as perguntas no banco de dados do **Oracle da FIAP** pela porta 1521.

| Componente | Tecnologia |
|------------|------------|
| Nuvem | Microsoft Azure (Grupo de Recursos, VM, VNet, NSG, IP público) |
| Sistema da VM | Ubuntu Server 24.04 LTS |
| Container | Docker (imagem `nucleo-api`, multi-stage, usuário não-root) |
| Aplicação | Java 25 + Spring Boot 4.1.1 (CRUD de `Pergunta` e jogo Mito ou Verdade) |
| Banco | Oracle Database da FIAP (perfil `oracle` da API) |

## Arquivos de DevOps

| Arquivo | Onde roda | O que faz |
|---------|-----------|-----------|
| `Dockerfile` | VM | Receita da imagem da API (build com Maven, execução com JRE 25) |
| `.dockerignore` | VM | Impede que `.env`, `target/` e `data/` entrem na imagem |
| `deploy-infra.sh` | Cloud Shell / Git Bash | Cria Grupo de Recursos, VM e libera a porta 8080 |
| `setup-app.sh` | Dentro da VM | Instala o Docker, gera a imagem e sobe o container |
| `cleanup.sh` | Cloud Shell / Git Bash | Apaga o Grupo de Recursos e comprova a exclusão |

## Como executar

### 1. Criar a infraestrutura (Azure CLI)

```bash
az login
chmod +x deploy-infra.sh
./deploy-infra.sh
```

Comandos principais usados pelo script:

```bash
az group create --name rg-nucleo-sprint1 --location southafricanorth
az vm create --resource-group rg-nucleo-sprint1 --name vm-nucleo-api --image Ubuntu2404 --size Standard_B2als_v2 --authentication-type password --admin-username <RM> --admin-password <SENHA> --vnet-name vnet-nucleo --nsg nsg-nucleo --public-ip-address pip-nucleo --public-ip-sku Standard
az vm open-port --resource-group rg-nucleo-sprint1 --name vm-nucleo-api --port 8080 --priority 1010
```

### 2. Subir a API na VM (Docker)

```bash
ssh <RM>@<IP_PUBLICO_DA_VM>
git clone https://github.com/nucleo-challenge/DevOps-Tools-Cloud-Computing.git
cd DevOps-Tools-Cloud-Computing
cp .env.example .env
nano .env        # DB_URL, DB_USERNAME e DB_PASSWORD do Oracle
chmod +x setup-app.sh
./setup-app.sh
```

Comandos Docker usados pelo script:

```bash
sudo docker build -t nucleo-api .
sudo docker run -d --name nucleo-api -p 8080:8080 --env-file .env --restart unless-stopped nucleo-api
sudo docker ps
sudo docker logs -f nucleo-api
```

As credenciais do banco entram no container pelo `--env-file` em tempo de execução. Elas não ficam no Dockerfile, na imagem nem no GitHub.

### 3. Testar pelo IP público

```bash
# Listar perguntas
curl -X GET http://<IP_PUBLICO_DA_VM>:8080/perguntas

# Cadastrar uma pergunta
curl -X POST http://<IP_PUBLICO_DA_VM>:8080/perguntas \
  -H "Content-Type: application/json" \
  -d '{"enunciado":"A tecnologia nuclear possui aplicacoes na medicina.","resposta":"V","explicacao":"A medicina nuclear usa radiofarmacos em diagnosticos e tratamentos.","referencia":"AMAZUL - Kick-off FIAP - 13OUT26, p. 34"}'

# Sortear uma pergunta para o jogo
curl -X GET http://<IP_PUBLICO_DA_VM>:8080/jogo/perguntas/aleatoria

# Conferir a resposta do jogador
curl -X POST http://<IP_PUBLICO_DA_VM>:8080/jogo/perguntas/1/respostas \
  -H "Content-Type: application/json" \
  -d '{"resposta":"V"}'
```

Swagger: `http://<IP_PUBLICO_DA_VM>:8080/swagger-ui`

| Método | Caminho | Função |
|--------|---------|--------|
| GET | `/` | Índice JSON com os caminhos da API |
| POST | `/perguntas` | Cadastrar |
| GET | `/perguntas` | Listar (texto, page, size, sort) |
| GET | `/perguntas/{id}` | Consultar |
| PUT | `/perguntas/{id}` | Atualizar |
| DELETE | `/perguntas/{id}` | Excluir |
| GET | `/jogo/perguntas/aleatoria` | Sortear pergunta sem gabarito |
| POST | `/jogo/perguntas/{id}/respostas` | Conferir V/F com explicação |

### 4. Apagar os recursos

```bash
chmod +x cleanup.sh
./cleanup.sh
```

## Como Docker e Cloud Computing podem ser utilizados na solução proposta pelo grupo?

<!-- RASCUNHO para o grupo revisar -->

O Núcleo é um app educativo que precisa estar disponível para escolas e estudantes em qualquer lugar, sem depender da máquina de quem desenvolveu. O **Docker** resolve isso empacotando a API Java com tudo o que ela precisa (JRE 25, dependências e configuração) em uma imagem. A mesma imagem roda igual no notebook do desenvolvedor, na homologação e na nuvem, o que elimina o problema do "na minha máquina funciona" e deixa o deploy repetível. O Dockerfile multi-stage gera uma imagem enxuta, a execução com usuário não-root reduz a superfície de ataque, e as credenciais do banco entram só na execução, sem ficar gravadas na imagem.

A **Cloud Computing** (Microsoft Azure) fornece a infraestrutura sob demanda: em poucos minutos o grupo cria uma VM com IP público acessível pela internet, sem comprar servidor. Como tudo é criado por scripts com Azure CLI (Infraestrutura como Código), o ambiente pode ser recriado ou apagado a qualquer momento, e o grupo paga só pelo tempo de uso. Em períodos de maior acesso, como campanhas em escolas, a mesma abordagem permite aumentar o tamanho da VM ou subir novas instâncias do container. Assim, a combinação dos dois deixa a solução portátil, escalável e com custo controlado.
