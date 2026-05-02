# 監視・通知基盤の構築まとめ

## 概要
k3sクラスターとは独立した専用EC2インスタンス（t3.small）にDocker Composeで監視スタックを構築。PodがCrashLoopBackOffになったらSlackに通知する。

## 構成
```
monitoringインスタンス（EC2 t3.small）
├── Prometheus   ← kube-state-metricsからメトリクスを収集
├── Alertmanager ← アラートをSlackに通知
└── Grafana      ← 可視化（今後活用）
```

## なぜクラスター外に置いたか
- Workerノードのメモリが少ない（available ~490MB）
- 監視インスタンスが落ちてもアプリに影響しない
- Grafanaなど後から追加しやすい

---

## kube-state-metrics（`manifests/kube-state-metrics.yaml`）

k8sクラスター内にDeploymentとして配置。PodやDeploymentの状態をメトリクスとして公開する。

```
k8sクラスター内
└── kube-state-metrics Pod（namespace: monitoring）
    └── NodePort 32300で外部公開
            ↑
    monitoringインスタンスのPrometheusが30秒ごとにスクレイプ
```

### デプロイ方法
```bash
# control-planeで
sudo kubectl create namespace monitoring
sudo kubectl apply -f manifests/kube-state-metrics.yaml
```

### RBACについて
kube-state-metricsはk8sのAPIからPodやDeploymentの情報を読み取る必要がある。
そのためServiceAccount・ClusterRole・ClusterRoleBindingを作成して権限を付与している。

---

## Prometheus（`prometheus/`）

### prometheus.yml
kube-state-metricsのエンドポイントをスクレイプ対象として設定。
2台のWorkerノードのIPとNodePortを指定。

### rules.yml
| アラート名 | 条件 | 待機時間 |
|-----------|------|---------|
| PodCrashLooping | 5分間でrestartが増加 | 1分 |
| PodNotReady | Readyでない状態が継続 | 5分 |

対象namespaceを `neko`, `sensor-api`, `sensor-view` に限定。

#### 重複排除のポイント
kube-state-metricsを2台のWorkerから収集しているため、同じPodのアラートが2つ発火する。
`sum without(instance, job)` でinstanceラベルを除去してから集計することで1つにまとめる。

```yaml
expr: sum without(instance, job) (increase(kube_pod_container_status_restarts_total{...}[5m])) > 0
```

---

## Alertmanager（`alertmanager/alertmanager.yml.example`）

### ルーティング
namespaceラベルを見て、対応するSlackチャンネルに振り分ける。

```
namespace: neko        → #alerts-neko
namespace: sensor-api  → #alerts-sensor-api
namespace: sensor-view → #alerts-sensor-view
それ以外               → blackhole（無視）
```

defaultをblackholeにすることで、対象外のnamespace（kube-systemなど）の通知を無視できる。

### group_byについて
`group_by: [namespace, alertname, pod]` を設定することで、同じPodの複数アラートを1通にまとめる。

---

## セキュリティグループの設定
| 送信元 | 送信先 | ポート | 用途 |
|--------|--------|--------|------|
| monitoringインスタンス | sg-master | 6443 | PrometheusがkubeAPIにアクセス |
| monitoringインスタンス | sg-worker | 32300 | kube-state-metricsのスクレイプ |

---

## 秘匿情報の扱い
- `alertmanager/alertmanager.yml`（Slack Webhook URL含む）は`.gitignore`で除外
- `alertmanager.yml.example`をテンプレートとしてGit管理
- monitoringインスタンス上でexampleをコピーしてURLを直接書き込む運用

---

## 起動方法
```bash
git clone https://github.com/<your-name>/monitoring.git
cd monitoring
cp alertmanager/alertmanager.yml.example alertmanager/alertmanager.yml
vi alertmanager/alertmanager.yml  # Webhook URLを設定
cp .env.example .env
vi .env                           # GRAFANA_PASSWORDを設定
docker compose up -d
```

---

## 今日のポイントまとめ

| テーマ | 学んだこと |
|--------|-----------|
| Prometheus | scrape_configs, alerting rules, PromQL, sum without |
| Alertmanager | routing, receivers, group_by, blackhole, resolve_timeout |
| kube-state-metrics | k8sのメトリクス公開、RBAC |
| セキュリティ | 秘匿情報をgitignore、SGで最小権限 |
| Docker Compose | 複数サービスをまとめて管理 |
