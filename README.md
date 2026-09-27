                    Docker Host
                         │
                  ┌──────▼──────┐
                  │   AI_local  │
                  │   network   │
                  └──────┬──────┘
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
│  OpenCode   │   │ FreeLLMAPI  │   │   Ollama    │
│             │──►│    :3001    │   │   :11434    │
└─────────────┘   └─────────────┘   └─────────────┘
                                         │
                                    ┌────▼────┐
                                    │  Model  │
                                    │ Qwen /  │
                                    │ DeepSeek│
                                    └─────────┘

1. Criar a rede
docker network create AI_local

Se já existir, não precisa recriar.

docker network ls | grep AI_local
2. Criar o volume do Ollama
docker volume create ollama_data

3. Subir o Ollama

3.1 Criar volume do Ollama
docker volume create ollama_data

Para CPU:

docker run -d \
  --name ollama \
  --restart unless-stopped \
  --network AI_local \
  --gpus all \
  -p 11434:11434 \
  -v ollama_data:/root/.ollama \
  ollama/ollama

4. Verificar
docker ps

E:

curl http://localhost:11434/api/tags
5. Baixar um modelo

Por exemplo, Qwen:

docker exec -it ollama ollama pull qwen3

Ou, se quiser usar DeepSeek:

docker exec -it ollama ollama pull deepseek-r1

Verifique:

docker exec -it ollama ollama list
6. Testar diretamente
curl http://localhost:11434/api/generate \
  -d '{
    "model": "qwen3",
    "prompt": "Olá, teste do Ollama",
    "stream": false
  }'
7. Do OpenCode para Ollama

Como todos estão na AI_local, o OpenCode pode acessar:

http://ollama:11434

e não:

http://localhost:11434

Dentro do container OpenCode:

curl http://ollama:11434/api/tags
Stack final
AI_local
│
├── opencode
│
├── freellmapi
│   └── :3001
│
└── ollama
    └── :11434
        ├── qwen3
        └── deepseek-r1

Importante: se sua máquina tiver GPU NVIDIA, eu recomendo subir o Ollama com acesso à GPU em vez da configuração CPU acima. Nesse caso, 
posso montar o docker-compose.yml completo com OpenCode + FreeLLMAPI + Ollama + GPU, todos na AI_local.

#################################################
NVIDIA Container Toolkit não está instalado. Seu driver está OK; falta apenas a camada que permite ao Docker usar a GPU.

Como você está em Ubuntu, faça assim.

1. Adicionar o repositório NVIDIA
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
2. Atualizar o APT
sudo apt update
3. Instalar
sudo apt install -y nvidia-container-toolkit
4. Confirmar
nvidia-ctk --version

Deve retornar algo parecido com:

NVIDIA Container Toolkit CLI version ...
5. Configurar o Docker
sudo nvidia-ctk runtime configure --runtime=docker

Depois:

sudo systemctl restart docker
6. Testar a GPU no Docker

Agora repita:

docker run --rm --gpus all \
  nvidia/cuda:12.8.1-base-ubuntu24.04 \
  nvidia-smi

Você deve finalmente ver:

GeForce MX450
7. Depois seguimos para o Ollama

Quando esse teste funcionar, vamos subir:

AI_local
├── opencode
├── freellmapi
└── ollama
    └── NVIDIA MX450

E o Ollama ficará acessível internamente pelo OpenCode/FreeLLMAPI através de:

http://ollama:11434

Faça primeiro os passos 1–6. Se algum comando der erro, pare nele e me envie a saída; principalmente se ocorrer no apt update ou nvidia-ctk runtime configure.

###################################
Instalação em docker-compose

1. Criar a rede

Antes de subir:

docker network create AI_local 2>/dev/null || true

Verifique:

docker network inspect AI_local
2. Subir o Ollama

Como o opencode nesse compose é um container interativo, eu começaria subindo:

docker compose up -d ollama

Verifique:

docker compose ps
3. Testar GPU
docker exec ollama nvidia-smi

Deve aparecer:

NVIDIA GeForce MX450
4. Baixar o modelo
docker exec -it ollama ollama pull qwen3:0.6b

Teste:

docker exec -it ollama ollama run qwen3:0.6b
5. Testar a API

No host:

curl http://localhost:11434/api/tags

Dentro do OpenCode:

curl http://ollama:11434/api/tags

A diferença é importante:

HOST
└── localhost:11434

CONTAINER
└── ollama:11434
FreeLLMAPI

Como seu FreeLLMAPI atual já está rodando em outro Compose, basta conectá-lo à AI_local.

No docker-compose.yml do FreeLLMAPI, adicione:

