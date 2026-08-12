# 🔐 Sealed Secrets Management Guide

Инструкция по безопасной генерации и шифрованию секретов для Kubernetes с использованием `kubeseal`.

Файлы `SealedSecret` содержат асимметрично зашифрованные данные и **безопасны для хранения в публичных и приватных Git-репозиториях**.

---

## 🚀 Способ 1: Однострочник (Recommended)

Самый безопасный способ, так как незашифрованный секрет **не сохраняется на диск**.

### Создание секрета из строки (literal):
```bash
kubectl create secret generic cloudflared-secret \
  --namespace=default \
  --from-literal=TUNNEL_TOKEN='your_token_here' \
  --dry-run=client -o yaml | \
  kubeseal --controller-name=sealed-secrets --controller-namespace=kube-system --format yaml > cloudflared-sealed-secret.yaml

```

### Создание секрета из файла (например, `.json` или `.env`):

```bash
kubectl create secret generic app-config \
  --namespace=default \
  --from-file=credentials.json=./path/to/credentials.json \
  --dry-run=client -o yaml | \
  kubeseal --controller-name=sealed-secrets --controller-namespace=kube-system --format yaml > app-config-sealed-secret.yaml

```

---

## 🛠 Способ 2: Через временный файл (Двухэтапный)

Если нужно сначала проверить сгенерированный Kubernetes-манифест.

1. **Генерируем сырой YAML:**
```bash
kubectl create secret generic cloudflared-secret \
  --namespace=default \
  --from-literal=TUNNEL_TOKEN='your_token_here' \
  --dry-run=client -o yaml > cloudflared-secret-raw.yaml

```


2. **Запечатываем через kubeseal:**
```bash
kubeseal --controller-name=sealed-secrets --controller-namespace=kube-system \
  -f cloudflared-secret-raw.yaml \
  -o yaml > cloudflared-sealed-secret.yaml

```


3. **КРИТИЧНО: Удаляем незашифрованный файл!**
```bash
rm cloudflared-secret-raw.yaml

```



---

## ✈️ Способ 3: Офлайн-режим (Без доступа к кластеру)

Полезно для CI/CD или если кластер за файрволом/VPN.

1. **Скачиваем публичный сертификат (выполняется 1 раз при наличии доступа к K8s):**
```bash
kubeseal --controller-name=sealed-secrets --controller-namespace=kube-system --fetch-cert > pub-sealed-secrets.pem

```


*(Файл `pub-sealed-secrets.pem` содержит только открытый ключ, его можно закоммитить в репозиторий).*
2. **Шифруем локально в любом месте:**
```bash
kubectl create secret generic cloudflared-secret \
  --namespace=default \
  --from-literal=TUNNEL_TOKEN='your_token_here' \
  --dry-run=client -o yaml | \
  kubeseal --cert pub-sealed-secrets.pem --format yaml > cloudflared-sealed-secret.yaml

```



---

## 💡 Области действия (Scopes)

По умолчанию `kubeseal` привязывает секрет **к имени секрета и namespace**. Если переименовать файл или применить его в другой namespace — контроллер не сможет его расшифровать.

Если нужно ослабить ограничение:

* **Использовать в рамках всего Namespace (имя секрета можно менять):**
Добавь флаг `--scope namespace-wide`
* **Использовать в любом Namespace кластера:**
Добавь флаг `--scope cluster-wide`

**Пример c ограничением namespace-wide:**

```bash
kubeseal --scope namespace-wide --controller-name=sealed-secrets --controller-namespace=kube-system -f raw.yaml -o yaml > sealed.yaml

```

