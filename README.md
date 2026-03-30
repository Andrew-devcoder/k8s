# Kubernetes Repo for Domains and Projects

Це центральний `k8s` repo не для одного застосунку, а для ваших доменів і майбутніх проектів.

Тобто логіка тепер така:

- кожен новий сайт або API = окремий каталог у `projects/`
- у ньому лежать `Namespace`, `Deployment`, `Service`
- кожен проект може мати свій `Ingress`, який вирішує, який хост іде в який сервіс

`apps/frontend` і `apps/backend` були моїм неправильним припущенням про структуру. Я це прибрав.

## Поточна структура

```text
projects/
  main-site/
  resume-site/
  kraftplan-web/
  kraftplan-api/
  b2b/
  iren/
infra/
  cert-manager/
  ingress/
clusters/
  production/
```

## Уже підготовлені домени

Зараз у repo закладені такі хости:

- `andrii-kovpak.dev`
- `cv.andrii-kovpak.dev`
- `resume.andrii-kovpak.dev`
- `kraftplan.andrii-kovpak.dev`
- `api.kraftplan.andrii-kovpak.dev`
- `b2b.andrii-kovpak.dev`
- `iren.andrii-kovpak.dev`

Відповідність проектів:

- `projects/main-site` -> `andrii-kovpak.dev`
- `projects/resume-site` -> `cv.andrii-kovpak.dev`, `resume.andrii-kovpak.dev`
- `projects/kraftplan-web` -> `kraftplan.andrii-kovpak.dev`
- `projects/kraftplan-api` -> `api.kraftplan.andrii-kovpak.dev`
- `projects/b2b` -> `b2b.andrii-kovpak.dev`
- `projects/iren` -> `iren.andrii-kovpak.dev`

## SSL і сабдомени

У repo додана базова схема для автоматичного SSL:

- `cert-manager` випускає і продовжує сертифікати Let's Encrypt
- `ClusterIssuer` лежить у [infra/cert-manager/clusterissuer-letsencrypt-prod.yaml](/Users/admin/Projects/k8s/infra/cert-manager/clusterissuer-letsencrypt-prod.yaml)
- для безпечної перевірки є ще [infra/cert-manager/clusterissuer-letsencrypt-staging.yaml](/Users/admin/Projects/k8s/infra/cert-manager/clusterissuer-letsencrypt-staging.yaml)
- проект `iren` має свій `Ingress` у [projects/iren/ingress.yaml](/Users/admin/Projects/k8s/projects/iren/ingress.yaml)

Для `iren.andrii-kovpak.dev` сертифікат буде окремим. Це нормально і саме так зазвичай робиться для сабдоменів.

Сертифікат на `andrii-kovpak.dev` не покриває `iren.andrii-kovpak.dev`, якщо це не wildcard сертифікат `*.andrii-kovpak.dev`.

## DNS

На вашому скріні проблема правильна: `A`-запис не може вказувати на `*.andrii-kovpak.dev`.

Правильно так:

- `A` запис: ім'я `@`, значення `IP` вашого ingress/load balancer
- `A` запис: ім'я `*`, значення `IP` вашого ingress/load balancer
- або `CNAME` запис: ім'я `*`, значення `andrii-kovpak.dev` чи hostname балансера, якщо ваш DNS-провайдер це підтримує

Тобто wildcard DNS можна зробити, але:

- для `A` запису значенням має бути саме IPv4 адреса
- для `CNAME` значенням має бути доменне ім'я

Приклад:

```text
Type: A
Name: @
Value: 203.0.113.10

Type: A
Name: *
Value: 203.0.113.10
```

## Як додавати новий проект

1. Створюєте каталог `projects/<project-name>/`
2. Додаєте `namespace.yaml`, `deployment.yaml`, `service.yaml`, `ingress.yaml`, `kustomization.yaml`
3. У `ingress.yaml` ставите потрібний `host` і `tls.secretName`
4. DNS для цього хоста має вказувати на ingress/load balancer
5. Додаєте project у [clusters/production/kustomization.yaml](/Users/admin/Projects/k8s/clusters/production/kustomization.yaml)

Мінімальний приклад:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
  rules:
    - host: app.example.com
```

Після цього `cert-manager` сам створить і буде поновлювати сертифікат.

Спочатку зручно тестувати нові хости через `letsencrypt-staging`, а після успішної перевірки міняти анотацію на `letsencrypt-prod`.

## GHCR простими словами

`ghcr.io` це GitHub Container Registry.

Це реєстр Docker-образів від GitHub. Туди GitHub Actions може пушити зібрані image.

Приклад:

- код лежить у GitHub repo `andrii-kovpak/resume-site`
- GitHub Actions збирає image
- image пушиться як `ghcr.io/andrii-kovpak/resume-site:2026-03-27-1`
- Kubernetes на сервері тягне саме цей image

Тобто:

- GitHub = де лежить код
- GHCR = де лежить готовий контейнерний образ
- Kubernetes = де цей образ запускається

## Як GHCR працює у вашій схемі

Потік такий:

1. Ви пушите код у repo проекту.
2. GitHub Actions збирає Docker image.
3. Actions пушить його в `ghcr.io`.
4. Потім ви вручну оновлюєте image в цьому `k8s` repo.
5. Кластер застосовує оновлений manifest.
6. Deployment робить rollout.

Важливо:

Kubernetes сам не "бачить", що в GHCR з'явився новий image. Йому треба або новий tag/digest у manifest, або окремий ручний deploy-крок.

## Чому не варто покладатися на `latest`

`latest` поганий для production, бо:

- не видно, яка саме версія зараз у проді
- rollback стає мутним
- складно відтворити конкретний реліз

Краще так:

- `ghcr.io/andrii-kovpak/resume-site:2026-03-27-1`
- або ще краще:
- `ghcr.io/andrii-kovpak/resume-site@sha256:...`

## Секрет для pull з GHCR

Якщо package приватний, Kubernetes має авторизуватися в GHCR.

Приклад для одного namespace:

```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --namespace <project-namespace> \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_TOKEN
```

Так само треба створити цей секрет і в інших namespace, де буде deploy.

## Apply

Проекти:

```bash
kubectl apply -k clusters/production
```

TLS і проекти:

```bash
kubectl apply -k infra/cert-manager
kubectl apply -k clusters/production
```

Перевірка:

```bash
kubectl get ingress -A
kubectl get clusterissuer
kubectl get certificate -A
kubectl describe ingress iren -n iren
kubectl describe certificate iren-andrii-kovpak-dev-tls -n iren
```

## Rollback

Приклад:

```bash
kubectl rollout history deployment/<deployment-name> -n <project-namespace>
kubectl rollout undo deployment/<deployment-name> -n <project-namespace>
```

## Рекомендований production flow

1. Кожен проект має свій source repo.
2. Source repo збирає image і пушить у `ghcr.io`.
3. Ви вручну оновлюєте image у цьому `k8s` repo.
4. Ви вручну застосовуєте manifests на сервері.
5. Kubernetes робить rollout.
