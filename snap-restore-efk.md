## Snapshot and Restore ElasticSearch on Openshift ##

### Pré-requisitos:

* Elasticsearch instalado no Openshift;
* Servidor de NFS com acesso liberado para o Openshift
* PV criado e apontando para o NFS, /disponível para o pvc do namespace do Elasticsearch.


### 1. Disponibilizar um NFS ReadWriteMany (RWX - The volume can be mounted as read-write by many nodes)

[Configurando um NFS Server no Rhel8](https://access.redhat.com/documentation/pt-br/red_hat_enterprise_linux/8/html/managing_file_systems/nfs-server-configuration_exporting-nfs-shares)

### 2. Configurar PV para nfs dentro do Openshift
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001 
spec:
  capacity:
    storage: 5Gi 
  accessModes:
  - ReadWriteMany 
  nfs: 
    path: /tmp 
    server: 172.17.0.2
```

### 3. Configurar PVC dentro do statefulset do Elasticsearch:
* Configurando uma nova env para montar o volume dentro do pod
```
        - env:
            - name: path.repo
              value: /backup
```
* Criando o volumeMounts apontando para o pv do NFS
```
          volumeMounts:
            - mountPath: /usr/share/elasticsearch/data
              name: es-storage-db-hml
            - mountPath: /backup
              name: es-backup-hml-nfs-volume2
```
* Configurando o volume para vincular ao claim:
```
- name: es-backup-hml-nfs-volume2
          persistentVolumeClaim:
            claimName: es-backup-hml-nfs-claim2
```

### 4. Criando o repo path dentro do Elasticsearch via API:
```
curl -XPUT -H "content-type:application/json" 'http://localhost:9200/snapshot/backup' -d '{"type": "fs","settings":{"location":"my_backup_location"}}'
```

* Esta é a opçao que podemos fazer via cerebro:
```
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "my_backup_location"
  }
}
```
### 5. Realizando o snapshot via API:
* Rodando na minha máquina local
```
curl -XPUT -H "content-type:application/json" 'http://localhost:9200/_snapshot/backup' -d '{"type":"fs","settings":{"location":"/home/rpinto/es-backup","compress":true}}'
```
* Executando o comando no Elasticsearch do Openshift:
```
curl -XPUT -H "content-type:application/json" 'http://localhost:9200/_snapshot/backup' -d '{"type":"fs","settings":{"location":"/backup","compress":true}}'
```
## Para listar o backup:
```
curl -XGET 'http://localhost:9200/_snapshot/_all?pretty'
```
OBS: É possível visualizar o backup via cérebro, também.


## Algums comandos extras:
### Para listar índices e exibir seus tamanhos:
```
curl -XGET 'http://localhost:9200/_cat/indices'
```

### Aguardar até que o comando seja finalizado usando o "wait_for_completion=true":
```
curl -XPUT 'http://localhost:9200/_snapshot/esbackup/first-snapshot?wait_for_completion=true'
```
```
curl -XPUT 'http://localhost:9200/_snapshot/backup/primeiro-backup?wait_for_completion=true'
```

### Simulando um deleção de todos os ínidices:(CUIDADO!)
```
curl -XDELETE 'http://localhost:9200/_all'
```

### Listando os índices:
```
curl -XGET 'http://localhost:9200/_cat/indices'
```

## Restaurando o backup e garantindo que vamos pegar do diretório correto:
```
curl -XPOST 'http://localhost:920/_snapshot/backup/first-snapshot/_restore?wait_for_completion=true'
```

## Listando os índices para garantir que o processo vai ocorrer com sucesso
```
curl -XGET 'http://localhost:9200/_cat/indices'
```

## 6. Configurar Job para executar Script Diariamente:

* script-snap.sh

```
#!/bin/bash
DATE=$(date +%Y-%m-%d-%H-%M-%S)

status_code=$(curl --write-out %{http_code} --silent --output /dev/null -XPUT --insecure https://localhost:9200/_snapshot/backup/backup-data-$DATE?wait_for_completion=true)

if [[ "$status_code" -ne 200 ]] ; then
        exit 1
else
        exit 0
fi
```

# Procedimento de Restore

1. - Parar Kibana
2. - Remover Indices (Cerebro)
3. - Restaurar Snapshot (Cerebro)
4. - Iniciar Kibana

OBS: 
* Subir um elasticsearch zerado, de preferência sem ter o kibana rodando e sem os índices iniciais criados
* Caso o kibana esteja rodando deve-se parar o kibana pois ele fica tentando recriar os índices templates de forma automatica, impossibilitando o restore.

FONTE:
[Snapshot e Restore Elasticsearch Doc](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/modules-snapshots.html)