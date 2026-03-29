# Kubernetes Repo for Domains and Projects

Це центральний `k8s` repo не для одного застосунку, а для ваших доменів і майбутніх проектів.

Тобто логіка тепер така:

- кожен новий сайт або API = окремий каталог у `projects/`
- у ньому лежать `Namespace`, `Deployment`, `Service`
- окремо є один `Ingress`, який вирішує, який хост іде в який сервіс

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

## Один ingress на все

Ваш задум правильний: DNS ви не чіпаєте тут, а контролюєте тільки `Ingress rules`.

Для цього є:

- [infra/ingress/namespace.yaml](/Users/admin/Projects/k8s/infra/ingress/namespace.yaml)

Технічний нюанс Kubernetes:

`Ingress` не може напряму ходити в `Service` іншого namespace.

Тому схема тут така:

1. Є namespace `edge`.
2. У `edge` живе один `Ingress`.
3. Там же є bridge services.
4. Вони перенаправляють на сервіси конкретних проектів у їхніх namespace.

Конкретні manifests для `Ingress` і bridge services краще додавати тільки тоді, коли проект уже готовий до зовнішнього доступу.

## Як додавати новий проект

1. Створюєте каталог `projects/<project-name>/`
2. Додаєте `namespace.yaml`, `deployment.yaml`, `service.yaml`, `kustomization.yaml`
3. Коли проект готовий до зовнішнього доступу, додаєте bridge service в `infra/ingress/`
4. Додаєте host rule для потрібного домену або сабдомену в `infra/ingress/`
5. Додаєте project у [clusters/production/kustomization.yaml](/Users/admin/Projects/k8s/clusters/production/kustomization.yaml)

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

Ingress:

```bash
kubectl apply -f infra/ingress/namespace.yaml
kubectl apply -f infra/ingress/<service-bridges>.yaml
kubectl apply -f infra/ingress/<ingress>.yaml
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
