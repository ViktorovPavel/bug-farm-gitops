# 🚀 Talos Linux Cluster (`bug-farm`)

Инструкция по разворачиванию Kubernetes-кластера из 3 узлов Control Plane на базе **Talos OS** с настройкой точек монтирования для **OpenEBS**.

---

## 📋 Требования

* 3 подготовленные виртуальные машины или bare-metal сервера с загруженной Talos OS.
* Установленные CLI-утилиты: `talosctl` и `kubectl`.
* Сетевые адреса узлов:
  * `192.168.0.100` — Control Plane 1
  * `192.168.0.101` — Control Plane 2
  * `192.168.0.102` — Control Plane 3

---

## 🛠️ Развертывание

### 1. Генерация конфигурации
Генерируем манифесты кластера с эндпоинтом на первом узле:

```bash
talosctl gen config bug-farm [https://192.168.0.100:6443](https://192.168.0.100:6443)
```

### 2. Настройка `talosctl`
Подключаем сгенерированный конфиг и задаем список эндпоинтов:

```bash
export TALOSCONFIG="./talosconfig"
talosctl config endpoint 192.168.0.100 192.168.0.101 192.168.0.102
talosctl config node 192.168.0.100
```

*(Опционально) Интеграция в системный конфиг:*
```bash
talosctl config merge ./talosconfig
```

### 3. Применение конфигурации
Отправляем манифесты на все узлы кластера:

```bash
talosctl apply-config --insecure --nodes 192.168.0.100 --file controlplane.yaml
talosctl apply-config --insecure --nodes 192.168.0.101 --file controlplane.yaml
talosctl apply-config --insecure --nodes 192.168.0.102 --file controlplane.yaml
```

### 4. Инициализация кластера (Bootstrap)
После перезагрузки узлов запускаем первичную инициализацию Kubernetes:

```bash
talosctl bootstrap --nodes 192.168.0.100
```

Проверка состояния установки:
```bash
talosctl health --nodes 192.168.0.100
```

### 5. Настройка `kubectl`
Импортируем `kubeconfig` в локальную систему и переключаем контекст:

```bash
talosctl kubeconfig ~/.kube/config
kubectl config use-context admin@bug-farm
```

### 6. Конфигурация OpenEBS (Extra Mounts)
Применяем патч с дополнительными монтированиями для `kubelet` на всех узлах:

```bash
talosctl patch machineconfig --nodes 192.168.0.100 --patch @talos/openebs-patch.yaml
talosctl patch machineconfig --nodes 192.168.0.101 --patch @talos/openebs-patch.yaml
talosctl patch machineconfig --nodes 192.168.0.102 --patch @talos/openebs-patch.yaml
```

---

## ⚠️ Снятие Taint-ограничений с Control Plane (ВАЖНО)

По умолчанию в Kubernetes на узлы Control Plane накладывается специальный `taint` (`node-role.kubernetes.io/control-plane:NoSchedule`), который запрещает планировщику запускать обычные рабочие поды (Workloads) на управляющих нодах.

Так как данный кластер состоит **только из 3 Control Plane нод** (без выделенных Worker нод), необходимо снять этот `taint`, иначе ваши приложения (включая OpenEBS, базы данных, сервисы) останутся в статусе `Pending` и не смогут запуститься.

Выполните команду:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## ✅ Проверка

Проверяем доступность и готовность всех узлов кластера:

```bash
kubectl get nodes -o wide
```