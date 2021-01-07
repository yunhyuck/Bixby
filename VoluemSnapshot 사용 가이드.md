# VolumeSnapshot 사용 가이드

## 유의사항

- 본 가이드는 HyperCloud 3.8 버전 기준으로 작성 되었습니다.
  - K8s version: v1.15.3
  - Rook version: v1.2.7
    - Hypercloud-rook-ceph-installer tag v1.1.7 로 설치됨
  - Ceph version: v14.2.8
  - CDI version: v1.11.0
- 본 가이드에 사용하는 volumeSnapshot api version은 v1alpha1 으로 K8s version v1.17 부터 제공되는 v1beta1 버전과 **호환 되지 않습니다.**
  - v1alpha1 으로 사용중인 snapshot 데이터는 HyperCloud 버전을 업그레이드 하는 경우 유실됨을 알려드립니다.
- 해당 버전에서는 RBD, **block storage의 snapshot 기능만** 제공됨을 알려드립니다.
  - file storage의 snapshot 기능 사용을 위해서는 HyperCloud 버전 업그레이드가 필요합니다.
- K8s에서 정의하는 VoluemSnapshot은 새로운 volume을 만들 때, snapshot으로부터 데이터를 복구하여 기존의 데이터가 채워진 볼륨 생성하는 용도로 사용됩니다. 
  - 롤백, 즉 현재 볼륨 상태를 snapshot을 활용하여 이전상태로 돌리는 용도로 설계되지 않았습니다.

## 사전 작업

### 1. apiserver feature gate 켜기

- k8s v1.15에서는 snapshot 기능이 alpha 버전으로 제공되기 때문에 feature gate를 별도로 켜주셔야 합니다.
- `/etc/kubernetes/manifests/kube-apiserver.yaml` 중 spec > containers > command 란에 `- --feature-gates=VolumeSnapshotDataSource=true` 를 추가합니다.
- apiserver 가 약 1~2분 내로 재기동 됨을 확인합니다.

### 2. volumeSnapshotClass 생성

- 아래 yaml를 참고하여 volumeSnapshotClass를 생성합니다.

``` shell
$ kubectl apply -f snapshotclass.yaml
```

``` yaml
# snapshotclass.yaml
---
apiVersion: snapshot.storage.k8s.io/v1alpha1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
snapshotter: rook-ceph.rbd.csi.ceph.com
parameters:
  # Specify a string that identifies your cluster. Ceph CSI supports any
  # unique string. When Ceph CSI is deployed by Rook use the Rook namespace,
  # for example "rook-ceph".
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
```

- 생성된 volumeSnapshotClass 확인

``` shell
$ kubectl get volumesnapshotclass
NAME                      AGE
csi-rbdplugin-snapclass   128m
```

## volumeSnapshot 사용 가이드

### 1. volumeSnapshot 생성

- 아래 yaml를 참고하여 volumeSnapshot를 생성합니다.

``` shell
$ kubectl apply -f snapshot.yaml
```

``` yaml
# snapshot.yaml
---
apiVersion: snapshot.storage.k8s.io/v1alpha1
kind: VolumeSnapshot
metadata:
  name: rbd-pvc-snapshot
spec:
  snapshotClassName: csi-rbdplugin-snapclass
  source:
    name: rbd-pvc
    kind: PersistentVolumeClaim
```

- 생성한 volumesnapshot 확인

``` shell
$ kubectl get volumesnapshot
NAME                                 AGE
rbd-pvc-snapshot   42m
```

- 생성한 volumesnapshot 삭제

``` shell
$ kubectl delete volumesnapshot rbd-pvc-snapshot
```

### 2. volumeSnapshot 으로부터 pvc restore

- 아래 yaml를 참고하여 volumeSnapshot으로 부터 pvc를 복구 할 수 있습니다.

``` shell
$ kubectl apply -f pvc-restore.yaml
```

``` yaml
# pvc-restore.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc-restore
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: rbd-pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- 생성된 pvc 확인

``` shell
$ kubectl get pvc
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
rbd-pvc          Bound    pvc-318c6686-462d-4dc7-8eb9-da9e6cf15180   5Gi        RWO            rook-ceph-block   55m
rbd-pvc-restore   Bound    pvc-49ec3276-7934-4ad2-a77c-310b06b113ba   5Gi        RWO            rook-ceph-block   42m
```
