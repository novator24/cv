# Инструкция по развертыванию VLM системы для администратора Windows

Данная инструкция описывает процесс подготовки среды на Windows (с использованием WSL2) для локального запуска легковесного Kubernetes (k3s), инструмента управления k9s и развертывания системы складской сортировки на базе VLM.

## 1. Подготовка системы (WSL2)

Для полноценной работы Kubernetes на Windows мы будем использовать подсистему Windows для Linux (WSL2).

1. Откройте PowerShell от имени администратора и выполните:
   ```powershell
   wsl --install
   ```
   *(Если WSL уже установлен, убедитесь, что используется версия 2: `wsl --set-default-version 2`)*

2. После перезагрузки компьютера настройте дистрибутив Ubuntu (запустится автоматически).
3. **ВАЖНО для GPU**: Если у вас есть видеокарта NVIDIA, установите актуальные драйверы NVIDIA для Windows. WSL2 автоматически пробросит поддержку GPU внутрь Linux. Устанавливать отдельные Linux-драйверы внутри WSL2 **не нужно**.

## 2. Установка k3s внутри WSL

`k3s` — это сертифицированный легковесный дистрибутив Kubernetes.

1. Откройте терминал Ubuntu (WSL) и выполните команду установки:
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```
2. Убедитесь, что k3s запущен:
   ```bash
   sudo systemctl status k3s
   # или
   sudo k3s kubectl get nodes
   ```
3. Для удобства скопируйте конфигурацию Kubernetes, чтобы использовать её без sudo:
   ```bash
   mkdir -p ~/.kube
   sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
   sudo chown $USER:$USER ~/.kube/config
   ```

## 3. Установка k9s (в Windows или WSL)

`k9s` — это удобный консольный интерфейс для управления кластером Kubernetes.

**Вариант А: Установка в Windows (через Winget или Scoop)**
Откройте PowerShell и выполните:
```powershell
winget install k9s
# ИЛИ через scoop:
scoop install k9s
```
Вам нужно будет скопировать файл `kubeconfig` из WSL в Windows:
```powershell
mkdir $HOME\.kube
wsl cp ~/.kube/config $(wsl wslpath "$(echo $HOME)\.kube\config")
```

**Вариант Б: Установка внутри WSL (рекомендуется)**
В терминале Ubuntu:
```bash
curl -sS https://webinstall.dev/k9s | bash
source ~/.bashrc
```

Теперь просто введите команду `k9s` в терминале, чтобы открыть интерфейс управления кластером.

## 4. Развертывание системы складской сортировки

Все манифесты (или helm-чарты) разворачиваются стандартными средствами Kubernetes.

### Шаг 1: Поддержка GPU (NVIDIA Device Plugin)
Если вы используете GPU для YOLO и VLM, установите плагин внутри WSL:
```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm install --generate-name nvdp/nvidia-device-plugin
```

### Шаг 2: Запуск компонентов решения
Перейдите в директорию с проектом и примените манифесты (пример структуры):
```bash
# Создаем пространство имен
kubectl create namespace warehouse-vlm
kubens warehouse-vlm # если установлен kubectx/kubens, иначе добавляйте -n warehouse-vlm к командам

# 1. Запуск инфраструктуры (Redis/Kafka для обмена сообщениями)
kubectl apply -f infra/

# 2. Запуск YOLO детектора
kubectl apply -f apps/yolo-deployment.yaml

# 3. Запуск VLM (LFM2-VL-3B или аналога)
kubectl apply -f apps/vlm-deployment.yaml

# 4. Запуск сервиса интеграции и синхронизации камер
kubectl apply -f apps/camera-sync.yaml
```

### Шаг 3: Мониторинг в k9s
1. Запустите `k9s`.
2. Нажмите `0`, чтобы показать все namespaces, выберите `warehouse-vlm`.
3. Вы увидите список запущенных подов.
4. Выберите под (например, `vlm-inference-...`) и нажмите `l` (буква L в нижнем регистре), чтобы посмотреть логи в реальном времени. Здесь вы должны увидеть JSON-вывод с детекцией сортировки (например: `{"action": "sort", "item": "box_1"}`).

## 5. Обслуживание и устранение неполадок

- Если под VLM завис в статусе `Pending`, скорее всего, кластеру не хватает ресурсов GPU. В `k9s` нажмите `d` (Describe) на поде, чтобы увидеть причину.
- Для перезапуска компонента: в `k9s` выделите под и нажмите `Ctrl+D` (удалить под, deployment автоматически создаст новый).
- Чтобы остановить кластер k3s (для освобождения ОЗУ): `sudo systemctl stop k3s` (в WSL).