networks:
  AI_local:
    external: true
    name: AI_local

E no serviço freellmapi:

services:

  freellmapi:
    # configuração existente...

    networks:
      - AI_local

networks:
  AI_local:
    external: true
    name: AI_local

Depois:

docker-compose up -d

Agora teremos:

                         AI_local
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
   ┌──────────┐       ┌────────────┐      ┌──────────┐
   │ OpenCode │──────►│ FreeLLMAPI │      │  Ollama  │
   │          │       │   :3001    │      │  :11434  │
   └────┬─────┘       └────────────┘      └────┬─────┘
        │                                       │
        │                                       ▼
        │                                NVIDIA MX450
        │                                  2 GB VRAM
        │
        └──────────────────► ollama:11434

##################################

ENCRYPTION_KEY no ambiente do seu FreeLLMAPI, você pode pegar a chave diretamente do container atual.

1. Descobrir o nome do container
docker ps --format "table {{.Names}}\t{{.Image}}"

Se aparecer freellmapi, execute:

docker exec freellmapi printenv ENCRYPTION_KEY

Isso deve retornar a chave atual.

2. Se não aparecer

Verifique todas as variáveis:

docker inspect freellmapi \
  --format '{{range .Config.Env}}{{println .}}{{end}}' | grep ENCRYPTION_KEY
3. Se o FreeLLMAPI estiver no Compose atual

Também pode verificar:

docker-compose exec freellmapi env | grep ENCRYPTION_KEY

ou, se estiver usando o novo comando:

docker compose exec freellmapi env | grep ENCRYPTION_KEY
4. Importante

Não gere uma nova chave se você quer migrar o FreeLLMAPI existente.

A chave precisa permanecer a mesma para que os dados criptografados existentes continuem acessíveis.

Depois de obter:

ENCRYPTION_KEY=...

coloque somente a chave no .env:

ENCRYPTION_KEY=...

E proteja o arquivo:

chmod 600 .env

Se quiser confirmar antes da migração, rode:

docker exec freellmapi printenv ENCRYPTION_KEY

e também:

docker inspect freellmapi --format '{{range .Mounts}}{{println .Source " -> " .Destination}}{{end}}'

Esse segundo comando é importante porque nos permite identificar o banco/volume atual do FreeLLMAPI antes de trocar para o novo Compose.

######################
docker-compose config
docker-compose up -d
docker-compose ps
docker-compose logs -f freellmapi
docker-compose logs -f ollama
docker-compose logs -f

7. Testar Ollama
docker exec -it ollama ollama list

Se ainda não tiver modelo:

docker exec -it ollama ollama pull qwen3:0.6b

Depois:

docker exec -it ollama ollama run qwen3:0.6b
8. Testar GPU dentro do Ollama
docker exec ollama nvidia-smi
E:

docker exec ollama ollama list
9. Testar comunicação pela AI_local

FreeLLMAPI:

docker run --rm --network AI_local curlimages/curl \
  http://freellmapi:3001/v1/models

Ollama:

docker run --rm --network AI_local curlimages/curl \
  http://ollama:11434/api/tags

Antes de executar docker-compose up -d, eu recomendo validar os volumes do seu FreeLLMAPI atual, porque você já tem uma instalação existente e não queremos subir uma nova instância e perder o banco/configuração atual.

Execute:

docker inspect freellmapi \
  --format '{{range .Mounts}}{{println .Source " -> " .Destination}}{{end}}'
e:

docker volume ls | grep -i freellmapi

#####################

FreeLLMAPI informando que, para criar a primeira conta a partir de outro computador, você precisa do setup code gerado pelo servidor.

Como estamos rodando via Docker Compose, pegue o código nos logs:

docker-compose logs freellmapi

Para mostrar somente linhas relacionadas ao código:

docker-compose logs freellmapi | grep -iE 'setup|code|first account'

Ou, se o container já estiver rodando:

docker logs freellmapi 2>&1 | grep -iE 'setup|code|first account'

Você deve encontrar algo semelhante a:

Setup code: XXXXX-XXXXX
Se não aparecer

Acompanhe a inicialização em tempo real:

docker-compose logs -f freellmapi

Se precisar reiniciar para gerar/ver o código:

docker-compose restart freellmapi

Depois:

docker-compose logs freellmapi | grep -iE 'setup|code'
Alternativa

Como você está expondo:

ports:
  - "3001:3001"

no próprio computador onde o FreeLLMAPI está rodando, abra:

http://localhost:3001

A própria interface deverá permitir a criação da primeira conta.

Importante: o setup code é uma credencial de bootstrap. Não envie o código aqui se não for necessário; use-o diretamente na tela de configuração.