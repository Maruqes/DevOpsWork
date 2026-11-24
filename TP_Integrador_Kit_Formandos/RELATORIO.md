# Relatorio de evidencias - Trabalho Pratico Integrador 01-06

Nome: Gonçalo Coelho Marques   Data: 24/11/2026
URL do repositorio Git: https://github.com/Maruqes/DevOpsWork

## E01 - Ambiente (1 valor)
Comandos e versoes observadas:

```
$ docker --version
Docker version 29.10.0, build fdd4b8e
$ docker compose version
Docker Compose version 5.8.0
$ git --version
git version 2.56.1
$ docker info
Client: Docker Engine - Community
 Version:    29.10.0
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.39.1
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v5.8.0
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 15
  Running: 9
  Paused: 0
  Stopped: 6
 Images: 12
 Server Version: 29.10.0
 Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 CDI spec directories:
  /etc/cdi
  /var/run/cdi
 Swarm: inactive
 Runtimes: runc io.containerd.runc.v2
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: b7a25261e029d68978bbf290df0a62fc819b3c51
 runc version: v1.5.3-0-g46365c62
 init version: de40ad0
 Security Options:
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 7.3.4-200.fc44.x86_64
 Operating System: Fedora Linux 44 (KDE Plasma Desktop Edition)
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 15.56GiB
 Name: marques
 ID: d3ccccac-379d-43c2-b97c-6e545f39ea91
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
  ::1/128
 Live Restore Enabled: false
 Firewall Backend: iptables+firewalld
  EnableUserlandProxy: true
  UserlandProxyPath: /usr/bin/docker-proxy

```

o docker responde e as versoes estao ok
a api esta no app.py os testes estao em tests/test_app.py o compose esta no compose.yaml e a pipeline esta no Jenkinsfile

## E02 - Dockerfile e container (3 valores)
Tag da imagem; docker ps; HTTP /health; utilizador efetivo do container:

```
$ docker build -t natixis-support:1.0 .
$ docker run -d --name tp-api -p 127.0.0.1:18080:8000 natixis-support:1.0
$ docker ps
CONTAINER ID   IMAGE                 STATUS         PORTS                       NAMES
37b6d6136593   natixis-support:1.0   Up 3 seconds   127.0.0.1:18080->8000/tcp   tp-api
$ curl http://127.0.0.1:18080/health
{"status":"ok","service":"support-api","version":"1.0"}   (HTTP 200)
$ docker logs --tail 15 tp-api
INFO:     172.17.0.1:57436 - "GET /health HTTP/1.1" 200 OK
$ docker exec tp-api id
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
$ docker rm -f tp-api
```

com isto vemos que
tag: natixis-support:1.0
portas: host 127.0.0.1:18080 container 8000
id: uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
log: "GET /health HTTP/1.1" 200 OK
o health deu 200 com status ok e o container nao corre como root

## E03 - Compose, DNS, healthcheck e volume (4 valores)
Resultado de docker compose config; health; teste checker -> api; contador antes/depois da recriacao:

```
$ docker compose config --quiet
(sem erros)
$ docker compose up -d --build
$ docker compose ps
NAME                       IMAGE                   STATUS                    PORTS
natixis-integrador-api-1   natixis-support:local   Up 10 seconds (healthy)   127.0.0.1:18080->8000/tcp
$ docker compose --profile diagnostico run --rm checker -fsS http://api:8000/health
{"status":"ok","service":"support-api-integrador","version":"1.0"}
$ curl http://127.0.0.1:18080/visits
{"visits":0}
$ curl -X POST http://127.0.0.1:18080/visits
{"visits":1}
$ curl http://127.0.0.1:18080/visits
{"visits":1}
$ docker compose up -d --force-recreate
$ docker compose ps
natixis-integrador-api-1   natixis-support:local   Up 10 seconds (healthy)   127.0.0.1:18080->8000/tcp
$ curl http://127.0.0.1:18080/visits
{"visits":1}
```

com isto vemos que
config: o compose valida sem erros
health: a api fica healthy
dns: o checker chega a api pelo nome api e recebe status ok
contador: antes 1 depois de recriar 1
o contador nao se perdeu porque fica guardado no volume app-data que nao e apagado quando o container e recriado

## E04 - Incidente Docker (3 valores)
Sintoma, comandos de investigacao, causa-raiz, alteracao realizada, prova da recuperacao e contador preservado:

