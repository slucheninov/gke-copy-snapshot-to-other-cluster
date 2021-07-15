# Как перенести PVC диск из одного проэкта в другой или из одного кластера в другой

Данный метод применим для GKE, google cloud.
Копировать будем с использованием снапшотов, снапшоты снимаются с при моунчинных дисков.
Имеем два прокэкта P1 и P2.
В проекте P1 выясняем название диска.

```bash
#kubectl get pvc -n test
NAME                                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-kafka-0                                   Bound    pvc-474ae119-992f-43ea-b680-2e87657addb0   100Gi      RWO            standard       28h
#
#kubectl describe pvc data-kafka-0  -n test
Name:          data-kafka-0
Namespace:     test
StorageClass:  standard
Status:        Bound
Volume:        pvc-474ae119-992f-43ea-b680-2e87657addb0
Labels:        app.kubernetes.io/component=kafka
               app.kubernetes.io/instance=test
               app.kubernetes.io/name=kafka
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/gce-pd
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      100Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       kafka-0
Events:        <none>
````

Здесь наш интересует  Volume: pvc-474ae119-992f-43ea-b680-2e87657addb0

```bash
#kubectl describe pv pvc-474ae119-992f-43ea-b680-2e87657addb0  -n test
Name:              pvc-474ae119-992f-43ea-b680-2e87657addb0
Labels:            failure-domain.beta.kubernetes.io/region=us-central1
                   failure-domain.beta.kubernetes.io/zone=us-central1-b
Annotations:       kubernetes.io/createdby: gce-pd-dynamic-provisioner
                   pv.kubernetes.io/bound-by-controller: yes
                   pv.kubernetes.io/provisioned-by: kubernetes.io/gce-pd
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      standard
Status:            Bound
Claim:             test/data-kafka-0
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          100Gi
Node Affinity:
  Required Terms:
    Term 0:        failure-domain.beta.kubernetes.io/zone in [us-central1-b]
                   failure-domain.beta.kubernetes.io/region in [us-central1]
Message:
Source:
    Type:       GCEPersistentDisk (a Persistent Disk resource in Google Compute Engine)
    PDName:     gke-test-02-7949b14e-pvc-474ae119-992f-43ea-b680-2e87657addb0
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
Events:         <none>
```

Теперь мы видим как называется наш диск  PDName: gke-test-02-7949b14e-pvc-474ae119-992f-43ea-b680-2e87657addb0
Создаем с него снапшот, название снапшота будет соответствовать нашему  volume: pvc-46e222f4-2cfa-470c-ac7f-fe3bb52ac4ed который мы берем из проекта в который мы будем копировать диск, так же само 
смотрим PDName. В проекте P2 уже должен быть создан PVC с аналогичным диском, но не подключен ни к одному из контейнеров.

```bash
# gcloud --project P1 compute  disks snapshot gke-test-02-7949b14e-pvc-474ae119-992f-43ea-b680-2e87657addb0 --zone=us-central1-a --snapshot-names=pvc-474ae119-992f-43ea-b680-2e87657addb0

```

Снапшот готов.
Теперь создаем диск из снапшота, так же нужно скопировать дескрипшен и лейбелс, чтоб кубернетес мог увидеть нужный диск.

```bash
# gcloud --project P2 compute disks create pvc-474ae119-992f-43ea-b680-2e87657addb0 --source-snapshot https://www.googleapis.com/compute/v1/projects/test-57e40f6d/global/snapshots/pvc-474ae119-992f-43ea-b680-2e87657addb0 \
--zone=us-central1-b \
--description="{\"kubernetes.io/created-for/pv/name\":\"pvc-474ae119-992f-43ea-b680-2e87657addb0\",\"kubernetes.io/created-for/pvc/name\":\"data-kafka-0\",\"kubernetes.io/created-for/pvc/namespace\":\"test\"}" \
--labels goog-gke-volume=
```

Все готово