```
$ docker compose -f compose.yaml -f incidente.yaml up -d --force-recreate
$ docker compose -f compose.yaml -f incidente.yaml ps
natixis-integrador-api-1   Up 43 seconds (unhealthy)   127.0.0.1:18080->8000/tcp
$ docker compose -f compose.yaml -f incidente.yaml logs --tail 30 api
api-1  | INFO:     Uvicorn running on http://0.0.0.0:8001 (Press CTRL+C to quit)
$ docker compose -f compose.yaml -f incidente.yaml config
    aparece o file compose contruido
$ curl http://127.0.0.1:18080/health
curl: (56) Recv failure: Connection reset by peer
$ docker compose -f compose.yaml -f incidente.yaml exec api python -c "print(open('/proc/net/tcp').read())"
00000000:1F41 ... 0A      (1F41 = porta 8001 em LISTEN)

$ docker compose up -d --force-recreate api
$ docker compose ps
natixis-integrador-api-1   Up 10 seconds (healthy)   127.0.0.1:18080->8000/tcp
$ curl http://127.0.0.1:18080/health
{"status":"ok","service":"support-api-integrador","version":"1.0"}
$ curl http://127.0.0.1:18080/visits
{"visits":3}
```

runbook
sintoma: a api fica unhealthy e o curl ao /health no 18080 da connection reset
diagnostico: compose ps logs config e ver as portas em escuta dentro do container
causa: o incidente.yaml muda o command para --port 8001 mas o mapeamento e o healthcheck usam a 8000
correcao: voltar a configuracao base sem o incidente.yaml mantendo volume :D
validacao: a api volta a healthy o /health da status ok e o contador continua com o valor do violume

## E05 - Git (3 valores)
Branch, hashes de commits, merge e tag v1.0.0:

```
$ git add .
$ git commit -m "feat: ambiente integrador funcional"
$ git switch -c fix/health-info
$ git add app.py tests/test_app.py
$ git commit -m "fix: acrescentar identificacao no health"
$ python -m pytest -q
2 passed
$ git switch main
$ git merge fix/health-info
Fast-forward
$ git tag -a v1.0.0 -m "Release integrador 1.0.0"
$ git log --oneline --graph --decorate -6
* 47207f3 (HEAD -> main, tag: v1.0.0, fix/health-info) fix: acrescentar identificacao no health
* 8b71473 feat: ambiente integrador funcional
* 6f62560 (origin/main, origin/HEAD) Initial commit
$ git push -u origin main
$ git push origin v1.0.0
```

com isto vemos que
branch: fix/health-info
commits: 8b71473 (app funcional) e 47207f3 (campo team no health)
merge: fix/health-info entrou na main em fast-forward
tag: v1.0.0 anotada no commit 47207f3
o .gitignore deixa de fora .venv .env e caches e os testes passam com o campo team

## E06 - Jenkins SUCCESS (4 valores)
URL/numero do build, etapas e identificador da imagem construida:

```
job: natixis-integrador (pipeline from SCM https://github.com/Maruqes/DevOpsWork.git main, script TP_Integrador_Kit_Formandos/Jenkinsfile)
build: #2 http://127.0.0.1:8080/job/natixis-integrador/2/
[Pipeline] { (Checkout)
+ .venv/bin/python -m pytest -q
2 passed, 1 warning in 0.28s
+ docker build -t natixis-support:ci-2 .
Successfully tagged natixis-support:ci-2
+ docker image inspect natixis-support:ci-2 --format {{.Id}}
sha256:caafab374708b6d83a3020dc58627ff27760bc2f7bed532d7a7e979753f4b4c0
Finished: SUCCESS
```

com isto vemos que
build: #2 com SUCCESS
etapas: Checkout Tests e Docker Build todas ok no agente ci-python-docker
imagem: natixis-support:ci-2 com id sha256:caafab374708b6d83a3020dc58627ff27760bc2f7bed532d7a7e979753f4b4c0
o build #1 falhou so porque o jenkins nao chegava ao github e depois de resolver a rede o #2 passou

## E07 - Jenkins FAILURE e recuperacao (2 valores)
Build FAILURE, etapa que falhou e excerto relevante do log; build SUCCESS apos correcao:

```
commit ca15830 test: falha controlada status offline
build: #3 http://127.0.0.1:8080/job/natixis-integrador/3/
>       assert response.json()["status"] == "offline"
E       AssertionError: assert 'ok' == 'offline'
FAILED tests/test_app.py::test_health - AssertionError: assert 'ok' == 'offline'
1 failed, 1 passed, 1 warning in 0.29s
Stage "Docker Build" skipped due to earlier failure(s)
Finished: FAILURE

commit 3a4c894 test: repor assercao status ok
build: #4 http://127.0.0.1:8080/job/natixis-integrador/4/
2 passed, 1 warning in 0.28s
sha256:caafab374708b6d83a3020dc58627ff27760bc2f7bed532d7a7e979753f4b4c0
Finished: SUCCESS
```

com isto vemos que
failure: build #3 falhou na etapa Tests porque o teste esperava offline e a api devolve ok
docker build: nao correu foi skipped por causa da falha
recuperacao: build #4 com a assercao reposta deu SUCCESS

## Verificacao final
[x] Dockerfile [x] compose.yaml [x] app.py [x] tests/ [x] Jenkinsfile
[x] Historico Git [x] Tag v1.0.0 [x] Evidencias E01-E07
